# 51. Outbox, Inbox, Idempotency patterns

## Зачем нужны эти patterns

Разработчик впервые encountering microservices обычно пишет event publishing простым способом. Save entity к database, then send event to Kafka. Если работает в testing environments — считает done. Production reality приносит неожиданности. Что если process crashed между database commit и Kafka send? Что если Kafka available но temporarily slow — publish times out после commit already succeeded? Что если consumer processed message but crashed before offset committed — message re-delivered, business operation duplicated?

Разница между разработчиком «пишущим event publishing» и «понимающим reliable messaging» проявляется в production incident postmortems. Первый видит «событие потерялось между сервисами» и пишет ticket «investigate». Второй знает fundamental problem — atomic write across database и message broker impossible без 2PC/XA (unavailable для Kafka, undesirable для performance). Знает что Outbox pattern решает через shifting problem — write event к database в same transaction as business data, separate publisher reads outbox и publishes к broker. Гарантия — если DB commit прошёл то event будет published eventually. Знает что consumer duplicates возможны из-за at-least-once semantics broker — Inbox pattern (или equivalent idempotency mechanism) обязателен для safe processing.

В этом файле разберём эти patterns глубоко. Fundamental problem — атомарность БД plus broker невозможна прямо. Outbox pattern механика — struct таблицы, producer logic, publisher polling loop. Debezium/CDC как alternative подход. Inbox pattern для consumer side deduplication. Combined Outbox plus Inbox = effectively exactly-once. Broader idempotency approaches — идемпотентные UPDATE, UPSERT, optimistic version, Idempotency-Key header для HTTP APIs. Complete Spring implementation с ShedLock для multi-instance publisher. Realistic issues — outbox growth, publisher lag, ordering, dead-letter, schema evolution. Alternatives и their limitations.

## Fundamental problem

Пример. OrderService.createOrder должен. Сохранить Order в PostgreSQL. Опубликовать OrderCreated event в Kafka или RabbitMQ.

Что если между шагами что-то падает? Multiple failure modes.

Вариант А. Сначала БД, потом broker:
```java
@Transactional
public void createOrder(OrderRequest req) {
    Order o = new Order(req);
    orderRepo.save(o);           // DB commit
    // CRASH здесь
    kafka.send("orders", o);     // НЕ выполнено
}
```

Заказ сохранён в database. Событие не опубликовано. Downstream не узнает про новый order. Inventory не забронирует. Payment не будет charged. Lost update — business process incomplete.

Вариант Б. Сначала broker, потом БД:
```java
public void createOrder(OrderRequest req) {
    kafka.send("orders", ...);   // published
    // CRASH здесь
    orderRepo.save(o);            // не сохранено
}
```

Событие опубликовано. Order в database не создан. Downstream believes order exists. Inventory reserves goods для несуществующего order. Data inconsistency.

Both approaches fail при process crashes. Neither reliable.

XA/2PC теоретически решает — distributed transaction spanning database и Kafka. Practically не работает. Kafka не supports XA. Performance overhead massive. Coordinator SPOF. Discussed в файле 50 saga-pattern подробно.

Решение — Outbox pattern. Shifts problem от «atomic write across two systems» к «atomic write within one system plus eventually publish».

## Outbox pattern механика

Идея. Сохранить и бизнес-данные, и описание события в одной DB-транзакции. Отдельный процесс потом читает outbox таблицу, публикует к broker, отмечает как processed.

Гарантия. Если DB commit прошёл — outbox запись есть — event рано или поздно опубликуется. Publisher retry обеспечивает eventual delivery.

Структура таблицы:
```sql
CREATE TABLE outbox (
    id BIGSERIAL PRIMARY KEY,
    aggregate_type TEXT NOT NULL,          -- 'Order', 'Payment'
    aggregate_id TEXT NOT NULL,             -- '42'
    event_type TEXT NOT NULL,               -- 'OrderCreated'
    payload JSONB NOT NULL,                 -- serialized event
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ                -- NULL если не published
);

CREATE INDEX idx_outbox_unprocessed ON outbox(created_at) WHERE processed_at IS NULL;
```

