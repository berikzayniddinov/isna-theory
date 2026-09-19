# 40. Kafka: producer, consumer, offsets, consumer groups

## Producer детально

Основной цикл работы с producer:
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

Sync запись через future.get блокирует поток до получения confirmation от broker. Safe но медленно. Асинхронная запись через callback:
```java
producer.send(record, (meta, exception) -> {
    if (exception != null) {
        log.error("Send failed", exception);
    } else {
        log.info("Sent to partition={} offset={}", meta.partition(), meta.offset());
    }
});
```

Не блокирует producer thread. Callback вызывается в I/O-thread producer'а поэтому не должен блокировать на долго — быстрый callback обязателен.

## Producer настройки

Ключевые properties для production:
```properties
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer

acks=all
enable.idempotence=true
max.in.flight.requests.per.connection=5
retries=2147483647          # max int
delivery.timeout.ms=120000

# batch tuning
linger.ms=10
batch.size=32768
compression.type=zstd

# buffer
buffer.memory=33554432       # 32 MB
```

bootstrap.servers это список brokers для initial connection. Producer discovers full cluster через metadata request к любому из них. Обычно 2-3 addresses указывается для resilience.

acks=all plus enable.idempotence=true plus retries max — стандартный production setup для reliability.

max.in.flight.requests.per.connection=5 разрешает 5 unacked requests в одно время plus preserving ordering (с idempotence).

linger.ms=10 plus batch.size=32768 balance latency и throughput.

compression.type=zstd для network efficiency.

## Partitioning стратегии

По ключу default. Producer вычисляет hash(key) modulo partitions:
```java
new ProducerRecord<>("orders", "customer-42", order);
// hash("customer-42") % partitions → та же partition для этого customer
```

Гарантирует ordering per key. Все events one customer идут в one partition в правильном order.

Round-robin без ключа:
```java
new ProducerRecord<>("orders", null, order);
// каждое сообщение → следующая partition
```

Uniform distribution но no ordering guarantees per anything specific.

Sticky (Kafka 2.4+, default для без-ключа). Batch отправляется на одну partition для batching efficiency plus rotates on batch completion. Combines throughput benefits batching plus reasonable distribution.

Custom partitioner для business-specific logic:
```java
public class MyPartitioner implements Partitioner {
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        // custom logic based on value or business rules
    }
}
```

Регистрация через partitioner.class=com.example.MyPartitioner.

## Delivery guarantees сочетания

Combinations settings определяют guarantees.

At-most-once. acks=0 или 1 plus retries=0. Может потерять messages, не задваивает. Использование для metrics где потеря одной метрики не критична.

At-least-once. acks=all plus retries>0. Не потеряет message, может задвоить (при retry без idempotence). Standard for reliable data. Idempotency consumer критична.

Exactly-once в one producer. acks=all plus enable.idempotence=true. Не потеряет, не задваивает в рамках одной partition. Ideal для single-partition scenarios.

Exactly-once transactional требует transactional.id plus producer.initTransactions plus wrap operations in beginTransaction/commitTransaction. Для multi-partition atomic writes plus consumer offset commits в one transaction.

## Consumer детально

Основной цикл consumer:
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

poll model. Kafka pull-based (в отличие от RabbitMQ push). Consumer сам запрашивает данные через poll(timeout).

poll(timeout) забирает batch messages up to max.poll.records (default 500). Ждёт до timeout если нет данных. Также используется для heartbeat к coordinator — важная function.

Если между poll'ами прошло больше max.poll.interval.ms (default 5 минут) — coordinator считает consumer мёртвым и triggers rebalance. Отсюда правило — обрабатывать batch быстро, для долгих операций либо увеличить max.poll.interval.ms либо offload обработку в отдельные threads.

## Consumer ключевые настройки

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

auto.offset.reset определяет behavior при первом запуске когда нет saved offset.

earliest — с начала topic. Consumer reads все historical messages.

latest — только новые сообщения. Пропускает history.

none — throw exception если no offset. Для strict scenarios где либо offset есть либо fail.

Для новых consumers на existing topic обычно latest — не хотим reprocess historical data.

## Consumer group

Consumer group это набор consumers делящих обработку topic. Ключевая абстракция для parallelism.

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

Правило — каждая partition назначена ровно одному consumer в group. Если consumers меньше partitions некоторые consumers обслуживают multiple partitions. Если consumers больше partitions лишние простаивают.

Отсюда важное следствие. Partition это единица parallelism. Хочешь больше parallelism — создавай больше partitions. Меньше partitions чем ожидаемых consumers — waste resources.

