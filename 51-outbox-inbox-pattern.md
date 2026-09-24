# 51. Outbox, Inbox и идемпотентность: надёжный обмен сообщениями между сервисами

## Зачем это знать

Разработчик, впервые столкнувшийся с микросервисами, обычно пишет публикацию событий самым простым способом: сохранил сущность в БД, отправил событие в Kafka. Пока это работает на тестовом стенде — считает задачу завершённой. Но production приносит неожиданности. Что происходит если процесс упал между `commit` базы и `send` в Kafka? Что если Kafka доступна, но временно медленная — `send` истекает по таймауту после того как коммит уже прошёл? Что если consumer обработал сообщение, но упал до того как закоммитил offset — сообщение приходит повторно, бизнес-операция дублируется?

Разница между «пишу publishing событий» и «понимаю reliable messaging» проявляется в postmortem'ах после инцидентов. Первый видит «событие пропало между сервисами» и заводит тикет «расследовать». Второй знает фундаментальную проблему: атомарная запись в БД **и** брокер сообщений невозможна без 2PC/XA — а XA для Kafka недоступна и вредна для performance. Знает что **Outbox pattern** решает это переносом проблемы: событие пишется в БД в той же транзакции что и бизнес-данные, отдельный publisher читает outbox и публикует в брокер. Гарантия — если БД закоммитила, событие будет опубликовано eventually. Знает что дубликаты у consumer'а неизбежны из-за at-least-once семантики брокера — и **Inbox pattern** (или другой механизм идемпотентности) обязателен для безопасной обработки.

Разберём эти паттерны глубоко. Фундаментальная проблема — почему атомарность БД + брокер невозможна напрямую. Механика Outbox: структура таблицы, логика producer'а, publisher polling loop. Debezium/CDC как альтернативный подход. Inbox pattern для дедупликации на стороне consumer'а. Комбинация Outbox + Inbox = effectively exactly-once. Другие подходы к идемпотентности — идемпотентный UPDATE, UPSERT, optimistic version, Idempotency-Key header для HTTP API. Полная реализация в Spring с ShedLock для multi-instance publisher. Реальные проблемы прода — рост outbox, лаг publisher'а, ordering, dead-letter, эволюция схемы. Альтернативы и их ограничения.

## Фундаментальная проблема

Простой пример. `OrderService.createOrder` должен: сохранить Order в PostgreSQL и опубликовать `OrderCreated` в Kafka или RabbitMQ.

Что произойдёт если между этими шагами что-то упадёт? Несколько failure modes.

**Вариант А. Сначала БД, потом broker**:

```java
@Transactional
public void createOrder(OrderRequest req) {
    Order o = new Order(req);
    orderRepo.save(o);           // DB commit
    // CRASH здесь
    kafka.send("orders", o);     // НЕ выполнено
}
```

Заказ сохранён в БД. Событие не опубликовано. Downstream-сервисы не узнают про новый order. Inventory не забронирует товар. Payment не будет charged. Бизнес-процесс остался незавершённым.

**Вариант Б. Сначала broker, потом БД**:

```java
public void createOrder(OrderRequest req) {
    kafka.send("orders", ...);   // опубликовано
    // CRASH здесь
    orderRepo.save(o);            // не сохранено
}
```

Событие опубликовано. Order в БД не создан. Downstream считает что заказ существует. Inventory резервирует товар для несуществующего заказа. Data inconsistency, которую придётся руками разгребать.

Оба подхода падают при крашах процесса. Ни один не reliable.

**XA / 2PC** теоретически решает — распределённая транзакция, включающая и БД и Kafka. Практически не работает: Kafka не поддерживает XA, overhead на performance огромный, coordinator — single point of failure. Подробнее в файле 50 про saga-pattern.

Решение — **Outbox pattern**. Сдвигает проблему от «atomic write across two systems» к «atomic write within one system + eventual publish».

## Механика Outbox

Идея простая. Сохранить и бизнес-данные, и описание события — в одной DB-транзакции. Отдельный процесс потом читает outbox-таблицу, публикует в брокер, отмечает как processed.

Гарантия: если DB commit прошёл — outbox-запись существует — событие рано или поздно опубликуется. Retry в publisher'е обеспечивает eventual delivery.

**Структура таблицы**:

```sql
CREATE TABLE outbox (
    id BIGSERIAL PRIMARY KEY,
    aggregate_type TEXT NOT NULL,          -- 'Order', 'Payment'
    aggregate_id TEXT NOT NULL,             -- '42'
    event_type TEXT NOT NULL,               -- 'OrderCreated'
    payload JSONB NOT NULL,                 -- сериализованное событие
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ                -- NULL если не опубликовано
);

CREATE INDEX idx_outbox_unprocessed 
    ON outbox(created_at) WHERE processed_at IS NULL;
```

`aggregate_type` определяет тип сущности (Order, Payment). Часто используется как имя топика или routing key. `aggregate_id` — ID сущности, используется как partition key для сохранения порядка событий per entity. `event_type` — тип события (OrderCreated, OrderCancelled), определяет обработку consumer'ом. `payload` — сериализованное тело события (обычно JSON). `created_at` + `processed_at` отслеживают жизненный цикл.

**Partial index на unprocessed events** — критически важен. Publisher делает `WHERE processed_at IS NULL` — обычный индекс покроет всю таблицу (миллионы строк), partial индекс — только неопубликованные (тысячи в моменте). Разница в производительности на большой outbox — десятки раз.

**Producer в сервисе делает атомарную запись**:

```java
@Service
class OrderService {

    @Autowired OrderRepository orderRepo;
    @Autowired OutboxRepository outboxRepo;
    @Autowired ObjectMapper mapper;

    @Transactional
    public Order createOrder(OrderRequest req) {
        Order o = new Order(req);
        orderRepo.save(o);

        OutboxEntry entry = new OutboxEntry();
        entry.setAggregateType("Order");
        entry.setAggregateId(o.getId().toString());
        entry.setEventType("OrderCreated");
        entry.setPayload(mapper.writeValueAsString(new OrderCreatedEvent(o)));
        outboxRepo.save(entry);

        return o;
        // на commit — и Order, и OutboxEntry сохраняются атомарно
    }
}
```

Всё в одной транзакции. Если commit упал — обе записи откатились. Если прошёл — обе persisted. Никогда не бывает частичного состояния.

**Publisher (polling)** периодически читает неопубликованные:

```java
@Component
class OutboxPublisher {

    @Autowired OutboxRepository outboxRepo;
    @Autowired KafkaTemplate<String, Object> template;
    @Autowired ObjectMapper mapper;

    @Scheduled(fixedDelay = 500)
    @Transactional
    public void publish() {
        List<OutboxEntry> pending = 
            outboxRepo.findTop100ByProcessedAtIsNullOrderByCreatedAt();
            
        for (OutboxEntry entry : pending) {
            try {
                template.send(
                    entry.getAggregateType().toLowerCase() + "s",
                    entry.getAggregateId(),
                    entry.getPayload()
                ).get(5, TimeUnit.SECONDS);
                
                entry.setProcessedAt(Instant.now());
                outboxRepo.save(entry);
            } catch (Exception e) {
                log.error("Не удалось опубликовать outbox {}", entry.getId(), e);
                // не отмечаем как processed — повторим на следующей итерации
            }
        }
    }
}
```

Периодический fetch пачки неопубликованных, publish каждого, отметка processed. При ошибке — оставляем неотмеченным, следующий цикл повторит.

**Важный caveat — дубли возможны**. Если publish в брокер прошёл, но `processed_at` не обновился (краш между операциями) — следующий цикл опубликует ещё раз. Это **at-least-once delivery**. Consumer обязательно должен быть идемпотентным.

**Batching через LIMIT 100** предотвращает overwhelming publisher'а при массивной outbox. Множественные итерации постепенно обрабатывают backlog.

**Ordering**. Сортировка по `created_at`, `aggregate_id` как partition key в Kafka. События одной сущности сохраняют порядок через partition. События разных сущностей могут переупорядочиться — это обычно нормально, разные бизнес-сущности независимы.

**Retention**. Неограниченный рост outbox неприемлем. Периодическая очистка:

```sql
DELETE FROM outbox WHERE processed_at < now() - INTERVAL '7 days';
```

Либо партиционирование по `created_at` с dropping старых партиций — эффективнее для больших объёмов.

## Debezium и CDC — альтернатива polling'у

Вместо периодического опроса — **Change Data Capture** через Debezium. Читает WAL PostgreSQL напрямую, публикует изменения в Kafka в реальном времени.

