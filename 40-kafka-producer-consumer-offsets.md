# 40. Kafka: producer, consumer, offsets, consumer groups

Как реально работать с Kafka. Гарантии доставки, offset management, rebalancing.

---

## 1. Producer детали

### 1.1 Основной цикл

```java
Producer<String, Order> producer = new KafkaProducer<>(props);

ProducerRecord<String, Order> record = new ProducerRecord<>(
    "orders",                    // topic
    order.getCustomerId(),        // key (для partitioning)
    order                         // value
);

Future<RecordMetadata> future = producer.send(record);
RecordMetadata meta = future.get();   // sync — блокирует
// meta.partition(), meta.offset(), meta.timestamp()

producer.close();
```

### 1.2 Async с callback

```java
producer.send(record, (meta, exception) -> {
    if (exception != null) {
        log.error("Send failed", exception);
    } else {
        log.info("Sent to partition={} offset={}", meta.partition(), meta.offset());
    }
});
```

Не блокирует. Callback вызывается в I/O-потоке producer'а (не спать долго!).

### 1.3 Основные настройки

```properties
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer

acks=all
enable.idempotence=true
max.in.flight.requests.per.connection=5
retries=2147483647          # max
delivery.timeout.ms=120000

# batch tuning
linger.ms=10
batch.size=32768
compression.type=zstd

# buffer
buffer.memory=33554432       # 32 MB
```

### 1.4 Partitioning стратегии

**По ключу** (default):
```java
new ProducerRecord<>("orders", "customer-42", order);
// hash("customer-42") % partitions → та же partition для этого customer
```

**Round-robin** (без ключа):
```java
new ProducerRecord<>("orders", null, order);
// каждое сообщение → следующая partition
```

**Sticky** (Kafka 2.4+, default для без-ключа): batch отправляется на одну partition для эффективности.

**Custom**:
```java
public class MyPartitioner implements Partitioner {
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        // custom logic
    }
}
```

Регистрация: `partitioner.class=com.example.MyPartitioner`.

### 1.5 Delivery guarantees

Комбинация настроек:

**At-most-once**:
```
acks=0 (или 1) + retries=0
```

Может потерять, не задваивает.

**At-least-once**:
```
acks=all + retries>0
```

Не потеряет, может задвоить (при retry без idempotence).

**Exactly-once** (в одном producer):
```
acks=all + enable.idempotence=true
```

Не потеряет, не задвоит (в рамках одной partition).

**Exactly-once transactional** (см. `42-kafka-prod.md`):
```
+ transactional.id + producer.initTransactions()
```

Для многих partitions атомарно.

---

## 2. Consumer детали

### 2.1 Основной цикл

```java
Consumer<String, Order> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("orders"));

try {
    while (running) {
        ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(500));

        for (ConsumerRecord<String, Order> record : records) {
            try {
                process(record.value());
            } catch (Exception e) {
                log.error("Failed to process offset {}", record.offset(), e);
                // strategy: retry, skip, DLT
            }
        }

        consumer.commitSync();
    }
} finally {
    consumer.close();
}
```

### 2.2 poll модель

Kafka — **pull-based**. Consumer сам запрашивает данные. В отличие от Rabbit (push).

`poll(timeout)`:
- Забирает пачку сообщений (до `max.poll.records`, default 500).
- Ждёт до `timeout` если нет данных.
- **Также** используется для heartbeat к coordinator.

Если между poll'ами прошло больше `max.poll.interval.ms` (default 5 мин) → coordinator считает consumer мёртвым → rebalance.

**Правило**: обрабатывать batch быстро; для долгих операций — увеличить `max.poll.interval.ms` или обработка в отдельных потоках.

### 2.3 Ключевые настройки

