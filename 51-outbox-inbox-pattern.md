# 51. Outbox, Inbox и идемпотентность: как надёжно обмениваться событиями

## Зачем это знать

Ты пишешь микросервис, который принимает заказ. Логика простая: сохранить заказ в БД, отправить событие «новый заказ» в Kafka, чтобы другие сервисы (складской, платёжный, доставки) увидели и начали работать. Код на пять строк, работает на локальной машине, работает в тестах. Заливаешь в прод — иногда, раз в тысячу заказов, что-то ломается. Заказ создан в БД, но склад не забронировал товар. Или наоборот — склад забронировал, но заказа в БД нет. Клиент звонит и говорит «списали деньги, а заказа нет».

Причина — фундаментальная и не решается «правильным кодом» в лоб. Между двумя действиями (сохранить в БД + отправить в Kafka) есть **окно в несколько миллисекунд**, когда процесс может упасть, сервер может ребутнуться, сетевая карта может сойти с ума. Оба действия происходят в разных системах — в БД и в брокере — а они друг о друге ничего не знают. Гарантированно сделать оба или ни одного (**atomicity**) без специальных инструментов невозможно.

Разница между «пишу код который иногда падает» и «понимаю reliable messaging» — это знание почему нельзя просто «сначала БД, потом Kafka» или «использовать XA-транзакции». И знание конкретного решения — **Outbox pattern**. Он не убирает окно между двумя действиями, а **сдвигает** проблему: вместо «БД + брокер» становится «БД + БД» (событие сохраняется в БД в одной транзакции с бизнес-данными, отдельный процесс потом отправляет в брокер). Атомарность внутри одной БД — решённая проблема, транзакция либо коммитится вся, либо не коммитится совсем.

Второй слой проблемы — на стороне получателя. Даже когда мы гарантированно отправили сообщение, брокер может доставить его **дважды**. Consumer перезапустился между обработкой и подтверждением offset — сообщение приходит ещё раз, обработка повторяется. Тот же заказ создаётся два раза, тот же email отправляется дважды. Решение — **Inbox pattern** или другой механизм идемпотентности.

Разберём: почему нельзя «просто отправить в Kafka после commit». XA-транзакции и почему они не работают для Kafka. Outbox pattern — механика, структура таблицы, producer, publisher, гарантии. Timeline что происходит при разных crashes. Debezium/CDC как автоматизация publisher'а. Inbox pattern на стороне consumer'а. Комбинация Outbox + Inbox = effectively exactly-once. Другие подходы к идемпотентности (conditional UPDATE, UPSERT, Idempotency-Key). Полная реализация в Spring с ShedLock для multi-instance. Реальные проблемы: рост outbox, publisher lag, ordering, dead-letter, эволюция схемы. Диагностика в проде. Альтернативы и когда они работают.

Saga pattern — файл 50. Полный overview reliable messaging — там же. Здесь фокус конкретно на outbox/inbox механике.

## Проблема: почему нельзя «просто послать»

Разберём самый простой код обработки заказа:

```java
@Transactional
public void createOrder(OrderRequest req) {
    Order order = new Order(req);
    orderRepo.save(order);           // Шаг 1: сохранить в БД
    kafka.send("orders", order);     // Шаг 2: отправить событие
}
```

Кажется просто и правильно. Транзакция коммитится в конце метода. Что не так?

**Timeline «идеального» выполнения** (без ошибок):

```
T=0ms:  метод начался
T=1ms:  orderRepo.save() — Order в JPA persistence context (пока не в БД)
T=2ms:  kafka.send() — событие в Kafka producer buffer (не отправлено в брокер)
T=3ms:  метод вернул управление
T=4ms:  @Transactional interceptor коммитит транзакцию → Order в БД
T=5ms:  Kafka producer flush → событие уходит в брокер
```

Всё хорошо. Заказ в БД, событие в Kafka. Downstream-сервисы получат событие и начнут работать.

**Timeline с крашем после commit**:

```
T=0ms:  метод начался
T=1ms:  orderRepo.save() — Order в persistence context
T=2ms:  kafka.send() — событие в Kafka producer buffer
T=3ms:  метод вернул управление
T=4ms:  commit транзакции → Order в БД
T=5ms:  💥 CRASH процесса (kill -9, OOM, node failed)
        Kafka producer buffer уничтожен вместе с процессом.
        Событие никогда не отправится.
```

Заказ в БД есть. Событие потеряно. Downstream не знает про заказ. Клиент видит «заказ создан», но склад не бронирует товар. Тихая рассинхронизация.

Может «переставить местами»? Сначала Kafka, потом БД:

```java
public void createOrder(OrderRequest req) {
    Order order = new Order(req);
    kafka.send("orders", order);     // сначала событие
    orderRepo.save(order);            // потом БД
}
```

**Timeline с крашем**:

```
T=0ms:  метод начался
T=1ms:  kafka.send() → событие отправлено в брокер (успех)
T=2ms:  💥 CRASH процесса
        orderRepo.save() не выполнилось.
        В БД заказа нет.
```

Событие ушло, downstream думает что заказ есть. Склад бронирует товар для несуществующего заказа. Хуже чем первый вариант — теперь у нас **фантомные бизнес-операции**.

**Почему нельзя `@TransactionalEventListener` с `AFTER_COMMIT`?**

Spring предлагает такой паттерн — публиковать событие только после commit:

```java
@Transactional
public void createOrder(OrderRequest req) {
    Order order = new Order(req);
    orderRepo.save(order);
    eventPublisher.publishEvent(new OrderCreatedEvent(order));   // Spring event
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleOrderCreated(OrderCreatedEvent event) {
    kafka.send("orders", event);   // публикуется после commit
}
```

Выглядит красиво. Но проблема осталась — только сдвинулась во времени:

```
T=4ms:  commit → Order в БД
T=5ms:  @TransactionalEventListener triggered
T=6ms:  kafka.send() начал отправку
T=7ms:  💥 CRASH до того как send() завершился
        Событие потеряно.
```

Между commit и успешным send есть окно. Даже маленькое (микросекунды) — на масштабе тысяч запросов в секунду это будут регулярные потери. Раз в 10 000 запросов, раз в час, что-то теряется. В prod-среде это неприемлемо.

## XA / 2PC: почему это не решение

Формально atomic write across two systems решается через **XA (eXtended Architecture)** — стандарт распределённых транзакций. Работает через **Two-Phase Commit (2PC)**:

**Phase 1: prepare**. Coordinator спрашивает участников (БД и Kafka): «готовы закоммитить?». Каждый отвечает yes/no. Если хоть один сказал no — abort всем.

**Phase 2: commit**. Если все сказали yes — coordinator говорит всем commit. Если no — rollback.

Гарантия: либо все закоммитились, либо никто.

**Проблемы XA в реальности**:

**Kafka не поддерживает XA вовсе**. У Kafka есть свои транзакции (introduced в 0.11), но они координируют только внутри Kafka (несколько topics, exactly-once producer semantic). Между Kafka и внешней системой — не работает. Уже одна эта причина исключает XA-подход для Kafka.

**Coordinator = single point of failure**. Если координатор упал между prepare и commit — участники висят в «prepared» state, не зная commit or rollback. Ресурсы (locks, буферы) заблокированы. Требуется ручное вмешательство администратора для расчистки.

**Performance overhead огромный**. Каждая операция — минимум 4 сетевых roundtrip (prepare-request, prepare-response, commit-request, commit-response). При тысяче TPS — dead slow.

**Модерн БД (Postgres 15+) не активно развивают XA support**. Он есть, но считается legacy. Никто в новых системах не использует.

**Enterprise message brokers (Tibco, MQSeries) исторически поддерживали XA** — там 2PC работал. Kafka, RabbitMQ, современные брокеры — нет.

Итог: XA — теоретически правильное решение проблемы atomicity, практически не применимо с современным стеком. Нужен другой подход.

## Outbox pattern: сдвиг проблемы

Идея outbox — гениально простая. Вместо «БД + Kafka» делаем «БД + БД»:

1. В той же транзакции что и бизнес-данные сохраняем **описание события** в специальную таблицу `outbox`.
2. Отдельный процесс (publisher) читает outbox → отправляет в Kafka → отмечает как processed.

Транзакция БД теперь атомарна: либо Order и OutboxEntry сохранены вместе, либо ни один. Никаких Kafka в транзакции. Никакого XA.

**Гарантия**: если БД коммит прошёл — outbox-запись существует — событие рано или поздно опубликуется. Publisher retry обеспечивает eventual delivery.

**Компромисс**: событие публикуется не мгновенно, а с задержкой (обычно секунды — время между polling'ами publisher'а). Для большинства бизнес-задач это приемлемо. Если нужна миллисекундная latency — Debezium/CDC (ниже).

**Timeline с outbox и крашем**:

```
T=0ms:  метод начался
T=1ms:  orderRepo.save(order) — Order в persistence context
T=2ms:  outboxRepo.save(outboxEntry) — OutboxEntry в persistence context
T=3ms:  commit транзакции → И Order И OutboxEntry в БД атомарно
T=4ms:  💥 CRASH процесса
        Publisher ещё не отправил — но OutboxEntry в БД!
        
T=Nms:  Процесс перезапустился (или другая реплика продолжает работать).
        Publisher polling видит неотправленный OutboxEntry.
        Отправляет в Kafka. Отмечает processed.
```

Даже если процесс крашнулся до отправки — событие не потеряется. При следующем запуске publisher его найдёт.

## Механика outbox: структура таблицы

Классическая структура:

```sql
CREATE TABLE outbox (
    id BIGSERIAL PRIMARY KEY,
    aggregate_type TEXT NOT NULL,          -- 'Order', 'Payment', 'User'
    aggregate_id TEXT NOT NULL,             -- '42', 'user-abc-123'
    event_type TEXT NOT NULL,               -- 'OrderCreated', 'PaymentReceived'
    payload JSONB NOT NULL,                 -- сериализованное событие как JSON
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ                -- NULL если ещё не отправлено
);

CREATE INDEX idx_outbox_unprocessed 
    ON outbox(created_at) 
    WHERE processed_at IS NULL;
```

Пройдём по каждому полю:

**`id`** — просто primary key. BIGSERIAL, автоинкремент.

**`aggregate_type`** — что за сущность. `Order`, `Payment`, `User`. Часто используется как имя топика в Kafka (`orders`, `payments`, `users`). Отделяет разные потоки событий.

**`aggregate_id`** — конкретная сущность. `42` для `Order` id=42. Используется как **partition key** в Kafka — гарантирует что все события для одного Order попадают в одну partition, следовательно обрабатываются в порядке их создания. Без этого события `OrderCreated` и `OrderCancelled` для одного заказа могли бы обработаться не в том порядке.