**Как работает**. Debezium подключается к PostgreSQL через logical replication (нужен `wal_level=logical` и созданный replication slot). Каждое изменение (INSERT/UPDATE/DELETE) становится событием в Kafka. Consumers читают как обычно.

**Setup требует**: `wal_level=logical` в конфигурации PostgreSQL; replication slot созданный для Debezium; Debezium connector сконфигурированный в Kafka Connect; кому-то надо поддерживать Kafka Connect infrastructure.

**Outbox с Debezium** — через специальный SMT (Single Message Transform) — **Outbox Event Router**. Пишешь в outbox-таблицу — Debezium автоматически публикует в топик, определяемый через `aggregate_type`. Custom publisher не нужен вовсе.

**Плюсы**: никакого polling — real-time propagation с задержкой в миллисекунды; никакого custom publisher — Debezium делает всё; ordering через partition key; легко масштабируется.

**Минусы**: требует Kafka Connect + Debezium infrastructure; настройка logical replication PostgreSQL; ещё один компонент, который надо мониторить и обновлять.

**Прямой CDC без outbox**. Можно натравить Debezium прямо на бизнес-таблицы (Orders, Payments) — публиковать все изменения. Против этого подхода:

- Broker получает **implementation details таблиц** — вся схема, все колонки exposed.
- Coupling — изменение таблицы ломает consumers.
- Business semantic не clean — technical INSERT/UPDATE events вместо business events (OrderCreated, OrderCancelled).

**Outbox абстрагирует таблицы**. Публикует business events (`OrderCreated`), не technical events (row inserted в таблицу X). Чистый контракт с consumers. Правильный подход когда выбираешь Debezium.

## Inbox pattern — дедупликация на стороне consumer

Обратная сторона задачи. Как consumer защищается от повторов?

Producer публикует at-least-once (даже с outbox). Consumer может получить одно сообщение несколько раз. Причины: retry после сбоя, rebalance в consumer group триггерит re-delivery, рестарт без закоммиченного offset.

Если consumer пишет в БД — повторная обработка = дубликат записи. Вызывает external API — повторный вызов. Отправляет email — email уходит дважды.

**Inbox pattern**: consumer поддерживает таблицу processed message IDs:

```sql
CREATE TABLE inbox (
    message_id TEXT PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Consumer logic**:

```java
@Transactional
public void handle(OrderCreatedEvent event) {
    if (inboxRepo.existsById(event.getMessageId())) {
        log.debug("Пропускаем дубликат {}", event.getMessageId());
        return;
    }
    processOrder(event);
    inboxRepo.save(new InboxEntry(event.getMessageId()));
    // на commit — и бизнес-действия, и inbox запись атомарны
}
```

Проверка + обработка + запись в inbox — в одной транзакции. Атомарность гарантируется базой. Если краш между обработкой и сохранением в inbox — при retry проверяем против inbox, если нет — reprocess. Если saved — skip.

**Уникальный `messageId` — обязательное требование**. Producer должен генерировать unique ID на каждое сообщение (обычно UUID). При retry — тот же messageId. Иначе дедупликация не работает.

В Kafka messageId можно генерировать явно в headers или брать связку `topic-partition-offset` как естественный уникальный ключ. В RabbitMQ — устанавливать вручную через `MessageProperties.setMessageId`. Явный контроль необходим.

**Retention для inbox**:

```sql
DELETE FROM inbox WHERE processed_at < now() - INTERVAL '30 days';
```

Retention должен быть **больше** чем максимально возможная задержка повторной доставки. Если Kafka retention 7 дней, inbox должен хранить минимум 7+ дней. Иначе повторное сообщение, пришедшее после cleanup inbox, будет обработано как новое.

## Outbox + Inbox = effectively exactly-once

Комбинация обеспечивает end-to-end reliability:

```
[Service A]                              [Service B]
    │                                          │
    │  @Transactional                          │
    │  save Order + save OutboxEntry           │
    │  commit                                  │
    │                                          │
    │  OutboxPublisher polls                   │
    │  publishes to Kafka                      │
    │                                          │
    │              Kafka                       │
    │              │                            │
    │              ▼                            │
    │                                        [Consumer]
    │                                          │
    │                                          │  @Transactional
    │                                          │  check inbox
    │                                          │  если нет:
    │                                          │      process
    │                                          │      save inbox entry
    │                                          │  commit
