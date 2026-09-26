# 39. Kafka основы: distributed log, brokers, topics, partitions

## Зачем понимать Kafka глубже как «просто очереди»

Разработчик приходящий к Kafka из RabbitMQ background обычно первое время думает — «это просто ещё одна очередь сообщений». Producer пишет, consumer читает, брокер посередине. Кажется концептуально similar. Но эта аналогия скрывает fundamental differences которые определяют когда Kafka правильный выбор а когда RabbitMQ. И разработчик который не понимает эти differences часто использует Kafka wrong — как temporal queue where messages disappear after processing, теряя все advantages которые Kafka predлагает.

Kafka это не очередь. Kafka это распределённый log — immutable, append-only, ordered sequence событий которая хранится долго (дни, недели, месяцы, годы) и может быть перечитана многократно. Из этого fundamental design flows все Kafka's unique characteristics — high throughput благодаря sequential write patterns, replay capability для reprocessing, log compaction для materialized views, semantic differences в how consumers work. Понимание Kafka как log а не queue — key to using it effectively.

Разница между разработчиком «использующим Kafka» и «понимающим Kafka» проявляется в architectural decisions. Первый пишет producer send и надеется что «сообщение придёт». Второй знает что acks=0 fire-and-forget может silently lose messages при broker crashes, acks=1 может потерять если leader падает перед replication, acks=all с min.insync.replicas=2 обеспечивает durability только когда достаточно ISR. Знает что partition это единица parallelism — если topic имеет 3 partitions больше 3 consumers в group будут idle. Знает что ordering guarantee только внутри partition — message key determines partition через hash — все events one customer в one partition для ordered processing.

В этом файле разберём Kafka с этой fundamentally deep perspective. Что делает Kafka distributed log а не queue. Detailed comparison с RabbitMQ по multiple dimensions. Основные концепты — brokers, topics, partitions, offsets, replicas, ISR. Producer detailed — partitioner strategies, acks levels, batching, compression, idempotency. Consumer basics (deeper explanation в следующем файле). Retention policies plus log compaction. Segments plus files на disk. ZooKeeper deprecation plus KRaft transition. Controller role. Publish plus consume happy path stepping through. Practical usage в КНП.

## Kafka как distributed log

Fundamental design decision Kafka — distributed commit log. Понимание implications requires unpacking each word.

Log означает append-only ordered sequence. New events added at end. Existing events never modified. Order preserved. Analogous к database transaction log or system audit log — records как «this happened at this time».

Distributed означает spread across multiple nodes (brokers) для scalability и fault tolerance. Log physically partitioned across brokers. Каждая partition это independent log. Combined из всех partitions forms full topic.

Commit означает durable — once written и confirmed, stays written. Not transient like queue где message consumed and disappears.

Из этого design flows several key properties отличающие Kafka от queues.

Persistence beyond consumption. Message read by consumer stays in log. Multiple consumers can read same messages independently. Replay from any point in log possible. Data warehouse can rebuild state by re-reading historical events. Analytics jobs can process past data.

High throughput через sequential writes. Log's fundamental operation — append at end. Sequential writes to disk orders of magnitude faster than random. Modern disks including SSDs favor sequential access. Kafka's design leverages this — millions of messages per second per broker achievable.

Consumer decoupling. Producers publish без knowledge which consumers exist. Consumers read at own pace independently. New consumer can join and start from any point (beginning, current end, specific timestamp).

Ordering guarantees. Within partition — strict order preserved. Between partitions — no ordering guaranteed. Enables parallelism plus preserved order per key (all events one customer in one partition через hash routing).

Storage until retention limit. Not deleted upon consumption. Deleted after time period (retention.ms) или size limit (retention.bytes) или through log compaction (keep only latest per key).

## Kafka vs RabbitMQ: detailed comparison

Classical вопрос для собеседований и architectural decisions. Понимание differences определяет правильный выбор.

Model. Kafka log-based (persistent log с offsets). RabbitMQ queue-based (transient messages удаляются после ack). Fundamental different concepts.

