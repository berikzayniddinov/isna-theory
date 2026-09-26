# 93. Distributed transactions, Saga и Outbox pattern

## Проблема, которую решают эти паттерны

В монолитной архитектуре с одной базой данных транзакции — понятная штука. Ты открываешь BEGIN, делаешь несколько изменений в разных таблицах, коммитишь. PostgreSQL гарантирует атомарность: все изменения либо применились, либо ни одного. Если приложение упало посередине, при перезапуске PostgreSQL восстановится в consistent состояние через WAL. Разработчик получает ACID «бесплатно», не задумываясь как это работает.

В микросервисной архитектуре одну операцию бизнес-уровня выполняют несколько сервисов, каждый со своей базой. Классический пример: оформить заказ. Сервис заказов создаёт запись заказа. Сервис склада резервирует товар. Сервис платежей списывает деньги. Сервис уведомлений отправляет email. Всё это должно произойти либо целиком, либо не произойти вообще — иначе получим невозможные состояния (деньги списаны, но товар не зарезервирован; заказ создан, но платёж не прошёл).

Проблема в том, что распределённая атомарность в общем случае невозможна. Классическая теория (CAP theorem, FLP impossibility) доказывает: нельзя иметь одновременно strong consistency, availability и partition tolerance в распределённой системе. Приходится выбирать. Микросервисные архитектуры обычно выбирают eventual consistency — данные согласуются не мгновенно, а через некоторое время после того как все обменялись сообщениями. За это платят усложнением логики: приходится думать про промежуточные состояния, retry, идемпотентность, компенсирующие действия.

Распределённые транзакции решают эту задачу разными способами. Самый прямолинейный — **two-phase commit (2PC)** — пытается реализовать классические ACID через координацию между участниками. Работает, но плохо, и микросервисный мир от него отказался. **Saga pattern** — вместо атомарной транзакции последовательность локальных транзакций с явными компенсирующими действиями при отказе. **Outbox pattern** — техника для надёжной доставки сообщений между сервисами вместе с их локальными изменениями. Эти три подхода вместе покрывают большую часть реальных требований.

В этом файле разберём все три подробно. Механику 2PC, почему от него отказались, где всё же применяется. Saga в двух стилях (choreography vs orchestration), примеры реализации, ограничения. Outbox pattern — как гарантировать что сообщение будет доставлено ровно тогда, когда нужно, ни раньше, ни позже. Inbox pattern как дополнение для идемпотентной обработки. Как это всё выглядит в практике на PostgreSQL + Kafka/RabbitMQ.

## Two-phase commit (2PC)

2PC — классический алгоритм атомарной coordinate между несколькими участниками (participants), координатор (transaction manager) обеспечивает что все либо коммитят, либо все откатываются.

Работает так. Есть координатор — процесс, инициирующий транзакцию. Есть участники (два или больше) — базы данных или другие resource managers, у каждого локальный кусок распределённой транзакции.

**Фаза 1 (prepare)**. Координатор говорит каждому участнику «подготовься к коммиту». Каждый участник выполняет свою локальную часть транзакции, но НЕ коммитит окончательно. Вместо этого он записывает в свой лог prepared state — «готов закоммитить, если скажут». Отвечает координатору «prepared» или «abort» (если что-то не получилось).

**Фаза 2 (commit или rollback)**. Если все участники ответили prepared, координатор говорит всем «коммитить». Каждый коммитит окончательно, отвечает done. Транзакция завершена атомарно.

Если хоть один ответил abort в фазе 1, координатор говорит всем «откатить». Все откатываются.

В PostgreSQL есть встроенная поддержка prepared transactions. Вместо обычного `COMMIT` можно сделать `PREPARE TRANSACTION 'gid'`, где gid — глобальный идентификатор транзакции. Транзакция остаётся в prepared state, локи держатся, WAL записан, но окончательного коммита нет. Позже через `COMMIT PREPARED 'gid'` или `ROLLBACK PREPARED 'gid'` завершается решение.

Настройка: `max_prepared_transactions = 100` — сколько prepared transactions можно держать одновременно.

Пример simplified:

```sql
-- База A:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
PREPARE TRANSACTION 'transfer_2024_01_15_001_A';

-- База B:
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
PREPARE TRANSACTION 'transfer_2024_01_15_001_B';

-- Если оба prepared успешно:
COMMIT PREPARED 'transfer_2024_01_15_001_A';
COMMIT PREPARED 'transfer_2024_01_15_001_B';
```

