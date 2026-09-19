# 40. Kafka producers, consumers, offsets, consumer groups: практическая работа

## Зачем идти глубже в consumer/producer mechanics

В предыдущем файле разобрали Kafka как distributed log — brokers, topics, partitions, offsets, replicas, ISR. Разобрали basic producer plus consumer patterns. Этой глубины достаточно для понимания архитектуры на high level. Но реальные production issues проявляются на уровне detailed mechanics — как реально работает polling, что происходит во время rebalance, как offsets коммитятся и что происходит когда crash between processing и commit, почему consumer вдруг перестал получать messages несмотря на producer's продолжение sending.

Разработчик который знает Kafka only на архитектурном уровне часто попадает в classical traps. Consumer processes slowly — max.poll.interval.ms exceeded — coordinator marks consumer dead — rebalance triggered — все group frozen на несколько seconds — after rebalance consumer again slow — loop. Poison pill в topic — deserialization fails на каждый poll — infinite retry без progress. Hot partition из-за uneven key distribution — one consumer overloaded while others idle. Все эти problems solvable но требуют understanding detailed mechanics.

Разница между «работающим с Kafka» и «понимающим Kafka» именно здесь. Первый пишет consumer loop и надеется. Второй знает что poll returns immediately if data available, ждёт до timeout если no data, plus posts heartbeat to coordinator в separate thread. Знает что commit synchronizes offset к __consumer_offsets topic — this itself Kafka producer operation with its own reliability considerations. Знает что rebalance freezes все group — CooperativeStickyAssignor reduces impact через incremental rebalance. Знает что max.poll.interval.ms is separate from session.timeout.ms — first controls processing time between polls, second controls heartbeat interval.

В этом файле разберём эти detailed mechanics. Producer detailed — internal buffer, batching, callback threading, delivery guarantees combinations. Consumer detailed — poll model internally, offset management options, delivery semantics practical. Consumer group mechanics — assignment, rebalancing algorithms, coordinator role. Poison pill handling. Hot partition mitigation. Multi-threading strategies. Real-world типичные ошибки and их fixes.

## Producer детально: internal architecture

Основной цикл работы:
```java
Producer<String, Order> producer = new KafkaProducer<>(props);

ProducerRecord<String, Order> record = new ProducerRecord<>(
    "orders",                    // topic
    order.getCustomerId(),        // key (для partitioning)
    order                         // value
);

Future<RecordMetadata> future = producer.send(record);
RecordMetadata meta = future.get();   // sync — блокирует
```

Что происходит under the hood при producer.send. Not immediate network call к broker. Instead multiple internal steps.

First, serialization. key и value serializer applied — converting objects to byte arrays. Default serializers для standard types (String, Integer, ByteArray). Для Java objects обычно custom serializer или Avro/JSON approach.

Second, partitioner call. С key — hash(keyBytes) modulo numPartitions. Без key — round-robin или sticky. Result — target partition number.

Third, adding to producer's internal buffer. Producer maintains internal buffer of pending records organized by (topic, partition). Records accumulate до batch size limit или linger.ms timeout.

Fourth, when batch ready — sender thread (separate от caller thread) sends batch to broker. Producer это multi-threaded internally — caller threads add records to buffer, sender thread handles network I/O.

Fifth, broker responds acknowledging receipt (depending на acks setting). Success или failure recorded в record's Future. If callback registered — invoked at this point (в sender thread!).

Sync запись через future.get блокирует caller thread до получения confirmation. Safe но медленно. Async запись через callback:
```java
producer.send(record, (meta, exception) -> {
    if (exception != null) {
        log.error("Send failed", exception);
    } else {
        log.info("Sent to partition={} offset={}", meta.partition(), meta.offset());
    }
});
```

Не блокирует caller thread — returns immediately after adding to buffer. Callback вызывается в I/O thread producer'а later когда broker responds. Не блокировать long callback потому что this thread doing all I/O для producer — long callback blocks other sends.

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

bootstrap.servers это список brokers для initial connection. Producer discovers full cluster через metadata request к любому из them. Обычно 2-3 addresses указывается для resilience — если один broker недоступен, другой из list будет работать.