Delivery model. Kafka consumer pull — clients periodically poll for new data. RabbitMQ broker push — broker actively sends messages consumers when available. Pull vs push tradeoffs — pull gives consumer control over pace, push gives immediate delivery.

Ordering. Kafka guarantees ordering только внутри partition. RabbitMQ guarantees ordering в queue при одном consumer. Multi-consumer в обоих ломает strict ordering.

Multi-consumer patterns. Kafka consumer groups — одно сообщение доставляется одному consumer в каждой группе. Different groups get independent copies. RabbitMQ fanout exchange распространяет копии на много queues. Different concept but similar effect.

Throughput. Kafka очень высокий — миллионы messages в секунду per broker потому что sequential writes. RabbitMQ высокий но lower — обычно десятки тысяч в секунду. Kafka wins for high-volume scenarios.

Retention. Kafka по времени или размеру, не зависит от чтения. Messages accessible until retention limit. RabbitMQ удаляется после ack — transient by default. Different fundamental behaviors.

Replay. Kafka supports fully — consumer can seek to any offset и перечитать. RabbitMQ nope — только DLQ archive as workaround. Kafka's killer feature для event sourcing scenarios.

Priority. Kafka нет — offset order strictly. RabbitMQ да через priority queues.

Route logic. Kafka только по partition через hash key (простой). RabbitMQ богатый — topic exchange, headers exchange, complex routing.

Transactions. Kafka full transactional producer support. RabbitMQ ограничено — publisher confirms плюс transactions но performance implications.

Когда Kafka. Event streaming and event sourcing. Log aggregation across services. Change data capture (CDC) from databases. Metrics ingest. High throughput scenarios с millions of events. Required replay для reprocessing bugs или new consumers rebuilding state.

Когда RabbitMQ. Task queues для job processing. RPC-style request-response. Priority messages где ordering by importance. Complex routing patterns. Lower throughput requirements где Rabbit sufficient.

Не universal — often both used в same architecture для разных purposes. RabbitMQ для командного (task) messaging. Kafka для event streaming. Complementary tools.

## Основные концепты Kafka

Broker это один процесс Kafka. Cluster обычно 3-9 brokers для HA plus scalability. Каждый broker independently accepts producer writes и serves consumer reads для partitions на своей стороне.

Topic это логическая категория сообщений. Например orders, payments, user-events. Producer пишет в конкретный topic. Consumer subscribes на конкретные topics.

Partition это единица parallelism plus ordering. Topic делится на partitions — обычно 3-100+ depending на expected throughput. Каждая partition это упорядоченный log сообщений хранящихся на конкретном broker (plus replicas):
```
Partition 0:
[msg0] [msg1] [msg2] [msg3] [msg4] [msg5]
 offset 0-5
```

Ключевые properties partition. Ordering strict within partition — messages появляются в consumer в том order that written. Ordering не guaranteed between partitions. Parallelism unit — каждая partition can be processed independently от других.

Offset это позиция message в partition. Монотонно растёт с каждым new message. Unique per partition — разные partitions могут иметь same offset numbers для разных messages. Consumer хранит свой offset — до какой позиции прочитал.

Больше partitions равно больше parallelism. Consumers в group divide partitions between themselves — up к total partition count. More partitions permit more parallel consumers. But also overhead — metadata management, replication overhead, resource usage. Sweet spot обычно 6-30 depending on scale.

Replica это копия partition на другом broker. Replication factor определяет сколько replicas каждой partition. Standard for production — 3 replicas. Один broker это leader for каждой partition — обслуживает reads и writes. Остальные — followers — тянут данные с leader и держат synchronized copy.

At any moment одна partition имеет один leader на одном broker plus (replication factor - 1) followers на других brokers. При падении leader — один из followers выбирается новым leader через coordinated process controller-directed.