```properties
bootstrap.servers=broker:9092
group.id=order-processor
key.deserializer=...
value.deserializer=...

# offset поведение при первом старте
auto.offset.reset=earliest      # earliest / latest / none

# commit
enable.auto.commit=false         # правильно — false, коммитим сами
auto.commit.interval.ms=5000     # если true (плохо)

# poll
max.poll.records=500
max.poll.interval.ms=300000      # 5 min max между poll
session.timeout.ms=10000
heartbeat.interval.ms=3000

# fetch
fetch.min.bytes=1
fetch.max.wait.ms=500
```

### 2.4 auto.offset.reset

Что делать при первом запуске (нет сохранённого offset):
- **earliest** — с начала topic.
- **latest** — только новые сообщения (пропустить историю).
- **none** — exception.

Для новых consumer'ов на существующем topic — обычно `latest`.

---

## 3. Consumer group

### 3.1 Что это

**Consumer group** — набор consumer'ов, делящих обработку topic.

```
Topic "orders" (partitions 0, 1, 2, 3)
        │
        │  consumers subscribe to topic
        ▼
┌───── Group "order-processor" ─────┐
│   Consumer A → partitions 0, 1    │
│   Consumer B → partitions 2, 3    │
└───────────────────────────────────┘
```

Правило: **каждая partition назначена одному consumer в группе**.

- Если consumer < partitions → некоторые consumer'ы обслуживают несколько.
- Если consumer > partitions → часть consumer'ов простаивает.

Отсюда: **partition = единица параллелизма**. Хочешь больше параллелизма → больше partitions.

### 3.2 Разные group

Если разные group подписаны на тот же topic → каждая получает **свою копию** сообщений.

```
Topic "orders"
    │
    ├─ Group "order-processor" (consumers A, B) — получают все сообщения
    │
    ├─ Group "audit-logger" (consumers C, D) — тоже получают все
    │
    └─ Group "analytics" (consumers E) — все сообщения
```

Это как **fanout** в Rabbit.

### 3.3 Rebalancing

**Rebalance** — перераспределение partitions между consumer'ами группы. Триггеры:
- Новый consumer join.
- Consumer покинул (нормально или crash).
- Изменение partition count в topic.

Во время rebalance — **вся группа не потребляет** (freeze). Может занять секунды.

Проблема **rebalancing storm** — частые rebalance при нестабильных consumer'ах. Тюнить heartbeat / session.timeout.

### 3.4 Стратегии assignor

Как partitions распределяются:
- **RangeAssignor** (default) — топики по алфавиту, partitions по диапазонам.
- **RoundRobinAssignor** — по кругу, все topics вместе.
- **StickyAssignor** — минимизирует reassignment при rebalance.
- **CooperativeStickyAssignor** — incremental rebalance (Kafka 2.4+), не freeze.

Правило для новых consumer'ов: **CooperativeStickyAssignor** — меньше freeze.

---

## 4. Offset management

### 4.1 __consumer_offsets

Специальный topic Kafka. Consumer commit'ит свои offsets туда.

Формат: `(group.id, topic, partition) → offset`.

### 4.2 Auto commit

`enable.auto.commit=true` (default) — Kafka автоматически commit'ит каждые `auto.commit.interval.ms` (default 5 сек).

**Проблема**: commit происходит **до** обработки → возможна потеря при crash между commit и обработкой.

Или наоборот — обработал → до auto-commit крашнулся → снова обработает (duplicate).

**Правило**: **отключить auto-commit** для важных сообщений.

### 4.3 Manual commit

```properties
enable.auto.commit=false
```

Явно:
```java
consumer.commitSync();     // блокирующий, надёжный
consumer.commitAsync();    // не блокирует, callback
```

Обычная схема:
```java
while (running) {
    ConsumerRecords<...> records = consumer.poll(...);
    for (ConsumerRecord<...> record : records) {
        process(record);
    }
    consumer.commitSync();      // после всей пачки
}
```

**At-least-once**: commit **после** обработки. Если crash до commit — сообщения обработаются снова (idempotency важна).

### 4.4 Ручной offset

```java
// commit конкретных offsets
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
offsets.put(new TopicPartition("orders", 0), new OffsetAndMetadata(42L));
consumer.commitSync(offsets);

// seek
consumer.seek(new TopicPartition("orders", 0), 100L);   // прочитать с offset 100
consumer.seekToBeginning(...);
consumer.seekToEnd(...);
```