**`event_type`** — тип события. `OrderCreated`, `OrderCancelled`, `OrderShipped`. Consumer решает как обрабатывать на основе типа.

**`payload`** — само событие в JSON. Все данные которые нужны consumer'у (id, статус, суммы, timestamps, метаданные).

**`created_at`** — когда создано. Используется для сортировки при отправке (FIFO), для retention (удаление старых).

**`processed_at`** — когда отправлено в Kafka. `NULL` = ещё не отправлено. Publisher ищет по `WHERE processed_at IS NULL`.

**Partial index** — критически важен для производительности. Обычный индекс на `created_at` покрыл бы всю таблицу (миллионы старых обработанных записей). Partial индекс `WHERE processed_at IS NULL` — только неотправленные, обычно тысячи в моменте. Разница в производительности publisher'а на большой outbox — десятки раз.

## Producer: атомарная запись Order + OutboxEntry

Producer — сервис где происходит бизнес-операция. Внутри одной транзакции сохраняет и бизнес-данные, и outbox-запись:

```java
@Service
class OrderService {

    @Autowired OrderRepository orderRepo;
    @Autowired OutboxRepository outboxRepo;
    @Autowired ObjectMapper mapper;

    @Transactional
    public Order createOrder(OrderRequest req) {
        // Шаг 1: создать и сохранить бизнес-сущность
        Order order = new Order(req);
        orderRepo.save(order);

        // Шаг 2: создать outbox-запись в той же транзакции
        OutboxEntry entry = new OutboxEntry();
        entry.setAggregateType("Order");
        entry.setAggregateId(order.getId().toString());
        entry.setEventType("OrderCreated");
        entry.setPayload(mapper.writeValueAsString(
            new OrderCreatedEvent(order.getId(), order.getCustomerId(), order.getTotal())
        ));
        outboxRepo.save(entry);

        return order;
        // @Transactional commit в конце метода → 
        // и Order, и OutboxEntry записываются атомарно
    }
}
```

Ключевой момент — `@Transactional` вокруг всего метода. И `orderRepo.save`, и `outboxRepo.save` — в одной транзакции. Либо commit коммитит оба, либо rollback отменяет оба. Никаких промежуточных состояний где Order есть, а OutboxEntry нет (или наоборот).

**Payload** — обычно сериализуем как JSON через Jackson. Может быть protobuf, Avro — зависит от контракта с consumer'ами. Важно: содержать всё что нужно consumer'у **сейчас**. Не полагаться на «consumer сходит в БД и достанет» — БД в момент обработки может уже быть в другом состоянии (следующие операции), или сам Order уже cancelled.

**OrderCreatedEvent** — отдельный класс DTO для события. Не сериализуем сам Order (изменения его полей во времени сломают старые consumers). Event — immutable snapshot данных на момент создания заказа.

## Publisher: чтение outbox и отправка в Kafka

Publisher работает независимо от producer. Может быть в том же процессе (scheduled job) или в отдельном сервисе. Логика:

```java
@Component
class OutboxPublisher {

    @Autowired OutboxRepository outboxRepo;
    @Autowired KafkaTemplate<String, String> kafka;

    @Scheduled(fixedDelay = 500)   // каждые 500 мс
    @Transactional
    public void publish() {
        // 1. Читаем пачку неотправленных
        List<OutboxEntry> pending = outboxRepo
            .findTop100ByProcessedAtIsNullOrderByCreatedAt();
        
        // 2. Для каждой пытаемся отправить
        for (OutboxEntry entry : pending) {
            try {
                kafka.send(
                    entry.getAggregateType().toLowerCase() + "s",   // topic: "orders"
                    entry.getAggregateId(),                          // key: partition key
                    entry.getPayload()                                // value: JSON
                ).get(5, TimeUnit.SECONDS);   // ждём подтверждения от брокера
                
                // 3. Успех — отмечаем как processed
                entry.setProcessedAt(Instant.now());
                outboxRepo.save(entry);
                
            } catch (Exception e) {
                log.error("Не удалось опубликовать outbox id={}", entry.getId(), e);
                // Не отмечаем — следующая итерация повторит
            }
        }
    }
}
```

Пошаговый разбор:

**`@Scheduled(fixedDelay = 500)`** — метод вызывается каждые 500 мс после завершения предыдущего. Не `fixedRate` — если предыдущий вызов затянулся (много данных), не будем запускать параллельно.

**`findTop100By...`** — берём пачку по 100 штук. Не всю таблицу — иначе при большом бэклоге забьём память и Kafka не успеет обработать всё сразу. Batch по 100 — балансирует throughput и memory.

**Сортировка по `created_at`** — FIFO, старые события обрабатываются первыми. В сочетании с `aggregate_id` как partition key даёт правильный порядок событий для одной сущности.

**`kafka.send(...).get(5, TimeUnit.SECONDS)`** — отправляем и **ждём подтверждения** от брокера (максимум 5 секунд). Без `.get()` send асинхронный — вернётся сразу, ошибку узнаем позже (или никогда). Нам нужна синхронная гарантия что брокер принял сообщение — только тогда отмечаем как processed.