## Проблемы 2PC

2PC математически корректен, но на практике проблематичен.

**Blocking protocol**. Если координатор умер между фазой 1 и фазой 2, участники в prepared state держат локи и не могут решить, коммитить или откатить. В теории они «ждут возвращения координатора». Практически — это может парализовать базу на часы, пока кто-то вручную не разберётся.

Это критическая проблема для availability. `pg_prepared_xacts` показывает stuck transactions:

```sql
SELECT gid, prepared, owner, database, transaction 
FROM pg_prepared_xacts;
```

Если prepared давно и никто не финализирует — DBA должен вручную `ROLLBACK PREPARED 'gid'`. Автоматика сложна: как понять, что координатор точно умер и не восстановится? Как понять, что решение должно быть abort, а не commit?

**Latency**. Каждая транзакция — минимум четыре network round-trip: prepare request, prepare response, commit request, commit response. Плюс fsync на каждом шаге у каждого участника. Для двух участников это 5-10ms latency там где локальная транзакция была бы 1ms. Для 5+ участников уже неприемлемо.

**Тесная связь между сервисами**. Все участники должны знать друг про друга и координатора. Отказ одного участника — вся транзакция не может продвинуться. Это идёт против принципов loose coupling микросервисов.

**Ограниченная поддержка**. Не все системы поддерживают 2PC. NoSQL базы (MongoDB до определённой версии, Cassandra, DynamoDB) — не поддерживают. Message brokers (Kafka, RabbitMQ) — ограниченно. Cross-database (PostgreSQL + MySQL + MongoDB в одной транзакции) — практически невозможно надёжно.

**XA transactions** — стандарт для 2PC в Java-мире, реализуется через JTA (Java Transaction API). Application server (WebSphere, JBoss) выступает как координатор. Работает, но медленно, требует много настройки, все ресурсы должны поддерживать XA.

Из-за всех этих проблем **современные микросервисные архитектуры не используют 2PC для типичных задач**. Используют другие паттерны с eventual consistency. 2PC остаётся в специфических нишах: некоторые финансовые системы с жёсткими требованиями к атомарности, интеграции с legacy enterprise системами, определённые формы репликации.

## Saga pattern

Saga — альтернативный подход к распределённым транзакциям, специально спроектированный для микросервисной архитектуры. Идея пришла из статьи 1987 года («Sagas» Garcia-Molina и Kenneth Salem) и переосмыслена для современности.

Saga заменяет атомарную распределённую транзакцию **последовательностью локальных транзакций** с **компенсирующими действиями** на случай отказа. Каждый шаг saga — отдельная локальная транзакция в одном сервисе. Если один шаг падает, ранее выполненные шаги отменяются через свои компенсации.

Пример «оформить заказ»:

Шаг 1: сервис заказов — CreateOrder (создаёт заказ со статусом PENDING).
Шаг 2: сервис склада — ReserveItems (резервирует товары).
Шаг 3: сервис платежей — ChargePayment (списывает деньги).
Шаг 4: сервис заказов — ConfirmOrder (меняет статус на CONFIRMED).

Если шаг 3 падает (недостаточно денег), выполняются компенсации:
- Компенсация шага 2: ReleaseItems (освобождает резерв).
- Компенсация шага 1: CancelOrder (помечает заказ CANCELLED).

Ключевое отличие от 2PC: **все локальные транзакции commit'ятся сразу**, нет prepared state. Если что-то пошло не так — не rollback, а компенсация (новая транзакция, отменяющая эффект предыдущей).

Плюсы. Каждый сервис — independent, не блокируется другими. Нет распределённых lock'ов. Работает через любые механизмы (базы разных типов, message brokers). Scales горизонтально.

Минусы. Отсутствие isolation. Между шагами saga другие транзакции могут видеть промежуточные состояния (например, заказ CONFIRMED, но деньги ещё не списаны). Разработчик должен явно думать про эти состояния. Разработка сложнее — нужны компенсации для каждого шага, они должны быть надёжны (что если компенсация тоже падает?).

## Saga: choreography vs orchestration

Два стиля реализации saga.