Использование seek: replay событий, skip poisoned messages.

### 4.5 commitSync vs commitAsync

- **commitSync** — блокирует до подтверждения. Надёжно, но медленно.
- **commitAsync** — не блокирует, callback. Быстро, но при crash до callback — не commit'нулся.

Правило: **commitSync** после каждого batch + **commitAsync** между.

---

## 5. Delivery semantics

### 5.1 At-most-once

Commit **до** обработки:
```java
consumer.commitSync();
process(record);   // если упало здесь → сообщение потеряно
```

Использование: метрики (потеря одной ок).

### 5.2 At-least-once (default recommendation)

Commit **после** обработки:
```java
process(record);
consumer.commitSync();
```

Дубли возможны (crash между process и commit).

**Всегда идемпотентный consumer**.

### 5.3 Exactly-once

Комбо:
- **Idempotent producer** (`enable.idempotence=true`).
- **Transactional producer** для multi-partition atomic writes.
- **`isolation.level=read_committed`** на consumer.
- **Обработка + commit** внутри Kafka-транзакции.

Пример stream processing:
```java
producer.beginTransaction();
try {
    for (record : records) {
        Result r = process(record);
        producer.send(new ProducerRecord<>("results", r));
    }
    producer.sendOffsetsToTransaction(offsets, group);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

Не всегда возможно (если writes идут не только в Kafka). Подробно — `42-kafka-prod.md`.

---

## 6. Heartbeat и session

Consumer посылает heartbeat coordinator'у чтобы «я жив».

- `heartbeat.interval.ms=3000` — как часто (default 3 сек).
- `session.timeout.ms=10000` — если нет heartbeat N мс → dead → rebalance (default 10 сек, макс 30 сек до 3.0, 45 сек в 3.0+).

Плюс `max.poll.interval.ms=300000` (5 мин) — если между poll больше — dead.

Разница:
- Heartbeat идёт в отдельном потоке — независимо от обработки.
- max.poll.interval — время обработки batch.

Если обработка долгая:
- Увеличить `max.poll.interval.ms`.
- Уменьшить `max.poll.records`.
- Пауза-возобновление partition (`pause`, `resume`) — chunk-обработка.

---

## 7. Rebalance listener

Callback на rebalance:
```java
consumer.subscribe(List.of("orders"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> parts) {
        // Партиции забирают → commit оставшиеся offsets
        consumer.commitSync();
    }
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> parts) {
        // Новые partitions — можно инициализировать state
    }
});
```

Полезно для graceful commit при shutdown / rebalance.

---

## 8. Дедуп на consumer стороне

Даже с idempotent producer — дубли возможны при retry / rebalance.

Правило: consumer **должен быть идемпотентен**:

**A) Уникальный messageId + processed table**:
```java
if (processedRepo.existsById(record.key())) return;
process(record);
processedRepo.save(new Processed(record.key()));
```

**B) Идемпотентные UPDATE**:
```sql
UPDATE orders SET status='PROCESSED' WHERE id=? AND status='NEW';
-- второй раз ничего не изменит
```

**C) UPSERT**:
```sql
INSERT ... ON CONFLICT DO NOTHING;
```

---

## 9. Многопоточная обработка

По умолчанию Kafka consumer — **однопоточный** (один поток на consumer).

Для параллелизма:
1. **Больше consumers в группе** (до количества partitions).
2. **Внутри одного consumer — thread pool** для обработки:

```java
ExecutorService pool = Executors.newFixedThreadPool(10);