ISR (In-Sync Replicas) это replicas догоняющие leader (лаг меньше replica.lag.time.max.ms configuration — default 30 seconds). Только из ISR может быть выбран новый leader. Гарантия что новый leader имеет самые recent committed данные.

min.insync.replicas настройка определяет сколько ISR должно подтвердить write при acks=all. Обычно 2 (leader plus один follower). При insufficient ISR — writes fail preventing acknowledgment of data that could be lost.

## Producer детали

Producer записывает сообщения в topic:
```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "all");

Producer<String, String> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("orders", "key1", "message"));
producer.close();
```

Partitioner определяет в какую partition отправить message. Три standard strategies plus custom possible.

С ключом (default когда key provided) — hash(key) modulo partitions. Тот же ключ маршрутизируется в ту же partition обеспечивая ordering per key. Все events одного customer идут в одну partition — обрабатываются в порядке. Ключевая feature для business logic requiring ordering per entity.

Без ключа round-robin (older Kafka) или sticky partitioning (Kafka 2.4+). Sticky batches multiple messages в one partition до filled batch, потом switches. Better batching efficiency чем pure round-robin.

Правило — если важен ordering для чего-то (userId, orderId, sessionId) — использовать этот id как message key. Kafka guarantees same partition через hash routing. Consumer of that partition обрабатывает in order.

Custom partitioner для business-specific logic. Например routing по region — все Kazakhstan customers в partition 0, Russia в partition 1. Implements Partitioner interface. Registered через partitioner.class configuration.

acks setting определяет level reliability. Fundamental tradeoff durability vs performance.

acks=0 fire-and-forget. Producer не ждёт подтверждения. Максимальная скорость. Может потерять при broker unavailability или crashes. Only suitable для metrics или other data где occasional loss acceptable.

acks=1 leader подтвердил. Fast. Если leader падает до replication to followers — message lost. Compromise level — durability против performance.

acks=all все ISR подтвердили. Максимальная reliability. Slowest потому что must wait for followers to sync. Standard for production data где durability matters.

Правило для важных данных — acks=all plus min.insync.replicas=2 (leader plus один follower minimum). Обеспечивает guaranteed durability при available ISR.

Batching для throughput. Producer собирает messages в batch перед отправкой. Multiple messages sent в one network round-trip. linger.ms контролирует сколько ждать до отправки (default 0 — send immediately). batch.size максимальный размер batch (default 16 KB).

Увеличение linger.ms до 10-100 миллисекунд plus batch.size до 32KB даёт significantly better throughput ценой небольшой latency. Trade-off worth it для high-throughput workloads.

Compression через snappy, lz4, gzip, zstd algorithms. compression.type=zstd лучший баланс compression ratio и CPU cost. Больше compression меньше network usage но больше CPU для encoding/decoding.

Idempotent producer через enable.idempotence=true. Гарантирует что при retry не будет duplicates. Kafka присваивает producer ID plus sequence number на каждое message. Broker дедуплицирует базируясь на этих identifiers.

Как работает internally. Producer генерирует monotonically increasing sequence per partition. Sends message plus sequence to broker. Broker tracks max sequence seen per (producer_id, partition). If sequence already seen — duplicate, silently discarded. If sequence expected next — accepted. If gap in sequence — error (out of order).

Обязательно для критичных данных. Zero cost overhead — always turn on.

Delivery guarantees через комбинации settings. Full picture:

At-most-once. acks=0 или 1 plus retries=0. Может потерять messages, не задваивает.

At-least-once. acks=all plus retries>0. Не потеряет message, может задвоить (при retry без idempotence).

Exactly-once в one producer. acks=all plus enable.idempotence=true. Не потеряет, не задваивает в рамках одной partition.

Exactly-once transactional требует transactional.id plus producer.initTransactions plus wrap operations в beginTransaction/commitTransaction. Для multi-partition atomic writes plus consumer offset commits в one atomic operation.

## Consumer basics (детально в файле 40)