Разные groups подписанные на same topic получают каждая свою копию сообщений:
```
Topic "orders"
    │
    ├─ Group "order-processor" (consumers A, B) — получают все сообщения
    │
    ├─ Group "audit-logger" (consumers C, D) — тоже получают все
    │
    └─ Group "analytics" (consumers E) — все сообщения
```

Это equivalent fanout pattern в RabbitMQ. Каждая group processes independently — не влияют друг на друга.

## Rebalancing

Rebalance это перераспределение partitions между consumers группы. Triggered в нескольких сценариях. Новый consumer join группу. Consumer покинул (normal shutdown или crash). Изменение partition count в topic.

Во время rebalance вся группа не потребляет — freeze. Может занять секунды. Всё group operations paused до completion.

Rebalancing storm это проблема когда consumers нестабильны и rebalances происходят часто. Fixing через настройку heartbeat.interval.ms и session.timeout.ms.

Стратегии assignor определяют как partitions распределяются между consumers. RangeAssignor default — topics в alphabetical order, partitions в ranges. RoundRobinAssignor — round-robin по всем topics вместе. StickyAssignor минимизирует reassignment при rebalance — preserves previous assignments где возможно. CooperativeStickyAssignor (Kafka 2.4+) — incremental rebalance, не полный freeze.

Правило для новых consumers — CooperativeStickyAssignor. Значительно меньше disruption во время rebalance.

## Offset management

__consumer_offsets это специальный Kafka topic где consumers commit свои offsets. Format `(group.id, topic, partition) → offset`. Consumer при starting reads свой offset и продолжает с этой position.

Auto commit через enable.auto.commit=true (default). Kafka автоматически commits каждые auto.commit.interval.ms (default 5 секунд).

Проблема auto commit. Commit происходит независимо от обработки. Возможен crash между commit и обработкой — потеря message. Или обратный — обработал но crashed до next auto commit — reprocess при restart (duplicate).

Правило для важных сообщений — отключить auto-commit. enable.auto.commit=false.

Manual commit явно:
```java
consumer.commitSync();     // блокирующий, надёжный
consumer.commitAsync();    // не блокирует, callback
```

Стандартная схема at-least-once:
```java
while (running) {
    ConsumerRecords<...> records = consumer.poll(...);
    for (ConsumerRecord<...> record : records) {
        process(record);
    }
    consumer.commitSync();      // после всей пачки
}
```

Обработать сначала, только потом commit. Если crash до commit — messages обрабатываются снова при restart (idempotency важна для safety).

Ручной offset control:
```java
// commit конкретных offsets
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
offsets.put(new TopicPartition("orders", 0), new OffsetAndMetadata(42L));
consumer.commitSync(offsets);

// seek к конкретному offset
consumer.seek(new TopicPartition("orders", 0), 100L);
consumer.seekToBeginning(...);
consumer.seekToEnd(...);
```

seek useful для replay событий (start from earlier point), skip poisoned messages (jump past known bad offset).

commitSync vs commitAsync trade-offs. Sync блокирует до подтверждения — надёжно but slow. Async не блокирует plus callback — быстро но crash before callback loss commit.

Практика — commitSync после каждого batch. commitAsync between batches если want быстрый intermediate commits. Combination gives reliability plus reasonable performance.

## Delivery semantics в consumer

At-most-once — commit до обработки:
```java
consumer.commitSync();
process(record);   // если упало здесь → сообщение потеряно
```

Использование для metrics где потеря одной ок.

At-least-once (default recommendation) — commit после обработки:
```java
process(record);
consumer.commitSync();
```

Дубли возможны (crash между process и commit). Always идемпотентный consumer.

Exactly-once комбо всех features. Idempotent producer. Transactional producer для multi-partition writes. isolation.level=read_committed на consumer чтобы читать только committed transactions. Обработка plus commit внутри Kafka-транзакции.

Пример stream processing exactly-once:
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

Не всегда возможно если writes идут не только в Kafka (например также в БД). Тогда либо outbox pattern либо accept at-least-once plus idempotency.

## Heartbeat и session

Consumer посылает heartbeat coordinator'у чтобы «я жив». Два разных timing параметра.

heartbeat.interval.ms=3000 — как часто (default 3 секунды). Идёт в отдельном thread от processing.

session.timeout.ms=10000 — если нет heartbeat за N миллисекунд consumer считается dead и triggers rebalance. Default 10 секунд, максимум 30 секунд до Kafka 3.0, 45 секунд в 3.0+.

max.poll.interval.ms=300000 (5 минут) — независимый timeout. Если между poll calls больше — dead. Ловит случаи где heartbeat thread живой но main processing застрял.

Разница important. Heartbeat в отдельном thread — независимо от processing time. max.poll.interval для processing time — timeout batch handling.