while (running) {
    ConsumerRecords<...> records = consumer.poll(...);
    List<Future<?>> futures = new ArrayList<>();
    for (ConsumerRecord<...> record : records) {
        futures.add(pool.submit(() -> process(record)));
    }
    for (Future<?> f : futures) f.get();       // ждём все
    consumer.commitSync();
}
```

Кавет: теряется порядок обработки внутри partition. Если нужен порядок — не делать multi-threading внутри partition.

Spring Kafka имеет `concurrency` (см. `41-spring-kafka.md`) — по сути это несколько consumer'ов в одном приложении.

---

## 10. Real-world: типовые ошибки

### 10.1 Медленный consumer → rebalance

`max.poll.interval.ms` истёк → coordinator убил → rebalance → снова тот же сценарий. Loop.

Fix: уменьшить `max.poll.records` или увеличить `max.poll.interval.ms`.

### 10.2 Poison pill

Сообщение с невалидным форматом → Deserializer падает → poll бросает exception → бесконечный retry.

Fix:
- `ErrorHandlingDeserializer` (Spring Kafka) — оборачивает + отправляет в DLT.
- Custom code — try/catch десериализации.

### 10.3 Дубли из-за crash между process и commit

Обычная реальность. **Идемпотентность consumer'а** — обязательна.

### 10.4 Offset lag растёт

Consumer не догоняет producer. Причины:
- Медленный consumer.
- Мало consumers в группе.
- Мало partitions (нельзя добавить больше consumers чем partitions).
- Downstream БД/API тормозит.

Мониторить: **consumer lag** = `latest_offset - current_offset` per partition.

### 10.5 Hot partition

Один ключ занимает большую часть трафика → одна partition перегружена, другие пустуют.

Fix: sharding ключа, custom partitioner.

---

## 11. Kafka Streams (кратко)

Библиотека для stream processing поверх Kafka.

```java
KStream<String, Order> orders = builder.stream("orders");
orders
    .filter((k, v) -> v.getAmount() > 100)
    .mapValues(v -> new BigOrder(v))
    .to("big-orders");
```

Exactly-once, stateful (RocksDB), joins, aggregations.

Не в этом файле, но знать что существует.

---

## 12. Собесные вопросы

1. **Как producer выбирает partition?** — hash(key) % partitions; без ключа round-robin / sticky.
2. **Что такое acks?** — Уровень надёжности: 0 (fire), 1 (leader), all (все ISR).
3. **Идемпотентный producer — что даёт?** — Retry не даст дублей в рамках одной partition.
4. **Что такое consumer group?** — Группа consumer'ов, делят partitions одного topic.
5. **Разные consumer groups и один topic?** — Каждая группа получает всю копию (fanout).
6. **Что происходит при rebalance?** — Партиции перераспределяются; вся группа не потребляет.
7. **poll timeout — что делает?** — Как долго ждать при отсутствии данных; также идёт heartbeat.
8. **auto-commit vs manual commit?** — Auto: каждые N мс (риск потери/дублей); manual: явный контроль.
9. **commitSync vs commitAsync?** — Sync блокирует до подтверждения; Async не блокирует, callback.
10. **Delivery semantics?** — At-most-once (commit до), at-least-once (после), exactly-once (idempotent + transactional).
11. **Что такое offset lag?** — Разница latest_offset и current_offset consumer'а; растущий = отстаёт.
12. **auto.offset.reset — когда earliest vs latest?** — При первом запуске: earliest — с начала topic; latest — только новые.
13. **Как обеспечить идемпотентность consumer?** — processed_id table / UPSERT / conditional UPDATE.
14. **max.poll.interval.ms — что?** — Максимум времени между poll'ами; иначе dead → rebalance.
15. **Разница heartbeat.interval.ms и session.timeout.ms?** — Heartbeat как часто; session сколько без heartbeat = dead.

---

## Итог

- **Producer**: acks=all + idempotence + компрессия.
- **Ключ** = partition = порядок.
- **Consumer group** делит partitions.
- **Rebalance** = freeze; минимизировать через Cooperative assignor.
- **Manual commit** после обработки.
- **At-least-once** + **идемпотентность** = стандартный setup.
- **max.poll.interval.ms** > время обработки batch.
- Мониторить **consumer lag**.

Следующий — `41-spring-kafka.md`.