```

Итог. Producer-side атомарно — БД + publish (через outbox). Broker at-least-once — может дублировать. Consumer-side inbox фильтрует дубли. End-to-end — **effectively exactly-once**.

Это не «true exactly-once» в теоретическом CS-смысле (математически невозможно в распределённых системах). Но с точки зрения бизнес-наблюдаемой семантики — эквивалентно exactly-once. Дубли обнаружены, проигнорированы, бизнес-состояние согласовано.

## Другие подходы к идемпотентности

Inbox — один конкретный подход. Другие тоже работают, каждый в своих сценариях.

**Идемпотентный UPDATE через conditional clause**:

```sql
UPDATE orders SET status='PROCESSED' 
WHERE id=? AND status='NEW';
-- rows affected = 0 → уже обработано
-- rows affected = 1 → успех
```

Работает если операция — state machine transition. Проверяем текущий статус перед update. Затрагивает только если состояние соответствует ожидаемому precondition. Повторные вызовы после первого успеха — no-op.

**UPSERT для inserts**:

```sql
INSERT INTO users (id, email) VALUES (?, ?)
ON CONFLICT (id) DO NOTHING;
```

Или `DO UPDATE SET email = EXCLUDED.email` для upsert с семантикой обновления. Второй вызов с тем же ID не создаёт дубликат.

**Optimistic version через version columns**:

```sql
UPDATE orders SET status='X', version=version+1 
WHERE id=? AND version=?;
```

Не сработает если версия уже изменилась — кто-то опередил. Обнаружение concurrent modification. Обычно используется в JPA через `@Version`.

**Idempotency-Key header для HTTP API**. Клиент шлёт `Idempotency-Key: <unique-uuid>` в заголовке. Сервер:

- Первый вызов с этим ключом → обрабатывает запрос, сохраняет ответ.
- Повторный вызов с тем же ключом → возвращает сохранённый ответ, не выполняя операцию повторно.

Используется в Stripe API — обязательный паттерн для payment endpoints. Клиент отвечает за генерацию и tracking ключей. Сервер поддерживает key-to-response mapping (обычно с TTL 24 часа).

## Полная реализация в Spring

**Entity + repository**:

```java
@Entity
@Table(name = "outbox")
public class OutboxEntry {
    @Id @GeneratedValue 
    Long id;
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

**Сервис использует**:

```java
@Service
class OrderService {

    @Autowired OrderRepository orderRepo;
    @Autowired OutboxRepository outboxRepo;
    @Autowired ObjectMapper mapper;

    @Transactional
    public Order createOrder(OrderRequest req) {
        Order o = new Order(req);
        orderRepo.save(o);

        OutboxEntry entry = new OutboxEntry();
        entry.setAggregateType("Order");
        entry.setAggregateId(o.getId().toString());
        entry.setEventType("OrderCreated");
        try {
            entry.setPayload(mapper.writeValueAsString(
                new OrderCreatedEvent(o.getId(), o.getCustomerId(), ...)));
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
        outboxRepo.save(entry);

        return o;
    }
}
```

**Publisher scheduled job**:

```java
@Component
@Slf4j
class OutboxPublisher {

    @Autowired OutboxRepository outboxRepo;
    @Autowired KafkaTemplate<String, String> template;

    @Scheduled(fixedDelay = 500)
    @Transactional
    public void publish() {
        List<OutboxEntry> pending = 
            outboxRepo.findTop100ByProcessedAtIsNullOrderByCreatedAt();

        for (OutboxEntry entry : pending) {
            try {
                template.send(
                    entry.getAggregateType().toLowerCase() + "s",   // топик
                    entry.getAggregateId(),                          // key
                    entry.getPayload()                                // JSON
                ).get(5, TimeUnit.SECONDS);

                entry.setProcessedAt(Instant.now());
                outboxRepo.save(entry);
            } catch (Exception e) {
                log.error("Не удалось опубликовать outbox {}", entry.getId(), e);
                // не отмечаем — следующий цикл повторит
            }
        }
    }
}
```

Send с timeout — избегает подвисания publisher'а если Kafka медленная. `get()` дожидается фактической отправки синхронно, ошибка отправки бросает exception. Failed sends остаются неотмеченными для retry.

## ShedLock для multi-instance publisher

Если приложение развёрнуто в K8s с 3 репликами — все 3 будут polling outbox → двойные (тройные) публикации. Consumer должен разбираться, но лучше не создавать проблему на producer side.

**ShedLock** — distributed lock на scheduled jobs через shared БД:

```gradle
implementation 'net.javacrumbs.shedlock:shedlock-spring:5.10.0'
implementation 'net.javacrumbs.shedlock:shedlock-provider-jdbc-template:5.10.0'
```

```java
@Scheduled(fixedDelay = 500)
@SchedulerLock(
    name = "OutboxPublisher_publish", 
    lockAtLeastFor = "PT100MS", 
    lockAtMostFor = "PT30S"
)
public void publish() { ... }
```

Только одна реплика будет выполнять job в конкретный момент. Lock хранится в shared таблице `shedlock` в БД — все instances видят одну и ту же lock-таблицу через optimistic INSERT/UPDATE.

`lockAtLeastFor` — минимальное время удержания lock (защита от clock skew). `lockAtMostFor` — максимальное (страховка если инстанс упал не сняв lock — через это время его освободят автоматически).

**Реальный кейс из КНП** (память `knp-fno21-shedlock-stale-image-dup-regnum`). Без ShedLock scheduled job лупился параллельно на всех репликах → дублирующиеся INSERT'ы с одинаковыми регистрационными номерами. Deploy с ShedLock решил проблему в течение часа.

## Реальные проблемы прода

**Outbox таблица растёт быстро**. При активной публикации — десятки тысяч записей в час. Без cleanup — гигабайты за неделю, индексы деградируют. Периодический DELETE старых обработанных или партиционирование по `created_at` с drop старых партиций. Мониторить размер таблицы отдельной метрикой.

**Publisher lag**. Если publisher не успевает (много sends, Kafka медленная) → outbox копится → latency событий растёт. Симптом: growing `count(*) WHERE processed_at IS NULL`. Что делать: увеличить частоту scheduled (был 500ms → 100ms); увеличить batch size (100 → 500); распараллелить (несколько publisher'ов на разные partition ranges); или перейти на Debezium для higher throughput.

**Ordering**. `aggregate_id` как partition key в Kafka сохраняет порядок per business entity. События одного Order всегда в одной partition, читаются в порядке отправки. События разных Order могут переупорядочиться — обычно нормально, они независимы.

**Dead-letter обработка**. Если событие сериализовалось некорректно или consumer не может обработать даже после retry — нужен путь для этих сообщений. На стороне outbox — добавить колонку `error_count` и `last_error`, после N попыток перекладывать в DLT (dead-letter topic) или отдельную error-таблицу для ручного разбора. На стороне consumer — Spring Kafka ErrorHandler с DeadLetterPublishingRecoverer.

**Schema evolution**. События сохраняются в outbox с текущей схемой. При изменении схемы события в БД могут стать невалидными при десериализации consumer'ом. Правила: только backward compatible изменения (добавление nullable полей — ок, удаление или переименование — нет). Versioning в payload (`schemaVersion: 2`). Consumer поддерживает несколько версий одновременно в течение переходного периода.

## Диагностика в проде

Инцидент: consumer жалуется что не приходят события. Или наоборот — получает дубли.

**События не приходят**. Первое — проверить outbox:

```sql
SELECT count(*) FROM outbox WHERE processed_at IS NULL;
SELECT id, created_at, aggregate_type, event_type 
FROM outbox 
WHERE processed_at IS NULL 
ORDER BY created_at 
LIMIT 20;
```

Растёт `unprocessed`, самые старые — часы назад → publisher не работает или отстаёт. Проверить логи `OutboxPublisher` на ошибки Kafka. Проверить ShedLock: `SELECT * FROM shedlock WHERE name = 'OutboxPublisher_publish'` — не завис ли lock (`locked_until` в будущем далеко, но `locked_by` умерший инстанс).

Если `unprocessed = 0` — publisher работает, значит проблема на стороне брокера или consumer'а. `kafka-consumer-groups.sh --describe --group my-group` покажет lag consumer group. Растущий lag — consumer тормозит или упал.

**События дублируются**. Первое — проверить inbox consumer'а: он вообще проверяет messageId? Если да — проверить что messageId уникален (не берётся ли `Order.getId()` например, который может повториться в разных событиях жизненного цикла одного заказа). Правильный messageId — UUID именно на само сообщение, не на сущность.

Если inbox работает — дубли идут в реальности из producer'а. Это ожидаемо от Outbox pattern (at-least-once), consumer обязан справляться.

## Альтернативы Outbox и их ограничения

**Простая цепочка без outbox**:

```java
@Transactional
public void createOrder(...) {
    orderRepo.save(o);
    kafka.send(...);
}
```

Проблема: если `kafka.send()` выполнится, а транзакция откатится (например, из-за constraint violation в саму последнюю секунду) — фиктивное событие в брокере, orderа в БД нет. Или наоборот — send упал, tx закоммитилась — событие потеряно. Даже с `@TransactionalEventListener(AFTER_COMMIT)` — если процесс упадёт между commit и send, событие теряется.

Outbox надёжнее с точки зрения гарантий доставки.

**Kafka transactions с JPA**:

```java
@Transactional("kafkaTransactionManager")
public void createOrder(...) {
    orderRepo.save(o);
    kafka.send(...);
}
```

Проблема: `kafkaTransactionManager` управляет только транзакцией Kafka. JPA — отдельно. Нельзя атомарно закоммитить обе (без XA, которого нет в Kafka). `ChainedTransactionManager` (deprecated) — best-effort, не гарантия.

**Event Sourcing**. Другой подход — не хранить текущее состояние, только события. События — source of truth. Публикация автоматически при append новых событий. Совсем другая архитектура, сложность значительно выше. Оправдан только в specific сценариях (audit-heavy домены, complex temporal queries).

## Best practices

**Outbox для гарантии «БД commit → событие опубликуется»**. Фундаментальный reliability паттерн для микросервисов, публикующих события.

**Inbox или другой idempotency-ключ для consumer**. Комплиментарен outbox — предотвращает дубли на стороне обработки.

**UUID как messageId**. Universally unique — никаких коллизий между сессиями и инстансами.

**Retention outbox/inbox таблиц**. Предотвращает unbounded рост. Weekly или monthly cleanup schedules.

**ShedLock для publisher на multiple instances**. Предотвращает параллельную обработку одних outbox-записей несколькими репликами.

**Debezium для real-time / high volume** — вместо polling. Real-time propagation, но требует Kafka Connect infrastructure.

**Aggregate ID как partition key** — сохранение порядка per business entity.

**Мониторинг**: unprocessed outbox count (alert если растёт устойчиво); размер inbox (retention работает?); publisher lag (метрика свежести); ошибки publisher'а в логах.

## Заключение

Outbox pattern решает атомарность «БД save + event publish» через перенос проблемы. Событие сохраняется в БД в той же транзакции что и бизнес-данные. Отдельный publisher читает outbox, публикует в брокер, отмечает как processed. Гарантия at-least-once (не exactly-once): publisher может послать дубли если крашнется после send но перед update `processed_at`.

Debezium/CDC — real-time альтернатива polling'у. Читает WAL PostgreSQL напрямую. Требует Kafka Connect. Рекомендован для high-volume систем.

Inbox pattern для дедупликации на стороне consumer'а. Таблица с processed messageId. Проверка + обработка + запись в одной транзакции. Атомарность гарантирует отсутствие дублирующей обработки.

Outbox + Inbox = **effectively exactly-once**. Не теоретически чистый, но бизнес-наблюдаемо эквивалентный.

Идемпотентность обеспечивается через: уникальный messageId + inbox; conditional UPDATE (state machine transitions); UPSERT для inserts; optimistic version columns; Idempotency-Key HTTP header.

ShedLock для multi-instance publisher. Предотвращает параллельное выполнение job'ов через distributed lock. Реальный кейс `knp-fno21-shedlock-stale-image-dup-regnum` показал необходимость на практике.

Retention обязательна — outbox/inbox расти вечно не могут. Delete старых или партиционирование для массивных объёмов.

Мониторинг publisher lag критичен. Растущий lag = проблема с capacity, надо увеличивать частоту или переходить на Debezium.

Ordering сохраняется через aggregate_id как partition key.

Альтернативы (chained saga без outbox, Kafka transactions с JPA, Event Sourcing) — либо ненадёжны, либо требуют совсем другой архитектуры.

Дальше — паттерны resilience микросервисов: Circuit Breaker, Retry, Bulkhead, Rate Limiting, Fallback.