Если обработка долгая — options. Увеличить max.poll.interval.ms. Уменьшить max.poll.records (обрабатывать меньшие batches). Pause и resume partition для chunk-обработки — pause preventing next poll while processing continues, resume when ready.

## Rebalance listener

Callback выполняющийся при rebalance:
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

onPartitionsRevoked вызывается перед losing partitions. Хорошее место для commit offsets того что already processed чтобы не reprocess после rebalance.

onPartitionsAssigned вызывается при getting new partitions. Инициализация state, seek к desired offset if applicable, cleanup previous state.

Полезно для graceful commit при shutdown или rebalance scenarios.

## Дедуп на consumer стороне

Даже с idempotent producer дубли возможны при retry или rebalance scenarios. Consumer должен быть идемпотентен.

Уникальный messageId plus processed table:
```java
if (processedRepo.existsById(record.key())) return;
process(record);
processedRepo.save(new Processed(record.key()));
```

Идемпотентные UPDATE через condition:
```sql
UPDATE orders SET status='PROCESSED' WHERE id=? AND status='NEW';
-- второй раз ничего не изменит потому что status уже PROCESSED
```

UPSERT для inserts:
```sql
INSERT ... ON CONFLICT DO NOTHING;
```

## Многопоточная обработка

По default Kafka consumer однопоточный — один поток на consumer instance.

Для parallelism два подхода.

Первый — больше consumers в группе (до количества partitions). Каждый consumer в своём thread обрабатывает свой subset partitions. Простой approach — leverage built-in parallelism.

Второй — thread pool внутри одного consumer для обработки:
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

Caveat — теряется ordering обработки внутри partition. Если ordering matters — не делать multi-threading внутри partition.

Spring Kafka имеет concurrency setting который по сути создаёт несколько consumers в одном приложении. Более structured approach.

## Real-world типичные ошибки

Медленный consumer plus rebalance. max.poll.interval.ms истёк — coordinator убил — rebalance — after rebalance тот же consumer снова медленный — loop.

Fix. Уменьшить max.poll.records плюс увеличить max.poll.interval.ms. Или профилировать почему обработка медленная.

Poison pill. Message с invalid форматом — Deserializer падает при poll — poll throws exception — бесконечный retry потому что offset не commited.

Fix. ErrorHandlingDeserializer (Spring Kafka) оборачивает parsing и отправляет problematic messages в DLT (Dead Letter Topic). Или custom code с try/catch десериализации.

Дубли из-за crash между process и commit. Обычная реальность at-least-once semantics. Идемпотентность consumer'а обязательна — no way around.

Offset lag растёт. Consumer не догоняет producer. Причины. Медленный consumer processing time exceeds message arrival rate. Мало consumers в группе. Мало partitions (нельзя добавить больше consumers чем partitions). Downstream БД или API тормозит.

Мониторить consumer lag = latest_offset - current_offset per partition. Grafana dashboards с alerts на growing lag.

Hot partition — один key берёт большую часть трафика — одна partition перегружена, другие простаивают. Fix через sharding ключа либо custom partitioner distributing load more evenly.

## Kafka Streams кратко

Библиотека для stream processing поверх Kafka. Declarative API для transformations:
```java
KStream<String, Order> orders = builder.stream("orders");
orders
    .filter((k, v) -> v.getAmount() > 100)
    .mapValues(v -> new BigOrder(v))
    .to("big-orders");
```

Возможности. Exactly-once semantics built-in. Stateful operations через RocksDB local state stores. Joins между streams. Windowed aggregations по времени. Full stream processing capabilities.

Отдельная тема — не в этом file. Важно знать что existence — для сложных stream processing scenarios Kafka Streams мощнее чем raw consumer.

## Итоги

Producer. acks=all plus enable.idempotence=true plus reasonable batching через linger.ms и batch.size. compression.type=zstd для network efficiency.

Ключ равно partition равно ordering. Same key routes к same partition preserving order per business entity.

Consumer group делит partitions между consumers. Partition единица parallelism.

Rebalance freezes group during redistribution. CooperativeStickyAssignor минимизирует disruption.

Manual commit после обработки. commitSync для reliability. commitAsync between batches для performance.

At-least-once plus идемпотентность = стандартный setup. Exactly-once требует transactions plus consumer в read_committed mode.

max.poll.interval.ms больше времени обработки batch. Иначе rebalance loop.

Poison pill handling через ErrorHandlingDeserializer или explicit try/catch.

Мониторить consumer lag. Growing lag indicates capacity problem или downstream issue.

Multi-threading внутри consumer теряет partition ordering. Better увеличить consumers в группе если ordering не критичен.

Kafka Streams для сложных stream processing scenarios.

Дальше — Spring Kafka как high-level abstraction над raw Kafka client в Spring экосистеме.