**Try/catch вокруг send + update**. При ошибке (Kafka недоступна, timeout, network issue) — логируем и **не отмечаем**. Следующая итерация через 500 мс попытается снова. При постоянных ошибках сообщения накапливаются в очереди — мониторить `count WHERE processed_at IS NULL`, если растёт — что-то не так с брокером.

**Важный caveat: возможны дубли**. Что если Kafka приняла сообщение, но перед `entry.setProcessedAt()` процесс упал? Через 500 мс следующая итерация увидит запись всё ещё как неотправленную → отправит **ещё раз**. Kafka получит два одинаковых сообщения. Consumer должен уметь дедуплицировать (Inbox pattern, ниже) — это цена за at-least-once delivery.

**Timeline с крашем publisher'а**:

```
T=0:    publisher: SELECT unprocessed → получил entry id=42
T=1:    publisher: kafka.send(...) → Kafka сохранила сообщение
T=2:    💥 CRASH publisher'а до setProcessedAt
        В БД entry.processed_at = NULL (не обновилось).
        В Kafka сообщение уже есть.
        
T=N:    publisher перезапустился
T=N+1:  SELECT unprocessed → снова entry id=42 (processed_at всё ещё NULL)
T=N+2:  kafka.send(...) → в Kafka теперь ДВА сообщения с тем же содержанием
T=N+3:  setProcessedAt = now → в БД помечено
```

Дубликат в Kafka. Consumer обязан обработать это gracefully через дедупликацию.

## Что происходит на consumer стороне

Другой сервис (например `WarehouseService`) подписан на топик `orders`:

```java
@KafkaListener(topics = "orders")
public void handle(String payload) throws Exception {
    OrderCreatedEvent event = mapper.readValue(payload, OrderCreatedEvent.class);
    warehouseService.reserveItems(event);
}
```

**Проблема**: сообщение может прийти дважды. Причины:

1. **Publisher отправил дважды** (см. выше — краш между send и update).
2. **Consumer обработал, но упал до commit offset** — Kafka не знает что обработано, при перезапуске переотдаёт сообщение.
3. **Rebalance в consumer group** — partition переехала на другого consumer'а, он начинает читать с последнего committed offset.

Второй раз обрабатывать — плохо: два товара забронированы вместо одного, два email отправлены, два перевода денег списаны. Нужна **идемпотентность**.

## Inbox pattern: дедупликация на consumer стороне

Inbox — таблица с ID уже обработанных сообщений. При получении события проверяем: обрабатывали ли уже? Если да — пропускаем.

**Структура таблицы**:

```sql
CREATE TABLE inbox (
    message_id TEXT PRIMARY KEY,           -- уникальный ID сообщения
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Consumer с дедупликацией**:

```java
@Service
class WarehouseService {

    @Autowired InboxRepository inboxRepo;
    @Autowired ItemReservationRepository reservationRepo;

    @Transactional
    public void handle(OrderCreatedEvent event) {
        // 1. Проверяем: не обработано ли уже?
        if (inboxRepo.existsById(event.getMessageId())) {
            log.debug("Дубликат сообщения {}, пропускаем", event.getMessageId());
            return;
        }
        
        // 2. Выполняем бизнес-логику
        for (OrderLine line : event.getLines()) {
            reservationRepo.save(new Reservation(event.getOrderId(), line));
        }
        
        // 3. Записываем в inbox — этот messageId обработан
        inboxRepo.save(new InboxEntry(event.getMessageId()));
        
        // @Transactional commit → 
        // и бизнес-действия, и inbox запись атомарно
    }
}
```

Ключевой момент — **всё в одной транзакции**. Проверка → обработка → запись в inbox. Атомарность гарантирует что либо все три шага сделаны, либо ни один.

**Timeline с крашем consumer'а**:

```
Сценарий 1: Первое получение сообщения
T=0:  сообщение id=xyz-123 пришло из Kafka
T=1:  БД: inbox.existsById('xyz-123') → false
T=2:  бизнес-логика: reservationRepo.save(...)
T=3:  БД: inboxRepo.save(InboxEntry('xyz-123'))
T=4:  transaction commit → и reservation, и inbox записаны
T=5:  Kafka commit offset

Сценарий 2: Kafka отдала сообщение повторно (краш между обработкой и Kafka commit)
T=0:  сообщение id=xyz-123 пришло ЕЩЁ РАЗ
T=1:  БД: inbox.existsById('xyz-123') → TRUE (записано на прошлой попытке)
T=2:  return, никакой обработки
T=3:  Kafka commit offset (не пытаемся снова)

Сценарий 3: Краш во время первой обработки
T=0:  сообщение id=xyz-123 пришло
T=1:  БД: inbox.existsById('xyz-123') → false
T=2:  reservationRepo.save(...) — insert выполнен, но НЕ commit
T=3:  💥 CRASH процесса
      Транзакция rollback → reservation НЕ сохранен, inbox тоже НЕ записан.
      
T=N:  сообщение id=xyz-123 приходит снова (Kafka не увидела commit)
T=N+1: inbox.existsById('xyz-123') → false (ничего не записано)
T=N+2: обработка успешно, commit проходит.
```

Всё работает правильно. Атомарность в одной БД делает поведение предсказуемым.

## Как правильно генерировать messageId

Дедупликация работает только если каждое **сообщение** имеет **стабильный уникальный идентификатор**.

**Правильно**: UUID сгенерированный producer'ом при создании события:

```java
class OrderCreatedEvent {
    private final String messageId;   // UUID.randomUUID().toString()
    private final Long orderId;
    private final Long customerId;
    // ...
    