**Choreography**. Каждый сервис знает свою часть saga и сам решает, что делать. Общение через события — сервис публикует событие о завершении своего шага, другие сервисы подписаны и реагируют.

Оформление заказа choreography:
1. Order Service получает запрос CreateOrder → создаёт заказ → публикует OrderCreated event.
2. Inventory Service подписан на OrderCreated → резервирует товары → публикует ItemsReserved event.
3. Payment Service подписан на ItemsReserved → списывает деньги → публикует PaymentCharged event.
4. Order Service подписан на PaymentCharged → confirms заказ → публикует OrderConfirmed event.

При отказе (например, недостаточно денег):
- Payment Service публикует PaymentFailed event.
- Inventory Service реагирует → освобождает резерв → публикует ItemsReleased.
- Order Service реагирует → cancels заказ → публикует OrderCancelled.

Плюсы choreography. Decentralized — нет single point of failure или контроля. Каждый сервис отвечает за свою часть. Легко добавлять новые сервисы (просто подпишись на нужное event).

Минусы. Логика saga размазана по коду многих сервисов. Трудно понять «что происходит когда мы делаем X» — надо смотреть код всех сервисов и их подписки. Циклы: сервис A публикует event → B реагирует → публикует другой event → А снова реагирует → ... сложно отследить. Cascading changes — изменение шага требует изменения многих сервисов.

**Orchestration**. Есть **orchestrator** — отдельный сервис (или компонент), явно управляющий saga. Он последовательно вызывает нужные сервисы, ждёт ответа, решает следующий шаг.

Тот же пример orchestration:
1. Client вызывает OrderSaga.execute(orderData).
2. Orchestrator вызывает Order Service .createOrder() → получает orderId.
3. Orchestrator вызывает Inventory Service .reserveItems(orderId) → success.
4. Orchestrator вызывает Payment Service .chargePayment(orderId) → success/fail.
5. Если success: orchestrator вызывает Order Service .confirmOrder(orderId).
6. Если fail: orchestrator вызывает компенсации в обратном порядке.