Consumer читает сообщения из topic. Basic loop:
```java
Consumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("orders"));

while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
    for (ConsumerRecord<String, String> record : records) {
        process(record.value());
    }
    consumer.commitSync();
}
```

Detailed consumer discussion в следующем файле — poll model, offset management, consumer groups, rebalancing, delivery semantics, threading. Здесь только overview.

## Retention

Kafka хранит сообщения согласно retention policies. Multiple policies possible.

Time-based retention default. retention.ms=604800000 (7 дней). Через 7 дней старые segments удаляются. Standard для most topics.

Size-based retention. retention.bytes=1073741824 (1 GB per partition). При превышении — старые segments удаляются. Полезно для limiting disk usage.

Combined — earliest of time or size triggers deletion. Whichever hits first.

Log compaction это специальная policy. Не удаляет по времени, а держит последнее значение для каждого key:
```
Original log:
key=A, value=1
key=B, value=1
key=A, value=2
key=B, value=2

After compaction:
key=A, value=2
key=B, value=2
```

Использование для KV-store поверх Kafka. Event sourcing snapshots. Kafka Streams state stores. Configuration data.

Настройка через cleanup.policy=compact или delete,compact для гибрида (compact plus time-based cleanup).

Правильные retention значения. Обычные event topics — 7-30 дней retention достаточно. Analytics и replay scenarios — месяцы. Compaction для KV — вечно plus periodic compaction.

## Segments и файлы

Partition на диске это набор segments. Каждый segment это pair файлов plus indexes:
```
/var/lib/kafka/data/orders-0/
├── 00000000000000000000.log       ← сегмент 1 (offset 0 - 999)
├── 00000000000000000000.index
├── 00000000000000000000.timeindex
├── 00000000000000001000.log       ← сегмент 2 (offset 1000+)
├── 00000000000000001000.index
└── 00000000000000001000.timeindex
```

.log это сам message data — actual bytes messages. .index maps offset to byte position в .log file для быстрого поиска. .timeindex maps timestamp to offset для поиска по времени.

Новый segment открывается по достижении segment.bytes (default 1 GB) или segment.ms (default 7 days). Old segments become read-only sealed files. New writes go к current active segment.

Retention удаляет segments целиком не отдельные messages. Это делает retention operation быстрой — просто delete files. Cannot delete individual messages by design.

Sequential access patterns benefit filesystem cache. Recent writes stay в OS page cache. Consumers reading recent data serve from cache without disk I/O. Old data (past cache size) requires actual disk reads.

## ZooKeeper vs KRaft transition

Historical model — до Kafka 2.8 требовался Zookeeper для metadata (topics, partitions, replicas), controller election, ACLs, consumer offsets (до 0.10 — теперь в special topic).

Проблемы Zookeeper approach. Ещё одна distributed system для support и operations. Ограничение scale — около 200k partitions практический limit. Разная consistency model между Kafka и Zookeeper — sync issues возможны.

KRaft (KIP-500) mode начиная с Kafka 2.8 preview, production ready с 3.3+. Internal Raft consensus в Kafka вместо Zookeeper.

Плюсы. Одна distributed system для management. Больше scale — миллионы partitions поддерживаются. Быстрее startup и controller failover.

В Kafka 3.x можно выбирать между Zookeeper и KRaft. В 4.x (2024+) только KRaft — Zookeeper support полностью удалён. Trend в industry — избавление от external coordination systems в пользу internal Raft consensus.

## Controller role

Один broker выбран controller (координатор cluster). Ответственности controller.

Отслеживает состояние других brokers через heartbeats. Detect failures через missed heartbeats.

Управляет partition leader election при failures. Если leader for partition X down — controller выбирает new leader из ISR followers. Announces choice всем brokers plus clients.

Обрабатывает administrative операции — создание topics, изменение partition count, reassignment partitions. Coordinate operations across cluster.

В classical Kafka с Zookeeper controller выбирается через Zookeeper election mechanism. В KRaft mode — через internal Raft consensus among controller-eligible brokers.