    public OrderCreatedEvent(...) {
        this.messageId = UUID.randomUUID().toString();
        // ...
    }
}
```

При отправке в Kafka messageId передаётся либо в headers, либо в самом payload. При повторной отправке (retry) — тот же messageId. Consumer видит одинаковый ID → пропускает дубликат.

**Неправильно**: использовать `orderId` как messageId. Один Order может иметь несколько событий (`OrderCreated`, `OrderShipped`, `OrderCancelled`) — все с одним orderId. Consumer подумает что второе событие уже обработано (потому что orderId уже в inbox) — пропустит `OrderShipped` считая его дубликатом `OrderCreated`.

**Неправильно**: генерировать messageId в consumer'е. Каждое получение — новый UUID → дедупликация не работает.

**В Kafka**: messageId можно взять из связки `topic-partition-offset` — она уникальна для каждого сообщения. Или из headers (producer явно устанавливает). Или из payload (event внутри имеет messageId).

**В RabbitMQ**: messageId устанавливается через `MessageProperties.setMessageId(...)`. Явный контроль producer'а.

## Retention: чтобы таблицы не росли до бесконечности

Обе таблицы (outbox, inbox) должны периодически чиститься. Без cleanup — миллионы записей, индексы деградируют, VACUUM не успевает.

**Outbox retention**:

```sql
DELETE FROM outbox WHERE processed_at < now() - INTERVAL '7 days';
```

Хранить неделю — обычно достаточно для disaster recovery (если downstream пропустил сообщения, можно повторить). Может быть меньше (3 дня) или больше (30 дней) в зависимости от требований.

Ещё эффективнее — **партиционирование** outbox по `created_at` (месячные партиции). Удаление старых — `DROP PARTITION`, мгновенно, без bloat (см. файлы 91, 108).

**Inbox retention**:

```sql
DELETE FROM inbox WHERE processed_at < now() - INTERVAL '30 days';
```

Retention должен быть **больше** чем максимальная задержка повторной доставки. Если Kafka retention 7 дней, inbox надо хранить как минимум 7+ дней — иначе дубликат пришедший через 8 дней после первой обработки не найдёт свою запись в inbox и будет обработан как новый.

Стандартный запуск через `@Scheduled`:

```java
@Scheduled(cron = "0 0 3 * * *")   // каждую ночь в 3:00
public void cleanupOutbox() {
    int deleted = outboxRepo.deleteOlderThan(Instant.now().minus(7, ChronoUnit.DAYS));
    log.info("Deleted {} old outbox entries", deleted);
}
```

## Outbox + Inbox = «effectively exactly-once»

Комбинация даёт end-to-end гарантию:

```
[Service A: Producer]                    [Service B: Consumer]
        │                                          │
        │  @Transactional {                        │
        │    save(Order)                           │
        │    save(OutboxEntry)                     │
        │  } → атомарный commit                    │
        │                                          │
        │  Publisher (scheduled):                  │
        │  reads unprocessed outbox                │
        │  → kafka.send()                          │
        │  → marks processed                       │
        │                                          │
        │              Kafka                       │
        │              │                            │
        │              ▼                            │
        │        (может дублировать)                │
        │                                          ▼
        │                                    @KafkaListener
        │                                          │
        │                                          │  @Transactional {
        │                                          │    if in inbox → skip
        │                                          │    process business
        │                                          │    save(InboxEntry)
        │                                          │  } → атомарный commit