Плюсы orchestration. Логика saga в одном месте — легко читать, менять. Явный контроль. Проще отладить (журнал шагов orchestrator'а показывает всё).

Минусы. Orchestrator — single point of failure (или требует своего HA). Coupling — orchestrator знает про все сервисы. Более централизованная архитектура, что противоречит принципам микросервисов в чистом виде.

Практика: для простых saga (2-3 шага, redko меняется) — choreography. Для сложных, с многими шагами и ветвлениями — orchestration. Комбинирование тоже возможно: orchestration для core flow, choreography для side effects.

Инструменты orchestration в Java-мире: **Temporal**, **Cadence**, **Camunda**, **AWS Step Functions**. Дают надёжное исполнение долгих saga (часы, дни, недели), автоматический retry, восстановление после отказа orchestrator'а.

## Идемпотентность

В любой распределённой системе сообщения могут дублироваться (retry после сетевого таймаута, но исходное сообщение всё-таки дошло). Если обработать одно сообщение дважды и получить разные результаты — данные испортятся.

Отсюда фундаментальное требование: **все операции в распределённой системе должны быть идемпотентны**. Идемпотентная операция — та, что можно выполнить несколько раз с тем же результатом что и один раз.

Простой пример. Сообщение «списать 100 денег с счёта 5». Не идемпотентно: два запуска — списано 200. Правильно: «установить balance счёта 5 на значение 900 (было 1000)» — идемпотентно если использует expected old value. Или: «списать 100 денег с счёта 5, transaction_id = tx_abc_123» — сервис проверяет, обрабатывал ли уже tx_abc_123, если да — игнорирует.

Реализация в БД обычно через unique identifier сообщения. Каждое сообщение имеет уникальный ID. Сервис ведёт табличку обработанных ID:

```sql
CREATE TABLE processed_messages (
    message_id UUID PRIMARY KEY,
    processed_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

При обработке:

```sql
BEGIN;
-- Пытаемся вставить message_id
INSERT INTO processed_messages(message_id) VALUES ($1)
ON CONFLICT DO NOTHING;
-- Если conflict — сообщение уже обработано, ничего не делаем
-- Если success — обрабатываем сообщение
UPDATE accounts SET balance = balance - 100 WHERE id = 5;
COMMIT;
```

Все локальные изменения — в одной транзакции с insert'ом в processed_messages. Дубликат сообщения → INSERT падает → транзакция откатывается → данные не меняются.

Это **Inbox pattern**. Каждый сервис ведёт inbox — таблицу с идентификаторами обработанных сообщений. Гарантирует ровно-один-раз обработку даже при duplicate delivery.

## Outbox pattern

Обратная проблема. Сервис делает локальное изменение в БД и должен опубликовать событие в message broker (для других сервисов). Как гарантировать что event будет опубликован в точности когда локальное изменение commit'ится?

Наивный подход:

```java
@Transactional
public void createOrder(OrderRequest req) {
    Order order = new Order(...);
    orderRepository.save(order);              // DB write
    kafkaTemplate.send("orders", new OrderCreatedEvent(...));  // Kafka publish
}
```

Что не так. Если commit прошёл, но Kafka publish упал (network, broker down) — данные в БД, event не опубликован, другие сервисы не узнают. Если Kafka publish прошёл, но commit упал — event опубликован но данных нет, phantom.

Транзакция не атомарна: она включает DB commit и Kafka send, но эти два — независимые операции разных систем. Атомарность через 2PC — как мы разбирали, плохо.

**Outbox pattern** решает это. Идея: вместо прямой публикации в Kafka, сервис записывает сообщение в специальную **outbox table** в той же БД, в той же транзакции с основными изменениями. Отдельный процесс (или thread) читает outbox и публикует в Kafka.

Структура:

```sql
CREATE TABLE outbox (
    id BIGSERIAL PRIMARY KEY,
    aggregate_type VARCHAR(255) NOT NULL,
    aggregate_id VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    published_at TIMESTAMP
);
CREATE INDEX ON outbox (published_at) WHERE published_at IS NULL;
```

Приложение:

```java
@Transactional
public void createOrder(OrderRequest req) {
    Order order = new Order(...);
    orderRepository.save(order);
    // Записываем event в outbox в той же транзакции
    outboxRepository.save(new OutboxEvent(
        "Order", order.getId().toString(), "OrderCreated", 
        objectMapper.writeValueAsString(new OrderCreatedEvent(...))
    ));
}
```

Оба INSERT'а — в одной транзакции. Commit либо прошёл (и order сохранён, и outbox запись есть), либо откатился (ни того, ни другого).

Отдельный процесс — **outbox publisher** — опрашивает outbox:

```java
@Scheduled(fixedRate = 100)
public void publishOutbox() {
    List<OutboxEvent> events = outboxRepository.findUnpublished(100);
    for (OutboxEvent event : events) {
        try {
            kafkaTemplate.send(event.getTopic(), event.getPayload()).get();
            outboxRepository.markPublished(event.getId());
        } catch (Exception e) {
            log.error("Failed to publish, will retry", e);
        }
    }
}
```

Если публикация упала — не помечаем как published, следующий цикл попробует снова. Идемпотентность на стороне consumer'ов через message ID (см. Inbox выше).

Плюсы. Гарантия ровно-один-раз (или at-least-once, а через inbox на consumer стороне — effectively-exactly-once). Прямая транзакционность с локальной БД. Нет распределённых транзакций.

Минусы. Latency — event публикуется не мгновенно, а через задержку polling. Обычно 100-500ms в зависимости от частоты. Нагрузка на БД — outbox нужно постоянно опрашивать. Требуется VACUUM на outbox table.

## Change Data Capture как альтернатива polling

Проблема с polling outbox — небольшая latency и постоянные запросы к БД. Альтернатива — **Change Data Capture (CDC)**: инструмент напрямую читает WAL PostgreSQL и стримит изменения outbox таблицы в Kafka без polling.

**Debezium** — стандартный инструмент CDC. Работает как Kafka Connect connector. Подключается к PostgreSQL через logical replication (создаёт replication slot), декодирует WAL, получает поток INSERT/UPDATE/DELETE. Фильтрует по outbox таблице, публикует в Kafka.

Плюсы Debezium: near-realtime (десятки миллисекунд), нет нагрузки от polling, гарантированный порядок (WAL order), автоматический offset tracking. Стандартно integrated с Kafka.

Минусы: сложнее setup (Kafka Connect кластер, connector конфиг), operational overhead, replication slot должен быть alive (если Debezium умрёт надолго и slot заполнит WAL — база встанет), большая learning curve.

Debezium имеет специальный **outbox event router** — SMT (Single Message Transform), который вытаскивает поле payload из outbox записи и публикует как отдельный event в правильный топик.

Практика: для не очень больших нагрузок (тысячи событий в секунду) — простой polling публикатор подходит. Для высоких нагрузок или требований по latency — Debezium.

## Inbox + Outbox = надёжный обмен

Комбинация паттернов даёт надёжный exactly-once-effectively обмен сообщениями:

Producer сервис:
- Локальные изменения + запись в outbox в одной транзакции.
- Outbox publisher (polling или CDC) публикует в Kafka.
- At-least-once delivery: если publish упал, повторяется. Consumer может получить дубликаты.

Consumer сервис:
- Проверяет inbox (обработано ли message_id).
- Если нет — обрабатывает + записывает в inbox в одной транзакции.
- Дубликаты идемпотентно игнорируются.

Итог: каждое сообщение effectively обрабатывается ровно один раз, несмотря на возможные network failures и retries на любом этапе.

## Saga + Outbox: реальный микросервисный стек

Собираем это в единую архитектуру для КНП-подобной системы.

Order Service принимает POST /orders от клиента:

```java
@Transactional
public Order createOrder(OrderRequest req) {
    // 1. Локальная транзакция: создать order и записать outbox event
    Order order = orderRepository.save(new Order(req, Status.PENDING));
    outboxRepository.save(new OutboxEvent(
        "orders.events", "OrderCreated", order.getId(), 
        toJson(new OrderCreatedEvent(order))
    ));
    return order;
}
```

Outbox publisher постоянно публикует новые events в Kafka топик `orders.events`.

Inventory Service подписан на этот топик. Получает OrderCreated:

```java
@KafkaListener(topics = "orders.events")
@Transactional
public void handle(OrderCreatedEvent event) {
    // 1. Проверить inbox
    if (inboxRepository.exists(event.getMessageId())) {
        return; // уже обработали
    }
    inboxRepository.save(new InboxEntry(event.getMessageId()));
    
    // 2. Бизнес-логика: зарезервировать товары
    boolean reserved = tryReserve(event.getOrderId(), event.getItems());
    
    // 3. Публикуем результат
    if (reserved) {
        outboxRepository.save(new OutboxEvent(
            "inventory.events", "ItemsReserved", event.getOrderId(), 
            toJson(new ItemsReservedEvent(...))
        ));
    } else {
        outboxRepository.save(new OutboxEvent(
            "inventory.events", "ReservationFailed", event.getOrderId(),
            toJson(new ReservationFailedEvent(...))
        ));
    }
}
```

Payment Service подписан на inventory.events, реагирует на ItemsReserved, списывает деньги, публикует свой event.

Order Service подписан на все свои downstream events. Реагирует на ItemsReserved / PaymentCharged / ReservationFailed / PaymentFailed. Обновляет статус заказа соответственно.

Всё это — choreography saga. Каждый сервис знает только про свой input и свой output events. Логика saga размазана, но с ясными правилами. При любом failure — publish compensation event, downstream сервисы реагируют.

Мониторинг saga: обычно ведётся отдельно, view над всеми events по конкретному saga (по order_id). Показывает: OrderCreated → ItemsReserved → PaymentFailed → ItemsReleased → OrderCancelled. Легко debug'ить.

## Ограничения eventual consistency

Saga даёт eventual consistency, но не strong. Это означает конкретные вещи, которые разработчику нужно понимать.

**Промежуточные состояния видимы**. Между CreateOrder и PaymentCharged заказ существует со статусом PENDING. Если приходит другой запрос «покажи все заказы user'а», он увидит этот PENDING заказ. Приложение должно уметь работать с промежуточными состояниями — обычно через явные статусы в domain модели.

**Rollback не instant**. Если compensation в saga — «cancel order», между принятием этого решения и реальным изменением статуса order всё ещё виден как active. Обычно измеряется секундами. Приложение должно устраивать эта задержка.

**Read-your-writes не гарантирован**. Client создал order, сразу делает GET /orders/{id}. Если запрос попадёт на другой инстанс Order Service до того как каскадные updates применятся — может увидеть неcomplete state. Fix: sticky session на балансере, или client опрашивает пока не увидит нужный state.

**Порядок сообщений не гарантирован строго**. Kafka с одной partition гарантирует порядок только внутри partition. Между partitions — нет. Нужно быть аккуратным с партиционированием (обычно по aggregate ID).

**Duplicate events случаются**. Inbox гарантирует идемпотентную обработку, но код обработки должен помнить об этом. Всё, что не идемпотентно (например, отправка email) — требует отдельной защиты (проверить, не отправлено ли уже).

## Когда saga не нужна

Не любой процесс с несколькими сервисами требует saga. Простые случаи можно решить проще.

**Read-only aggregation**. Aggregation report — просто call API нескольких сервисов и объединить. Не нужна ни транзакция, ни saga. Если один сервис недоступен — partial result или error.

**Fire-and-forget notifications**. Опубликовал event, не важно что случится дальше. Consumer'ы обрабатывают independently, никакие компенсации не нужны.

**Асинхронные workflow без rollback semantics**. Ты запросил heavy report, backend начал его строить, notify user когда готово. Если строится сбоил — user не увидит, но других изменений нет. Не saga.

**Одиночная база даже в микросервисе**. Часто один микросервис держит несколько логически связанных aggregate. Изменения в них — обычная локальная транзакция PostgreSQL. Не saga, не distributed transaction.

Saga нужна, когда есть **несколько сервисов с локальным state**, каждый может отказать, требуется бизнес-consistent результат.

## Заключение

Distributed transactions — фундаментальная сложность микросервисной архитектуры. Одна операция бизнес-уровня выполняется в нескольких сервисах, каждый со своей БД. Классические ACID гарантии одной базы не масштабируются на distributed сценарий.

Two-phase commit — прямолинейное решение, работает через координированный prepare + commit. Проблемы: blocking (координатор умер — участники висят в prepared state), latency (много round-trip'ов), тесная связь между сервисами. Практически не используется в современных микросервисах, кроме специфических случаев с legacy integration.

Saga pattern — de facto стандарт. Последовательность локальных транзакций с компенсирующими действиями. Choreography (сервисы общаются через events, decentralized) vs Orchestration (отдельный orchestrator управляет процессом, centralized). Eventual consistency вместо strong — промежуточные состояния видимы, между шагами возможны partial states.

Outbox pattern решает проблему надёжной публикации event'ов вместе с локальными изменениями. Event пишется в outbox table в той же транзакции с основными данными; отдельный процесс (polling или CDC через Debezium) публикует в message broker. Гарантирует at-least-once delivery.

Inbox pattern на consumer стороне обеспечивает идемпотентную обработку. Каждое сообщение имеет уникальный ID, consumer записывает processed IDs в inbox table в транзакции с обработкой. Дубликаты игнорируются.

Комбинация Outbox + Inbox + идемпотентные операции даёт effectively-exactly-once обработку через at-least-once transport. Работает надёжно, но требует дисциплины: каждая операция идемпотентна, каждый сервис ведёт inbox/outbox, monitoring для отставания в publisher'ах.

Debezium — реалистичная альтернатива polling outbox для high-throughput систем. CDC через WAL, near-realtime, но операционная сложность.

Ограничения eventual consistency: промежуточные состояния видимы, read-your-writes не гарантирован, порядок сообщений между партициями не гарантирован, дубликаты возможны. Приложение должно быть спроектировано с учётом этих реалий.

Не всё требует saga. Read-only aggregation, fire-and-forget notifications, workflow без rollback — проще без неё. Saga — для настоящих многосервисных транзакций с бизнес-требованиями consistency.

Для КНП typical pattern: события между сервисами через Kafka, каждый сервис ведёт свой outbox и inbox, saga для сложных бизнес-процессов (обработка обращения, calculation cascade), orchestration через отдельный workflow service где логика сложная. Мониторинг задержек outbox publisher'а — важная метрика, если растёт — что-то не так с Kafka или сетью.

Дальше — практика. Реализуй простой Saga на двух сервисах в docker-compose (Order + Payment, PostgreSQL + Kafka + Debezium). Симулируй отказы: Payment down, Kafka down, Debezium down. Посмотри, как система восстанавливается. Замерь latency и duplicates при разных failure modes. Понимание приходит через реальные ошибки.