acks=all plus enable.idempotence=true plus retries максимальный — стандартный production setup для reliability. Any transient failure retried without duplicates thanks to idempotence.

max.in.flight.requests.per.connection=5 позволяет 5 unacked requests в one connection одновременно. Trade-off latency vs throughput. Higher values give better throughput но с idempotence нужно быть careful — Kafka guarantees ordering plus deduplication только up to 5 in-flight requests. Higher values may reorder messages.

linger.ms=10 plus batch.size=32768 balance latency и throughput. Producer waits up to 10ms собирая batch до 32KB. Мостики между maximum latency limit и maximum size limit.

compression.type=zstd для network efficiency. zstd provides хороший ratio при reasonable CPU cost. Compression happens per batch — compressed batches sent to broker, broker stores compressed на disk, consumers decompress on read. Network plus disk savings often 3-5x без noticeable performance impact.

buffer.memory=33554432 (32 MB) total memory available для buffering pending records. Больше accommodates больше bursts. Слишком большое value consumes JVM heap unnecessarily.

## Partitioning стратегии

По ключу default когда key provided:
```java
new ProducerRecord<>("orders", "customer-42", order);
// hash("customer-42") % partitions → та же partition для этого customer
```

Guarantees ordering per key. Все events one customer идут в one partition в правильном order.

Round-robin без ключа. Каждое сообщение → следующая partition. Uniform distribution но no ordering guarantees per anything specific.

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

## Delivery guarantees combinations

Combinations settings определяют guarantees.

At-most-once. acks=0 или 1 plus retries=0. Может потерять messages, не задваивает. Использование для metrics где потеря одной метрики не критична.

At-least-once. acks=all plus retries>0. Не потеряет message, может задвоить (при retry без idempotence). Standard для reliable data. Idempotency consumer критична.

Exactly-once в one producer. acks=all plus enable.idempotence=true. Не потеряет, не задваивает в рамках одной partition. Ideal для single-partition scenarios.

Exactly-once transactional требует transactional.id plus producer.initTransactions plus wrap operations в beginTransaction/commitTransaction. Для multi-partition atomic writes plus consumer offset commits в one transaction.

## Consumer детально: poll model

Consumer read model fundamentally pull based. Consumer decides when to fetch messages. Different от push model в RabbitMQ где broker pushes to consumer.