```

Итог:
- **Producer side**: атомарно БД + outbox. Публикация гарантированно случится (Publisher retry).
- **Broker**: at-least-once — может доставить N раз.
- **Consumer side**: inbox фильтрует дубликаты. Каждое сообщение обрабатывается ровно один раз.
- **End-to-end**: **effectively exactly-once**.

«Effectively» потому что в строгом теоретическом смысле exactly-once в распределённых системах невозможно (доказано FLP-теоремой). Но с точки зрения бизнес-наблюдаемой семантики — эквивалентно: заказ создан один раз, товар забронирован один раз, email отправлен один раз. Всё что важно бизнесу.

## Debezium/CDC: альтернатива polling'у

Publisher через `@Scheduled` — простой подход, но с задержкой (500 мс между polling'ами). Для real-time систем есть альтернатива — **CDC (Change Data Capture) через Debezium**.

**Как работает Debezium**:

1. Читает **WAL (Write-Ahead Log)** PostgreSQL напрямую через logical replication.
2. Каждый INSERT/UPDATE/DELETE становится событием.
3. Стриминг в Kafka topics в реальном времени.

Для outbox — специальный **SMT (Single Message Transform) `Outbox Event Router`**. Настраиваешь: «читай таблицу outbox, публикуй в топики по `aggregate_type`, key = `aggregate_id`, value = `payload`». Debezium делает всё автоматически.

**Setup требует**:

- В PostgreSQL: `wal_level=logical` в конфигурации.
- Создать replication slot для Debezium.
- Развернуть Kafka Connect с Debezium connector.
- Настроить connector: какая таблица, какие topics, transformation rules.

**Плюсы**:

- **Real-time propagation** — миллисекундная задержка вместо секундной.
- **Никакого custom publisher** — Debezium сам всё делает.
- **Никакой polling load** на БД — CDC читает WAL, не делает SELECT'ов.
- **Ordering** гарантирован через WAL порядок.

**Минусы**:

- **Требует Kafka Connect** — дополнительная инфраструктура для поддержки.
- **Настройка logical replication** — влияет на WAL, memory, disk usage.
- **Debezium — Java-приложение** — нужно мониторить, обновлять, поддерживать.

Прямой CDC на бизнес-таблицы (без outbox) — тоже возможен, но плохо: broker получает schema-level events («row inserted», «row updated»), не business events («OrderCreated», «OrderCancelled»). Coupling с internal schema, изменение схемы ломает consumers. Outbox pattern даёт clean business events.

**Когда Debezium имеет смысл**: high-volume системы (thousands events per second) где polling latency неприемлема, где инфраструктура Kafka Connect уже развёрнута. Для типичного enterprise с сотнями events per second — polling через `@Scheduled` проще и достаточно.

## Другие подходы к идемпотентности

Inbox — один из способов. Есть другие, каждый со своими use cases.

**Idempotent UPDATE через conditional clause**. Работает когда операция — state machine transition:

```sql
UPDATE orders SET status='PROCESSED' 
WHERE id=? AND status='NEW';
-- Если rows affected = 0 → уже обработано, ничего не сделали
-- Если rows affected = 1 → успешно перевели в PROCESSED
```

Проверка current status **до** update. Затрагивает только если состояние соответствует ожидаемому. Повторные вызовы после успеха — no-op (rows affected = 0). Нет необходимости в inbox, идемпотентность встроена в саму операцию.

**UPSERT для insert-операций**:

```sql
INSERT INTO users (id, email, name) VALUES (?, ?, ?)
ON CONFLICT (id) DO NOTHING;
```

Второй вызов с тем же ID — `DO NOTHING`, никакого дубликата. Или `DO UPDATE SET ...` для upsert-семантики (обновить если существует).

**Optimistic version с `@Version`** в JPA:

```java
@Entity
class Order {
    @Id Long id;
    @Version int version;
    // ...
}

@Transactional
public void update(Long id, Consumer<Order> update) {
    Order o = orderRepo.findById(id).orElseThrow();
    update.accept(o);
    orderRepo.save(o);   // UPDATE ... WHERE id=? AND version=?
    // Если version изменилась (кто-то опередил) → OptimisticLockException
}
```

Не совсем идемпотентность, но защита от concurrent modification. Полезно в сочетании с retry на этой ошибке.

**Idempotency-Key header для HTTP API** — используется в Stripe API. Клиент передаёт `Idempotency-Key: <unique-uuid>` в заголовке:

```
POST /payments
Idempotency-Key: abc123-def456
Content-Type: application/json
{
  "amount": 100.00,
  "currency": "USD"
}
```

Server:
- Первый вызов с ключом → обрабатывает, **сохраняет response** в БД с этим ключом.
- Повторный вызов с тем же ключом → возвращает **сохранённый response** без повторного выполнения операции.

Клиент отвечает за генерацию ключей (обычно UUID при первой попытке, тот же при retry). Server поддерживает key → response mapping с TTL (обычно 24 часа).

Практически: единственный правильный способ safe retry для не-idempotent HTTP-операций (POST /payments, POST /transfers). Без Idempotency-Key каждый retry рискует двойной операцией.

## Реализация в Spring — полный пример

Всё вместе — минимальный работающий Spring Boot setup.

**Entity + Repository**:

```java
@Entity
@Table(name = "outbox")
public class OutboxEntry {
    @Id @GeneratedValue Long id;
    String aggregateType;
    String aggregateId;
    String eventType;
    
    @Column(columnDefinition = "jsonb")
    String payload;
    
    Instant createdAt = Instant.now();
    Instant processedAt;
    // getters/setters
}

interface OutboxRepository extends JpaRepository<OutboxEntry, Long> {
    List<OutboxEntry> findTop100ByProcessedAtIsNullOrderByCreatedAt();
}
```

**Producer в бизнес-сервисе**:

```java
@Service
class OrderService {

    @Autowired OrderRepository orderRepo;
    @Autowired OutboxRepository outboxRepo;
    @Autowired ObjectMapper mapper;

    @Transactional
    public Order createOrder(OrderRequest req) {
        Order order = new Order(req);
        orderRepo.save(order);

        OutboxEntry entry = new OutboxEntry();
        entry.setAggregateType("Order");
        entry.setAggregateId(order.getId().toString());
        entry.setEventType("OrderCreated");
        try {
            entry.setPayload(mapper.writeValueAsString(
                new OrderCreatedEvent(
                    UUID.randomUUID().toString(),   // messageId!
                    order.getId(),
                    order.getCustomerId(),
                    order.getTotal(),
                    Instant.now()
                )
            ));
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
        outboxRepo.save(entry);

        return order;
    }
}
```

**Publisher — scheduled job**:

```java
@Component
@Slf4j
class OutboxPublisher {

    @Autowired OutboxRepository outboxRepo;
    @Autowired KafkaTemplate<String, String> kafka;

