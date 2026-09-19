# 51. Outbox + Inbox + Idempotency

Как атомарно сохранить в БД и опубликовать событие. Как дедуплицировать на consumer'е.

---

## 1. Задача

Пример: `OrderService.createOrder` должен:
1. **Сохранить** Order в PG.
2. **Опубликовать** `OrderCreated` в Kafka/RabbitMQ.

Что если между шагами упало?

**Вариант А — сначала БД, потом broker**:
```java
@Transactional
public void createOrder(OrderRequest req) {
    Order o = new Order(req);
    orderRepo.save(o);           // ← DB commit
    // ↓ CRASH здесь
    kafka.send("orders", o);     // ← НЕ выполнено!
}
```
Заказ сохранён, событие не опубликовано → **downstream не узнает** → inventory не забронирует → **lost update**.

**Вариант Б — сначала broker, потом БД**:
```java
public void createOrder(OrderRequest req) {
    kafka.send("orders", ...);   // опубликовано
    // ↓ CRASH здесь
    orderRepo.save(o);            // не сохранено
}
```
Событие есть, заказа в БД нет → **inventory забронирует под несуществующий order**.

**XA/2PC** решает, но неприемлемо (см. `50-saga-pattern.md`).

**Решение**: **Outbox pattern**.

---

## 2. Outbox pattern

### 2.1 Идея

Сохранить и **бизнес-данные**, и **описание события** в **одной DB-транзакции**. Отдельный процесс потом читает outbox → публикует → удаляет.

Гарантия: если DB commit прошёл → outbox запись есть → рано или поздно опубликуется.

### 2.2 Структура таблицы

```sql
CREATE TABLE outbox (
    id BIGSERIAL PRIMARY KEY,
    aggregate_type TEXT NOT NULL,          -- 'Order', 'Payment'
    aggregate_id TEXT NOT NULL,             -- '42'
    event_type TEXT NOT NULL,               -- 'OrderCreated'
    payload JSONB NOT NULL,                 -- {...}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ                -- NULL если ещё не опубликовано
);

CREATE INDEX idx_outbox_unprocessed ON outbox(created_at) WHERE processed_at IS NULL;
```

### 2.3 Producer

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

Всё в одной tx. Если commit failed — оба откатились.

### 2.4 Publisher (polling)