Основной цикл:
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
            }
        }

        consumer.commitSync();
    }
} finally {
    consumer.close();
}
```

Что происходит на poll(timeout). Multiple internal steps.

First — if consumer part of group и joined, checked whether rebalance needed. If yes — participate in rebalance protocol. Wait для new partition assignment.

Second — for each assigned partition, check if pending fetch requests в flight. If not — send fetch request к broker owning that partition. Multiple partitions от same broker batched в one fetch request.

Third — wait for fetch responses. При arrival — deserialize records. Store в internal buffer.

Fourth — return records to caller. Up to max.poll.records (default 500).

Fifth — if no records в буфере plus no data ready — wait up to timeout. Return empty result если timeout expires.

Каждый poll также used для heartbeat к coordinator. Consumer signals «я жив, обрабатываю» через regular polls. Missing polls (например processing takes too long) — coordinator marks consumer dead.

Если между poll'ами прошло больше max.poll.interval.ms (default 5 минут) — coordinator considers consumer dead и triggers rebalance. Отсюда critical rule — обрабатывать batch быстро, для долгих операций либо увеличить max.poll.interval.ms либо offload обработку в отдельные threads.

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

auto.offset.reset определяет behavior при первом запуске когда нет saved offset. earliest — с начала topic. latest — только новые сообщения пропуская history. none — throw exception если no offset.

Для новых consumers на existing topic обычно latest — не хотим reprocess historical data. Для migration scenarios earliest — process всё accumulated data.

## Consumer group mechanics

Consumer group это набор consumers делящих обработку topic. Fundamentally coordinated mechanism для parallel processing с maintaining partition ordering.

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

Внутри Kafka group coordinator (один из brokers) manages group. Assigns partitions to consumers. Tracks heartbeats. Detects failures. Coordinates rebalances.

Правило — каждая partition назначена ровно одному consumer в group. Если consumers меньше partitions — some consumers обслуживают multiple partitions. Если consumers больше partitions — лишние простаивают completely.

Отсюда important implication. Partition это единица parallelism. Хочешь больше parallelism — создавай больше partitions при создании topic. Меньше partitions чем ожидаемых consumers — waste resources. Планировать partition count based на maximum expected consumer count.

Разные groups подписанные на same topic получают каждая свою копию сообщений. Independent processing per group.

## Rebalancing

Rebalance это перераспределение partitions между consumers группы. Triggered в нескольких сценариях. Новый consumer join группу. Consumer покинул (normal shutdown или crash). Изменение partition count в topic.

Во время rebalance — вся группа не потребляет — freeze. Может занять секунды или минуты для large groups. All group operations paused до completion. Значительный impact на processing latency during rebalance.

Rebalancing storm это проблема когда consumers нестабильны и rebalances происходят часто. Обычно из-за slow processing exceeding max.poll.interval.ms triggering false failure detection. Fixing через настройку heartbeat.interval.ms и session.timeout.ms plus max.poll.interval.ms basedна expected processing time.

Стратегии assignor определяют как partitions распределяются между consumers.

RangeAssignor default. Topics в alphabetical order, partitions distributed в ranges. Consumer 1 gets partitions 0-3, Consumer 2 gets 4-7, etc. Simple но can create uneven distribution across topics.

RoundRobinAssignor. Round-robin distribution через all topics вместе. Better balance но не preserves topic locality.

StickyAssignor. Minimizes partition reassignment при rebalance. Preserves previous assignments where possible. Only necessary changes made.

CooperativeStickyAssignor (Kafka 2.4+). Incremental rebalance — не полный freeze. Consumers gradually reassigned один за одним, keeping most active. Significantly reduces disruption during rebalances.

Правило для новых consumers — CooperativeStickyAssignor. Значительно меньше disruption во время rebalance. Migration от older assignors requires coordinated update всех consumers в group.

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

Обработать сначала, только потом commit. Если crash до commit — messages обрабатываются снова при restart. Idempotency обязательна.

commitSync внутри — Kafka producer operation. Consumer opens connection к coordinator, sends offset commit request, waits for response. Small overhead per commit — не хочется commit после каждого message в high-throughput scenarios.

Ручной offset control:
```java
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
offsets.put(new TopicPartition("orders", 0), new OffsetAndMetadata(42L));
consumer.commitSync(offsets);