    @Scheduled(fixedDelay = 500)
    @Transactional
    public void publish() {
        List<OutboxEntry> pending = 
            outboxRepo.findTop100ByProcessedAtIsNullOrderByCreatedAt();
        
        if (pending.isEmpty()) return;

        for (OutboxEntry entry : pending) {
            try {
                kafka.send(
                    entry.getAggregateType().toLowerCase() + "s",
                    entry.getAggregateId(),
                    entry.getPayload()
                ).get(5, TimeUnit.SECONDS);
                
                entry.setProcessedAt(Instant.now());
                outboxRepo.save(entry);
                
            } catch (Exception e) {
                log.error("Failed to publish outbox id={}", entry.getId(), e);
                // не отмечаем — следующий цикл повторит
            }
        }
    }
}
```

## ShedLock для multi-instance publisher

Проблема: приложение развёрнуто в K8s с 3 репликами. Все 3 запускают `@Scheduled publish()` параллельно каждые 500 мс. Все 3 читают одни и те же outbox-записи → отправляют в Kafka по 3 раза каждое сообщение. Даже с inbox на consumer стороне — лишняя нагрузка на Kafka и БД publisher'а.

**ShedLock** — библиотека для distributed lock через shared БД. Только один инстанс выполняет job в момент времени.

```gradle
implementation 'net.javacrumbs.shedlock:shedlock-spring:5.10.0'
implementation 'net.javacrumbs.shedlock:shedlock-provider-jdbc-template:5.10.0'
```

Настройка:

```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "PT30S")
public class ShedLockConfig {
    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(dataSource);
    }
}
```

Аннотация на scheduled методе:

```java
@Scheduled(fixedDelay = 500)
@SchedulerLock(
    name = "OutboxPublisher_publish", 
    lockAtLeastFor = "PT100MS", 
    lockAtMostFor = "PT30S"
)
@Transactional
public void publish() {
    // теперь только один инстанс выполняет это в момент времени
}
```

**Как работает**. При запуске метода ShedLock пытается атомарно INSERT/UPDATE запись в таблицу `shedlock` с именем `OutboxPublisher_publish` и `locked_until = now + lockAtMostFor`. Если удалось — этот инстанс получил lock, выполняет. Если запись уже существует и `locked_until > now` — другой инстанс уже держит, метод пропускается. После завершения — `locked_until = now + lockAtLeastFor` (гарантирует минимальное время удержания, защита от clock skew).

`lockAtMostFor` — страховка если инстанс упал не сняв lock. Через это время lock освободится автоматически.

**Реальный кейс из КНП** (память `knp-fno21-shedlock-stale-image-dup-regnum`). Scheduled job без ShedLock работал параллельно на 3 репликах → duplicate INSERT'ы с одинаковыми регистрационными номерами. Добавили ShedLock — проблема исчезла в течение часа после deploy.

## Реальные проблемы в проде

**Рост outbox таблицы**. Активная система пишет тысячи events в час. Без cleanup — миллионы строк, индексы деградируют. Мониторить `SELECT count(*) FROM outbox` — если растёт даже после cleanup, значит cleanup не успевает или publisher тормозит.

Решение: retention job (см. выше). Партиционирование по `created_at` для больших объёмов.

**Publisher lag**. Publisher не успевает отправлять — `count WHERE processed_at IS NULL` растёт со временем. Симптом: события доходят до consumer'ов с большой задержкой, downstream реагирует медленно.

Причины: batch size мал, `fixedDelay` слишком большой, Kafka тормозит, один publisher на всё. Решения: увеличить batch size (100 → 500); уменьшить fixedDelay (500 мс → 100 мс); распараллелить (несколько publisher'ов на разные aggregate_type); перейти на Debezium.

**Ordering между aggregate'ами**. `aggregate_id` как partition key даёт порядок **внутри** одного aggregate. Порядок между разными aggregate'ами не гарантирован — они в разных partition. Обычно это OK (разные Orders независимы), но если бизнес-логика требует global ordering — единственный способ через single partition (bottleneck).

**Dead letter handling**. Событие не может быть обработано consumer'ом даже после retries (плохой формат, business validation fail). Обычно кладут в **Dead Letter Topic** для manual разбора. На стороне outbox — добавить колонку `error_count` и `last_error`, после N попыток перекладывать в отдельную error-таблицу.

**Schema evolution**. Payload в outbox — снимок схемы event на момент создания. Через месяц consumers обновились до v2 схемы — старые события в outbox всё ещё v1. Правила:

- Только backward compatible изменения (добавление nullable полей — ок; удаление или переименование — нет).
- Versioning в payload: `{"schemaVersion": 2, "data": {...}}`.
- Consumer поддерживает несколько версий одновременно в течение переходного периода.

## Диагностика в проде

**Симптом «события не приходят до consumer'а»**. Пошаговая диагностика:

**Шаг 1 — проверить outbox**:

```sql
SELECT count(*) FROM outbox WHERE processed_at IS NULL;
```

Растёт — publisher не работает или отстаёт. Стабильно 0 — publisher работает.

Если растёт, посмотреть самые старые:

```sql
SELECT id, created_at, aggregate_type, event_type 
FROM outbox 
WHERE processed_at IS NULL 
ORDER BY created_at 
LIMIT 20;
```

Возраст самых старых — насколько publisher отстаёт. Часы — плохо.

**Шаг 2 — проверить логи publisher'а**. Есть ли ошибки Kafka? `Timeout`, `broker not available`, `authorization failed`? Проверить что Kafka доступна из pod'а publisher'а.

**Шаг 3 — проверить ShedLock** (если используется):

```sql
SELECT * FROM shedlock WHERE name = 'OutboxPublisher_publish';
```

Если `locked_by` — умерший инстанс, `locked_until` в будущем далеко — lock завис. Ждать до `locked_until` или руками UPDATE.

**Шаг 4 — если publisher работает, проверить Kafka**:

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group warehouse-service-group
```