Controller failure само по себе triggers election of new controller. Metadata state maintained (в Zookeeper classical или KRaft log) — new controller reads state и takes over. Cluster continues functioning through controller transition — data plane not affected.

## Publish и consume happy path

Publish flow полностью:
```
1. Producer.send(record)
2. Partitioner выбирает partition (по ключу или round-robin)
3. Записывается в batch
4. При заполнении batch или linger.ms → отправка на broker (leader partition)
5. Leader записывает в log (append-only)
6. Followers тянут → записывают
7. acks=all → все ISR подтвердили → producer получает ack
```

Каждый step имеет tuning implications. Batching (steps 3-4) balance latency vs throughput. Replication (step 6) durability vs speed. ISR waiting (step 7) confidence vs completion time.

Consume flow:
```
1. Consumer.subscribe("orders")
2. Присоединяется к consumer group (coordinator assigns partitions)
3. Group coordinator назначает partitions
4. Consumer.poll(timeout)
5. Broker отправляет batch сообщений начиная с последнего offset
6. Consumer обрабатывает
7. Consumer.commitSync() → offset сохраняется в __consumer_offsets
```

Detailed каждого step в следующем файле.

## Deployment типичный

Kafka cluster 3-5 brokers для standard HA setup. Replication factor 3 requires minimum 3 brokers. 5 brokers allows более aggressive replication settings.

Zookeeper ensemble 3-5 nodes если использется классический mode. Или KRaft controllers в новых setups.

Kafka Connect для integration с databases и other systems. Optional plugin architecture для source и sink connectors.

Schema Registry (Confluent) для Avro schemas. Ensures schema evolution compatibility между producers и consumers. Critical для long-term data pipeline reliability.

ksqlDB или Kafka Streams для stream processing поверх Kafka data. Higher-level abstractions над raw consumer API.

Monitoring. JMX metrics для broker health. Prometheus scraping через JMX exporter. Grafana dashboards. kafka-consumer-groups.sh script для consumer lag monitoring. Consumer lag это main operational metric — divergence between produced и consumed indicates capacity problems.

## Terminology summary

Topic — категория messages.
Partition — упорядоченный log внутри topic для parallelism.
Offset — позиция message в partition.
Broker — один Kafka node.
Producer — пишет messages в topics.
Consumer — читает messages из topics.
Consumer group — набор consumers deliver same topic delivering distinct partitions.
Rebalance — перераспределение partitions между consumers of group.
Replica — копия partition на другом broker.
ISR — In-Sync Replicas догоняющие leader.
Leader/Follower — роли replica per partition.
Controller — cluster coordinator.
Retention — политика хранения old messages.
Log compaction — keep only latest per key policy.

## Итоги

Kafka это distributed log не queue. Persistent log storage с offset-based access. Sequential write patterns give high throughput. Retention beyond consumption enables replay.

Kafka vs RabbitMQ разные модели. Kafka log-based для streaming, high throughput, replay. Rabbit queue-based для task queues, RPC, complex routing. Часто used together для different purposes.

Основные концепты. Broker — процесс. Topic — категория. Partition — упорядоченный log unit of parallelism. Offset — позиция message в partition. Replica — копия для HA. ISR — replicas догоняющие leader.

Producer plus partitioner routing. Key hash для same-partition ordering — key business entity events in same partition. Round-robin или sticky для without key. acks levels balance reliability и speed. Idempotent producer для no duplicates.

Consumer pull based через poll. Consumer groups для parallelism. Обработка после poll plus commit offsets. Details в файле 40.

Retention time-based или size-based или combined. Log compaction для KV stores поверх Kafka.

Segments — physical файлы. Retention deletes целые segments not individual messages.

KRaft заменяет Zookeeper начиная с 3.3+. В 4.x only KRaft. Trend toward internal Raft consensus.

Controller координирует cluster — leader elections, admin operations. Election в KRaft mode через Raft.

Дальше — детальный разбор producer, consumer, offsets, consumer groups как practical work with Kafka.