consumer.seek(new TopicPartition("orders", 0), 100L);
consumer.seekToBeginning(...);
consumer.seekToEnd(...);
```

seek useful для replay событий (start from earlier point), skip poisoned messages (jump past known bad offset), reprocess specific range.

commitSync vs commitAsync trade-offs. Sync блокирует до подтверждения — надёжно but slow. Async не блокирует plus callback — быстро но crash before callback loss commit.

Практика — commitSync после каждого batch. commitAsync между batches если want быстрый intermediate commits. Combination gives reliability plus reasonable performance.

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

Exactly-once комбо всех features. Idempotent producer. Transactional producer для multi-partition writes. isolation.level=read_committed на consumer чтобы читать только committed transactions. Обработка plus commit внутри Kafka-транзакции:
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

Consumer посылает heartbeat coordinator'у чтобы signal alive. Два разных timing параметра важно understand.

heartbeat.interval.ms=3000 — как часто (default 3 секунды). Идёт в отдельном thread от processing.

session.timeout.ms=10000 — если нет heartbeat за N миллисекунд consumer считается dead и triggers rebalance. Default 10 секунд.

max.poll.interval.ms=300000 (5 минут) — независимый timeout. Если между poll calls больше — dead. Ловит случаи где heartbeat thread живой но main processing застрял.

Разница important. Heartbeat в отдельном thread — независимо от processing time. max.poll.interval для processing time — timeout batch handling.

Если обработка долгая — options. Увеличить max.poll.interval.ms. Уменьшить max.poll.records (обрабатывать меньшие batches). Pause и resume partition для chunk-обработки — pause preventing next poll while processing continues, resume when ready.

## Rebalance listener

Callback выполняющийся при rebalance:
```java
consumer.subscribe(List.of("orders"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> parts) {
        consumer.commitSync();
    }
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> parts) {
        // Инициализация state для новых partitions
    }
});
```

onPartitionsRevoked вызывается перед losing partitions. Хорошее место для commit offsets already processed чтобы не reprocess после rebalance.

onPartitionsAssigned вызывается при getting new partitions. Инициализация state, seek к desired offset if applicable, cleanup previous state.

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

Spring Kafka имеет concurrency setting который по сути создаёт несколько consumers в одном приложении. Более structured approach через framework abstraction.

## Real-world типичные ошибки

Медленный consumer plus rebalance. max.poll.interval.ms истёк — coordinator убил — rebalance — after rebalance тот же consumer снова медленный — loop.

Fix. Уменьшить max.poll.records плюс увеличить max.poll.interval.ms. Или профилировать почему обработка медленная — обычно external calls внутри processing loop.

Poison pill. Message с invalid форматом — Deserializer падает при poll — poll throws exception — бесконечный retry потому что offset не commited.

Fix. ErrorHandlingDeserializer (Spring Kafka) оборачивает parsing и отправляет problematic messages в DLT (Dead Letter Topic). Или custom code с try/catch десериализации плюс skip poisoned messages.

Дубли из-за crash между process и commit. Обычная реальность at-least-once semantics. Идемпотентность consumer'а обязательна — no way around.

Offset lag растёт. Consumer не догоняет producer. Причины. Медленный consumer processing time exceeds message arrival rate. Мало consumers в группе. Мало partitions (нельзя добавить больше consumers чем partitions). Downstream БД или API тормозит.

Мониторить consumer lag = latest_offset - current_offset per partition. Grafana dashboards с alerts на growing lag. Sustained lag increase indicates capacity problem needing action.

Hot partition — один key берёт большую часть трафика — одна partition перегружена, другие простаивают. Fix через sharding ключа либо custom partitioner distributing load more evenly. Sometimes require redesign key strategy — не use natural id как key если distribution uneven.

## Kafka Streams кратко

Библиотека для stream processing поверх Kafka. Declarative API для transformations:
```java
KStream<String, Order> orders = builder.stream("orders");
orders
    .filter((k, v) -> v.getAmount() > 100)
    .mapValues(v -> new BigOrder(v))
    .to("big-orders");
```

Возможности. Exactly-once semantics built-in. Stateful operations через RocksDB local state stores. Joins между streams. Windowed aggregations по времени.

Отдельная тема — не в этом file. Важно знать что existence — для сложных stream processing scenarios Kafka Streams мощнее чем raw consumer.

## Итоги

Producer internal — buffer, batching, sender thread separate от caller. acks=all plus enable.idempotence=true plus reasonable batching через linger.ms и batch.size. compression.type=zstd для network efficiency.

Ключ равно partition равно ordering. Same key routes к same partition preserving order per business entity.

Consumer pull model. Poll returns immediately if data, waits до timeout otherwise. Также drives heartbeats к coordinator.

Consumer group делит partitions между consumers. Partition единица parallelism. Coordinator manages assignment plus rebalancing.

Rebalance freezes group during redistribution. CooperativeStickyAssignor минимизирует disruption через incremental rebalance.

Manual commit после обработки. commitSync для reliability. commitAsync между batches для performance.

At-least-once plus идемпотентность = стандартный setup. Exactly-once требует transactions plus consumer в read_committed mode.

max.poll.interval.ms больше времени обработки batch. Иначе rebalance loop.

Poison pill handling через ErrorHandlingDeserializer или explicit try/catch.

Мониторить consumer lag. Growing lag indicates capacity problem или downstream issue.

Multi-threading внутри consumer теряет partition ordering. Better увеличить consumers в группе если ordering не критичен.

Kafka Streams для сложных stream processing scenarios с exactly-once semantics.

Дальше — Spring Kafka как high-level abstraction над raw Kafka client в Spring экосистеме.