Растущий `LAG` — consumer не успевает или упал. Проверить логи consumer'а.

**Симптом «события дублируются на consumer стороне»**. Проверить:

**1. Есть ли inbox check в consumer коде**?

```java
if (inboxRepo.existsById(event.getMessageId())) { return; }
```

Если нет — добавить.

**2. Правильный ли messageId используется**? Не берётся ли `event.getOrderId()` вместо специального `messageId`? Каждое **сообщение** должно иметь свой уникальный ID, не бизнес-сущность.

**3. Правильный ли Retention inbox**? Если Kafka retention 7 дней, а inbox 1 день — дубликат пришедший через 2 дня будет обработан как новый.

**4. Атомарность транзакции**? Проверка + бизнес-логика + inbox save должны быть в одной `@Transactional`. Если inbox save в отдельной транзакции после бизнес-логики — краш между ними даст дубликат.

## Альтернативы outbox и их ограничения

**Простая цепочка без outbox**:

```java
@Transactional
public void createOrder(...) {
    orderRepo.save(o);
    kafka.send(...);
}
```

Проблема разобрана в начале. Не работает reliably. Не использовать в production.

**@TransactionalEventListener AFTER_COMMIT**. Тоже разобрано. Не помогает — окно между commit и send остаётся.

**Kafka transactions с JPA**:

```java
@Transactional("kafkaTransactionManager")
public void createOrder(...) {
    orderRepo.save(o);
    kafka.send(...);
}
```

`kafkaTransactionManager` управляет только транзакцией Kafka. JPA — отдельно. Нельзя атомарно закоммитить обе (без XA, которого нет в Kafka). `ChainedTransactionManager` (deprecated) — best-effort, не гарантия.

**Event Sourcing**. Совсем другая архитектура: не храним current state, только события. Текущее состояние вычисляется replay'ом. Публикация автоматически при append новых событий. Сложность значительно выше, оправдан только в specific сценариях (audit-heavy домены, complex temporal queries).

**Change Streams в MongoDB / DocumentDB**. Аналог CDC для MongoDB. Работает как Debezium для Postgres, но встроен в MongoDB.

## Заключение

Проблема: атомарная запись в БД + брокер невозможна без XA, а XA не работает с Kafka (не поддерживается + performance overhead + coordinator SPOF).

**Outbox pattern** решает через сдвиг проблемы. Событие сохраняется в таблицу `outbox` в той же транзакции что и бизнес-данные — атомарность внутри одной БД, решённая проблема. Отдельный publisher читает outbox → отправляет в брокер → отмечает как processed. Гарантия: если БД коммит прошёл, событие рано или поздно опубликуется.

**At-least-once delivery** — цена простоты. Publisher может отправить дубли (краш между send и update processed_at). Consumer обязан быть идемпотентным.

**Inbox pattern** — дедупликация на consumer стороне. Таблица processed messageId. Проверка + обработка + запись в inbox в одной транзакции. Атомарность гарантирует что каждое сообщение обрабатывается ровно один раз.

**Outbox + Inbox = effectively exactly-once**. Не теоретически pure exactly-once (невозможно), но бизнес-наблюдаемо эквивалентно: каждая операция выполняется один раз.

**MessageId** — обязательно уникальный **на каждое сообщение** (UUID), не на бизнес-сущность. Producer генерирует, включает в event, consumer использует для dedup.

**Retention** — обе таблицы должны периодически чиститься. Outbox: 7 дней достаточно обычно. Inbox: минимум как Kafka retention (7+ дней). Партиционирование для больших объёмов.

**ShedLock** для multi-instance publisher. Distributed lock через shared БД. Один инстанс за раз выполняет job. Реальный prod-кейс — без ShedLock три реплики создавали дубли registration numbers.

**Debezium/CDC** — real-time альтернатива polling. Читает WAL Postgres. Требует Kafka Connect infrastructure. Для high-volume систем оправдан.

**Другие подходы к идемпотентности**:
- **Conditional UPDATE** (state machine transitions).
- **UPSERT** для inserts (`ON CONFLICT DO NOTHING/UPDATE`).
- **Optimistic version** (`@Version`) — защита от concurrent modification.
- **Idempotency-Key HTTP header** — правильный способ safe retry для POST-операций.

**Реальные проблемы**: рост таблиц (retention job); publisher lag (увеличить batch/уменьшить delay/распараллелить/Debezium); ordering только внутри aggregate; dead letter для unprocessable сообщений; schema evolution через versioning.

**Диагностика в проде**: SQL по outbox (растущий count unprocessed = publisher issue); ShedLock таблица (застрявший lock); Kafka consumer lag (consumer не успевает); проверка messageId уникальности и inbox retention.

**Альтернативы (не работают reliably)**: простая цепочка, `@TransactionalEventListener`, Kafka transactions с JPA без XA.

Saga pattern (для оркестрации нескольких сервисов) — файл 50. Distributed transactions overview — 93. Здесь была глубина по outbox/inbox: почему нельзя проще, механика step by step, timeline при крашах, полная Spring реализация, ShedLock, диагностика в проде.