aggregate_type identifies тип entity (Order, Payment). Used often as topic name или routing key. aggregate_id это ID сущности. Used как partition key для preserving ordering per entity. event_type это тип события — OrderCreated, OrderCancelled. Determines consumer handling. payload это сериализованное event body (typically JSON). created_at plus processed_at track lifecycle.

Partial index на unprocessed events makes queries fast. Publisher queries WHERE processed_at IS NULL — index dramatically speeds this up especially при large outbox.

Producer в service делает атомарную запись:
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
        // на commit — и Order, и OutboxEntry атомарно
    }
}
```

Все в одной transaction. Если commit failed — оба откатились. Если commit прошёл — оба persisted. Никогда partial states.

Publisher (polling) периодически reads unprocessed:
```java
@Component
class OutboxPublisher {

    @Autowired OutboxRepository outboxRepo;
    @Autowired KafkaTemplate<String, Object> template;
    @Autowired ObjectMapper mapper;

    @Scheduled(fixedDelay = 500)
    @Transactional
    public void publish() {
        List<OutboxEntry> pending = outboxRepo.findTop100ByProcessedAtIsNullOrderByCreatedAt();
        for (OutboxEntry entry : pending) {
            try {
                Object event = mapper.readValue(entry.getPayload(),
                    Class.forName(entry.getEventType()));
                template.send(entry.getAggregateType().toLowerCase() + "s",
                    entry.getAggregateId(), event).get();
                entry.setProcessedAt(Instant.now());
                outboxRepo.save(entry);
            } catch (Exception e) {
                log.error("Failed to publish outbox entry {}", entry.getId(), e);
                // не отмечаем как processed → повторим следующий цикл
            }
        }
    }
}
```

Периодический fetch batch of unprocessed. Publish each. Mark processed. On failure — leave marked unprocessed, retry next iteration.

Caveat — дубли возможны. Если publish прошёл к broker но processed_at не обновился (crash between operations) — следующий цикл опубликует снова. At-least-once delivery. Consumer must быть идемпотентен.

Batching через LIMIT 100. Prevents overwhelming publisher при massive outbox. Multiple iterations process backlog gradually.

Ordering. Ordered by created_at plus aggregate_id как partition key. Same aggregate events preserve order через partition. Different aggregates могут reorder что typically OK — different business entities independent.

Retention. Outbox growing indefinitely не acceptable. Periodic cleanup:
```sql
DELETE FROM outbox WHERE processed_at < now() - INTERVAL '7 days';
```

Или partitioning по created_at, drop старых partition. More efficient для massive volumes.

## Debezium plus CDC alternative

Вместо polling — Change Data Capture (CDC) через Debezium. Reads database WAL directly, publishes changes to Kafka. Real-time propagation.

Как работает. Debezium reads WAL PostgreSQL через logical replication. Every change (INSERT/UPDATE/DELETE) becomes event. Streamed to Kafka topics. Consumers read normally.

Setup requires. wal_level=logical в PostgreSQL configuration. Replication slot created для Debezium. Debezium connector configured в Kafka Connect. Кто-то needs Kafka Connect infrastructure.

Outbox с Debezium через specific SMT (Single Message Transform). Пишешь в outbox таблицу — Debezium automatically публикует. Готовый Outbox Event Router — reads outbox rows, publishes к topics based on aggregate_type.

Плюсы. Никакого polling — real-time propagation. Никакого custom publisher — Debezium handles. Ordering через partition key. Легко масштабировать.

Минусы. Требует Kafka Connect plus Debezium infrastructure. Настройка logical replication PostgreSQL. Ещё одна infra piece to manage.

Прямой CDC без outbox. Debezium читать напрямую бизнес-таблицы Orders, Payments — publishes changes.

Против direct CDC. Broker получает implementation details таблиц — schema, все fields exposed. Coupling — изменение таблицы ломает consumers. Business semantic не clean — technical INSERT/UPDATE events instead of business events (OrderCreated, OrderCancelled).

Outbox абстрагирует таблицы. Publishes business events (OrderCreated) not technical events (row inserted). Cleaner contract с consumers. Recommended approach когда Debezium chosen.

## Inbox pattern для consumer deduplication

Обратная сторона. Как consumer дедуплицирует retries?

Producer публикует at-least-once. Consumer может получить сообщение несколько раз. Причины — retry после failure, rebalance triggering re-delivery, restart без committed offset.

If consumer. Пишет в БД — повторное processing = duplicate records. Вызывает external API — repeat call. Sends email — sends twice.

Inbox pattern. Consumer maintains таблицу processed message IDs:
```sql
CREATE TABLE inbox (
    message_id TEXT PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Consumer logic:
```java
@Transactional
public void handle(OrderCreatedEvent event) {
    if (inboxRepo.existsById(event.getMessageId())) {
        log.debug("Skip duplicate {}", event.getMessageId());
        return;
    }
    processOrder(event);
    inboxRepo.save(new InboxEntry(event.getMessageId()));
    // на commit — и бизнес-действия, и inbox запись атомарно
}
```

Проверка plus обработка plus запись в inbox — в одной transaction. Atomicity guaranteed by database. If crash between processing и inbox save — на retry checked против inbox, если не там — reprocess. Если saved — skip.

Уникальный messageId requirement. Producer должен генерировать unique ID на каждое сообщение (typically UUID). При retry — тот же messageId. Ensures deduplication работает.

В Kafka producer генерирует автоматически через record metadata. В RabbitMQ — устанавливать вручную через MessageProperties.setMessageId. Explicit control needed.

Retention inbox. Так же как outbox — periodic cleanup:
```sql
DELETE FROM inbox WHERE processed_at < now() - INTERVAL '30 days';
```

Ретеншн должен быть больше чем максимально possible time delay duplicate delivery. If Kafka retains 7 days, inbox needs 7+ days. Otherwise duplicate arrival after inbox cleanup — reprocessed as new.

## Outbox plus Inbox equal effectively exactly-once

Combination обеспечивает end-to-end reliability:
```
[Service A]                                [Service B]
    │                                          │
    │  @Transactional                          │
    │  save Order + save OutboxEntry           │
    │  commit                                  │
    │                                          │
    │  ────────────────────────────►           │
    │  OutboxPublisher polls                   │
    │  publishes to Kafka                      │
    │                                          │
    │              Kafka                       │
    │              │                            │
    │              ▼                            │
    │                                          │
    │  ────────────────────────────►           │
    │                                        [Consumer]
    │                                          │
    │                                          │  @Transactional
    │                                          │  check inbox
    │                                          │  if not exists:
    │                                          │      process
    │                                          │      save inbox entry
    │                                          │  commit
```

Result. Producer side atomic — БД plus publish (через outbox). Broker at-least-once — может дублировать. Consumer side inbox фильтрует дубли. End-to-end — effectively exactly-once.

Not true exactly-once в theoretical CS sense (impossible в distributed systems). But effectively — business observable semantics equivalent к exactly-once. Duplicates detected, ignored, business state consistent.

## Broader idempotency approaches

Inbox один specific approach. Другие также работают.

Идемпотентный UPDATE через conditional clause:
```sql
UPDATE orders SET status='PROCESSED' WHERE id=? AND status='NEW';
-- rows affected = 0 → уже обработано
-- rows affected = 1 → успех
```

Работает если операция это state machine transition. Check current status перед update. Only affects if in expected precondition state. Repeat calls no-op after first success.

UPSERT для inserts:
```sql
INSERT INTO users (id, email) VALUES (?, ?)
ON CONFLICT (id) DO NOTHING;
```

Or DO UPDATE SET email = EXCLUDED.email для upsert with update semantics. Second call с same ID не duplicates.

Optimistic version через version columns:
```sql
UPDATE orders SET status='X', version=version+1 
WHERE id=? AND version=?;
```

Не сработает если версия уже изменилась — кто-то опередил. Concurrent modification detection.

Idempotency-Key header для HTTP APIs. Client sends Idempotency-Key: unique-uuid header. Server:
- Первый вызов с этим key → обрабатывает, сохраняет response.
- Повторный вызов с тем же key → возвращает saved response.

Используется Stripe API. Полезно для payments, POST operations. Client responsible для generating и tracking keys. Server maintains key-to-response mapping.

## Полная реализация в Spring

Entity plus repository:
```java
@Entity
@Table(name = "outbox")
public class OutboxEntry {
    @Id @GeneratedValue Long id;
    String aggregateType;
    String aggregateId;
    String eventType;
    @Column(columnDefinition = "jsonb") String payload;
    Instant createdAt = Instant.now();
    Instant processedAt;
    // getters/setters
}

interface OutboxRepository extends JpaRepository<OutboxEntry, Long> {
    List<OutboxEntry> findTop100ByProcessedAtIsNullOrderByCreatedAt();
}
```

Service использует:
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

Publisher scheduled job:
```java
@Component
@Slf4j
class OutboxPublisher {

    @Autowired OutboxRepository outboxRepo;
    @Autowired KafkaTemplate<String, String> template;

    @Scheduled(fixedDelay = 500)
    @Transactional
    public void publish() {
        List<OutboxEntry> pending = outboxRepo.findTop100ByProcessedAtIsNullOrderByCreatedAt();

        for (OutboxEntry entry : pending) {
            try {
                template.send(
                    entry.getAggregateType().toLowerCase() + "s",   // topic
                    entry.getAggregateId(),                          // key
                    entry.getPayload()                                // JSON
                ).get(5, TimeUnit.SECONDS);

                entry.setProcessedAt(Instant.now());
                outboxRepo.save(entry);
            } catch (Exception e) {
                log.error("Failed to publish outbox {}", entry.getId(), e);
                // не отмечаем — следующий цикл повторит
            }
        }
    }
}
```

Send с timeout — avoids hanging publisher indefinitely если Kafka slow. Get result triggers actual send и waits synchronously. Failed sends leave records unprocessed для retry.

## ShedLock для multi-instance publisher

Если приложение в K8s с 3 репликами — все 3 будут polling outbox → задвоение publications.

ShedLock — distributed lock на scheduled jobs:
```gradle
implementation 'net.javacrumbs.shedlock:shedlock-spring:5.10.0'
implementation 'net.javacrumbs.shedlock:shedlock-provider-jdbc-template:5.10.0'
```

```java
@Scheduled(fixedDelay = 500)
@SchedulerLock(name = "OutboxPublisher_publish", 
               lockAtLeastFor = "PT100MS", 
               lockAtMostFor = "PT30S")
public void publish() { ... }
```

Только одна реплика будет выполнять в moment. Lock stored in shared database — все instances see same lock table.

Реальный кейс из КНП memory knp-fno21-shedlock-stale-image-dup-regnum. Без ShedLock scheduled job лупился параллельно на всех репликах → duplicate INSERTs с same регистрационными номерами. Deploy с ShedLock решил проблему.

## Realistic issues

Outbox таблица растёт быстро. Periodic DELETE старых. Партиционирование по created_at (drop old partitions instantly). Мониторить размер.

Publisher lag. Если publisher медленный (many sends) → outbox копится → latency событий растёт. Increase scheduled frequency. Batch (multiple sends per iteration). Или Debezium вместо polling для higher throughput.

Ordering. Aggregate_id как partition key preserves ordering per business entity. Same order всё events в одной partition. Different orders могут reorder что usually OK — different aggregates independent.

Dead-letter. Если событие сериализовалось некорректно или consumer не может обработать. Outbox — пометить error_count, после N попыток — в DLT. Inbox consumer — DLT через error handler.

Schema evolution. События сохраняются в outbox. При изменении schema — старые события могут быть невалидны при десериализации.

Правила. Backward compatible изменения. Versioning в event payload. Consumer'ы handle multiple versions.

## Alternatives Outbox

Chained saga без outbox. Просто transaction plus publish без outbox:
```java
@Transactional
public void createOrder(...) {
    orderRepo.save(o);
    kafka.send(...);
}
```

Проблема. Если send выполнится, а tx откатится — фиктивное событие. Или наоборот. Даже с @TransactionalEventListener AFTER_COMMIT — если процесс упадёт между commit и send — потеря.

Outbox надёжнее в reliability perspective.

Kafka transactions. Kafka plus JPA в одной transaction:
```java
@Transactional("kafkaTransactionManager")
public void createOrder(...) {
    orderRepo.save(o);
    kafka.send(...);
}
```

Проблема. kafkaTransactionManager управляет только Kafka. JPA — отдельно. Нельзя атомарно commit оба (без XA). ChainedTransactionManager (deprecated) best-effort, не гарантия.

Event Sourcing. Не хранишь текущее состояние, только события. Events — source of truth. Публикация automatically при append. Different architecture entirely. Complex — worthwhile только specific scenarios.

## Best practices

Outbox для guarantee «БД commit → event опубликуется». Fundamental reliability pattern для microservices publishing events.

Inbox / idempotency ключ для consumer. Complement to outbox — прevents duplicate processing on consumer side.

UUID as messageId. Universally unique identifiers ensure no collisions across sessions.

Retention outbox / inbox таблиц. Prevent unbounded growth. Weekly или monthly cleanup schedules.

ShedLock для publisher на multiple instances. Prevent parallel processing of same outbox records.

Debezium для real-time / high volume (вместо polling). Real-time propagation. But requires Kafka Connect infrastructure.

Aggregate ID как partition key — сохранение порядка per business entity.

Мониторинг. Unprocessed outbox count — alert if grows sustainably. Inbox size — retention working. Publisher lag — freshness metric.

## Итоги

Outbox pattern решает атомарность «БД save plus event publish» через shifting problem. Event stored в БД в same transaction as business data. Separate publisher reads outbox — publishes к broker — marks processed.

Гарантия at-least-once (не exactly-once). Publisher может послать дубли если crash после send но перед update processed_at.

Debezium/CDC — real-time alternative к polling. Reads WAL PostgreSQL. Requires Kafka Connect. Recommended для high volume.

Inbox pattern для consumer deduplication. Table processed message IDs. Check plus process plus save в one transaction. Atomicity guarantees no duplicate processing.

Outbox plus Inbox equal effectively exactly-once. Not theoretically pure but business observably equivalent.

Идемпотентность обеспечивается через. Уникальный messageId plus inbox. Conditional UPDATE (state machine transitions). UPSERT для inserts. Optimistic version columns. Idempotency-Key HTTP header.

ShedLock для multi-instance publisher. Prevents parallel job execution через distributed lock. Реальный кейс knp-fno21-shedlock-stale-image-dup-regnum demonstrates необходимость.

Retention необходима — outbox/inbox расти вечно не acceptable. Delete старых. Партиционирование для massive volumes.

Publisher lag monitoring critical. Growing lag indicates capacity problem — increase frequency или switch к Debezium.

Ordering preserved through aggregate_id partition key.

Alternatives — chained saga (unreliable), Kafka transactions с JPA (не работает без XA), Event Sourcing (complex, different architecture).

Дальше — микросервисов resilience patterns comprehensively — Circuit Breaker, Retry, Bulkhead, Rate Limiting, Fallback.