Периодически читает необработанные из outbox → публикует:

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
                Object event = mapper.readValue(entry.getPayload(), Class.forName(entry.getEventType()));
                template.send(entry.getAggregateType().toLowerCase() + "s",
                    entry.getAggregateId(), event).get();
                entry.setProcessedAt(Instant.now());
                outboxRepo.save(entry);
            } catch (Exception e) {
                log.error("Failed to publish outbox entry {}", entry.getId(), e);
                // не отмечаем как processed → повторим
            }
        }
    }
}
```

**Кавет**: **дубли возможны**. Если publish прошёл, но `processed_at` не обновился (crash) → следующий цикл опубликует снова.

Отсюда — **at-least-once delivery**. Consumer должен быть идемпотентен.

### 2.5 Retention

Outbox растёт. Периодически удалять старые processed:
```sql
DELETE FROM outbox WHERE processed_at < now() - INTERVAL '7 days';
```

Или партиционирование по created_at, drop старых partition.

---

## 3. Debezium / CDC вариант

Вместо polling — **Change Data Capture** через **Debezium**.

### 3.1 Как работает

Debezium читает **WAL PostgreSQL** через logical replication → публикует изменения в Kafka.

Настройка: `wal_level=logical` + replication slot.

### 3.2 Outbox с Debezium

Пишешь в outbox таблицу — Debezium автоматически публикует.

Готовый **Outbox Event Router** (Kafka Connect SMT) — читает outbox таблицу, публикует в правильный topic (по aggregate_type).

Плюсы:
- **Никакого polling** — real-time.
- **Никакой custom publisher** — Debezium сам делает.
- **Ordering** через partition key.
- Легко масштабировать.

Минусы:
- Требует Kafka Connect + Debezium.
- Настройка logical replication PG.
- Ещё одна infra piece.

### 3.3 Прямой CDC (без outbox)

Можно Debezium читать напрямую бизнес-таблицы (Orders, Payments) → публиковать изменения.

**Против**: broker получает **implementation details** таблиц (schema, все поля). Coupling. Изменение таблицы → ломает consumers.

Outbox — **бизнес-события**. Абстрагирует таблицы.

---

## 4. Inbox pattern

Обратная сторона: как consumer **дедуплицирует**.

### 4.1 Проблема

Producer публикует at-least-once → consumer может получить сообщение несколько раз (retry, rebalance, restart).

Если consumer:
- Пишет в БД → повторное обработка = дубликат записи.
- Вызывает external API → повторный вызов.
- Отправляет email → отправит второй раз.

### 4.2 Решение — Inbox

Consumer держит таблицу **inbox** обработанных сообщений (message_id).

```sql
CREATE TABLE inbox (
    message_id TEXT PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Логика consumer:
```java
@Transactional
public void handle(OrderCreatedEvent event) {
    if (inboxRepo.existsById(event.getMessageId())) {
        log.debug("Skip duplicate {}", event.getMessageId());
        return;
    }
    processOrder(event);
    inboxRepo.save(new InboxEntry(event.getMessageId()));
    // на commit → и бизнес-действия, и inbox запись атомарно
}
```

Ключевое: **проверка + обработка + запись в inbox — в одной tx**.

### 4.3 Уникальный messageId

Producer должен генерировать **уникальный** ID на каждое сообщение (обычно UUID). При retry — **тот же** messageId.

В Kafka producer генерирует автоматически. В RabbitMQ — устанавливать вручную через `MessageProperties.setMessageId`.

### 4.4 Retention inbox

Так же как outbox — периодически удалять старые записи:
```sql
DELETE FROM inbox WHERE processed_at < now() - INTERVAL '30 days';
```

Ретеншн должен быть больше чем максимально возможное время задержки duplicate delivery.

---

## 5. Outbox + Inbox = exactly-once processing

Комбинация двух:

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

Результат:
- **Producer side** — atomic БД + publish (through outbox).
- **Broker** — at-least-once (может задваивать).
- **Consumer side** — inbox фильтрует дубли.
- **End-to-end** — **effectively exactly-once**.

---

## 6. Idempotency

Более общий концепт. Способы:

### 6.1 Уникальный ID + inbox таблица

Классика (§4).

### 6.2 Идемпотентный UPDATE

```sql
UPDATE orders SET status='PROCESSED' WHERE id=? AND status='NEW';
-- rows affected = 0 → уже обработано; = 1 → успех
```

Работает если операция — переход между состояниями (state machine).

### 6.3 UPSERT

```sql
INSERT INTO users (id, email) VALUES (?, ?)
ON CONFLICT (id) DO NOTHING;
-- или DO UPDATE SET email = EXCLUDED.email
```

Не задваивает.

### 6.4 Optimistic version

```sql
UPDATE orders SET status='X', version=version+1 WHERE id=? AND version=?;
```

Не сработает если версия уже изменилась (кто-то опередил).

### 6.5 Idempotency-Key header (HTTP)

Клиент шлёт `Idempotency-Key: unique-uuid` header. Сервер:
- Первый вызов с этим key → обрабатывает, сохраняет response.
- Повторный вызов с тем же key → возвращает сохранённый response.

Используется Stripe API. Полезно для платежей / POST-операций.

---

## 7. Реализация Outbox в Spring — полный пример

### 7.1 Entity

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

### 7.2 Service использует

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

### 7.3 Publisher

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
                    entry.getAggregateId(),                          // key = order id
                    entry.getPayload()                                // JSON
                ).get(5, TimeUnit.SECONDS);

                entry.setProcessedAt(Instant.now());
                outboxRepo.save(entry);
            } catch (Exception e) {
                log.error("Failed to publish outbox {}", entry.getId(), e);
                // не отмечаем → следующий цикл повторит
            }
        }
    }
}
```

### 7.4 ShedLock для multiple instances

Если приложение в K8s с 3 репликами — все 3 будут polling outbox → задвоение публикаций.

**ShedLock** — distributed lock на scheduled jobs:

```gradle
implementation 'net.javacrumbs.shedlock:shedlock-spring:5.10.0'
implementation 'net.javacrumbs.shedlock:shedlock-provider-jdbc-template:5.10.0'
```

```java
@Scheduled(fixedDelay = 500)
@SchedulerLock(name = "OutboxPublisher_publish", lockAtLeastFor = "PT100MS", lockAtMostFor = "PT30S")
public void publish() { ... }
```

Только одна реплика будет выполнять в моменте.

Реальный кейс из ИСНА memory `knp-fno21-shedlock-stale-image-dup-regnum`: без ShedLock scheduled job лупился параллельно → duplicate INSERTs. Fix — deploy с ShedLock.

---

## 8. Realistic issues

### 8.1 Outbox таблица растёт быстро

- Периодические DELETE старых.
- Партиционирование по created_at.
- Мониторить размер.

### 8.2 Publisher lag

Если publisher медленный (много sends) → outbox копится → latency событий растёт.

- Увеличить `fixedDelay` (чаще).
- Batch (несколько events за один цикл).
- Debezium вместо polling.

### 8.3 Ordering

Одинаковый partition key (aggregate_id) → все события одного order идут в одну partition → сохраняется ordering.

Разные aggregate_id могут переупорядочиться на consumer side (если consumer group имеет несколько инстансов).

### 8.4 Dead-letter

Если событие сериализовалось некорректно / consumer не может обработать → куда?

- **Outbox**: пометить `error_count`, после N — в DLT.
- **Inbox** consumer: **DLT** через error handler.

### 8.5 Schema evolution

События сохраняются в outbox. При изменении schema — старые события могут быть невалидны.

Правила:
- **Backward compatible** изменения.
- **Versioning** в event payload.
- Consumer'ы handle multiple versions.

---

## 9. Альтернативы Outbox

### 9.1 Chained saga без outbox

Просто transaction + publish без outbox:
```java
@Transactional
public void createOrder(...) {
    orderRepo.save(o);
    kafka.send(...);
}
```

Проблема — если send выполнится, а tx откатится → **фиктивное событие**. Или наоборот.

Даже если `@TransactionalEventListener(AFTER_COMMIT)` — если процесс упадёт между commit и send → потеря.

**Outbox надёжнее**.

### 9.2 Kafka transactions

Kafka + JPA в одной transaction:
```java
@Transactional("kafkaTransactionManager")
public void createOrder(...) {
    orderRepo.save(o);
    kafka.send(...);
}
```

Проблема: `kafkaTransactionManager` управляет только Kafka. JPA — отдельно. Нельзя атомарно commit оба (без XA).

Есть **ChainedTransactionManager** (deprecated) — best-effort, не гарантия.

### 9.3 Event Sourcing

Не хранишь текущее состояние, только события. Events — source of truth. Публикация автоматом при append.

Другая архитектура целиком.

---

## 10. Best practices

1. **Outbox для guarantee «БД commit → event опубликуется»**.
2. **Inbox / idempotency ключ для consumer**.
3. **UUID as messageId**.
4. **Retention** outbox / inbox таблиц.
5. **ShedLock** для publisher на multiple instances.
6. **Debezium** для real-time / high volume (вместо polling).
7. **Aggregate ID как partition key** — сохранение порядка.
8. **Мониторинг** unprocessed outbox count, inbox size.

---

## 11. Собесные вопросы

1. **Что такое Outbox pattern?** — Событие сохраняется в БД в той же tx что бизнес-данные; отдельный publisher читает и публикует.
2. **Зачем Outbox?** — Атомарность «БД commit + publish» без XA.
3. **Гарантии Outbox?** — At-least-once (publisher может послать дубли если crash до update processed_at).
4. **Debezium vs polling?** — Debezium читает WAL PG → Kafka; real-time, без polling; сложнее setup.
5. **Что такое Inbox pattern?** — Consumer держит таблицу processed message IDs; проверяет перед обработкой.
6. **Outbox + Inbox — что даёт?** — Effectively exactly-once processing.
7. **Как обеспечить идемпотентность?** — UUID messageId + inbox / conditional UPDATE / UPSERT.
8. **Что делает ShedLock?** — Distributed lock для scheduled jobs; только одна реплика выполняет.
9. **Что такое Idempotency-Key?** — HTTP header для дедупликации на server side (Stripe).
10. **Retention outbox — зачем и как?** — Растёт быстро; DELETE старых или партиционирование.
11. **Ordering в Outbox?** — Aggregate ID как partition key → все события одного aggregate в одну partition.
12. **CDC vs Outbox — разница?** — CDC читает бизнес-таблицы напрямую (schema coupling); Outbox читает специальную event-таблицу (абстракция).
13. **Почему не Kafka transactions вместо Outbox?** — Kafka + JPA нельзя атомарно commit без XA.
14. **Что если publisher crash после kafka.send но до update processed_at?** — Дубль → consumer должен быть идемпотентен.
15. **Что случится если outbox таблица переполнится?** — Publisher медленный → lag событий; мониторинг unprocessed count критично.

---

## Итог

- **Outbox**: событие + бизнес-данные в одной DB-tx; publisher читает + отправляет.
- **Гарантия at-least-once** (не exactly-once).
- **Debezium/CDC** — real-time альтернатива polling'у.
- **Inbox**: consumer держит processed message IDs; дедупликация.
- **Outbox + Inbox = effectively exactly-once**.
- **UUID messageId** обязателен.
- **ShedLock** для publisher в multi-instance.
- **Retention** обоих таблиц.

Следующий — `52-microservices-resilience.md`.
