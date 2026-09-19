# 39. Kafka основы: broker, topic, partition, offset

## Что такое Kafka

Apache Kafka это distributed streaming platform. Не просто очередь — это распределённый log (log-based storage). Понимание этой фундаментальной discrepancy важно для правильного применения Kafka.

Идея. События (messages) пишутся в log — append-only структуру. Consumers читают log с любой позиции. Log НЕ удаляется после чтения (в отличие от RabbitMQ где сообщение исчезает после ack). Хранится долго — дни, недели, месяцы или годы в зависимости от retention policy.

Создан в LinkedIn в 2011 году для их internal messaging infrastructure. Open sourced и стал foundation для event streaming во многих enterprise системах. Активно развивается Apache Software Foundation.

## Kafka vs RabbitMQ

Классический вопрос для собеседований и архитектурных решений. Понимание различий определяет правильный выбор.

Model. Kafka log-based (persistent log с offset). RabbitMQ queue-based (transient messages удаляются после ack).

Delivery model. Kafka consumer pull — clients сами запрашивают messages. RabbitMQ broker push — broker активно отправляет messages consumers.

Ordering. Kafka guarantees ordering только внутри partition. RabbitMQ guarantees ordering в queue при одном consumer. Multi-consumer в обоих ломает strict ordering.

Multi-consumer patterns. Kafka consumer groups — одно сообщение доставляется одному consumer в каждой группе. RabbitMQ fanout exchange распространяет копии на много queues.

Throughput. Kafka очень высокий — миллионы messages в секунду per broker. RabbitMQ высокий но lower — обычно десятки тысяч в секунду.

Retention. Kafka по времени или размеру, не зависит от чтения. RabbitMQ удаляется после ack.

Replay. Kafka supports — можно перечитать с любого offset. RabbitMQ nope — только DLQ archive.

Priority. Kafka нет. RabbitMQ да через priority queues.

Route logic. Kafka только по partition (простой). RabbitMQ богатый — topic exchange, headers exchange, complex routing.

Transactions. Kafka full (transactional producer). RabbitMQ ограничено.

Когда Kafka. Event streaming и event sourcing. Log aggregation across services. Change data capture (CDC) from databases. Metrics ingest. High throughput scenarios. Требуется replay для reprocessing.

Когда RabbitMQ. Task queues для job processing. RPC-style request-response. Priority messages где ordering by importance. Complex routing patterns. Lower throughput requirements где Rabbit достаточно.

## Основные концепты

Архитектура Kafka cluster:
```
┌─── Kafka Cluster ────────────────────────────┐
│                                              │
│  ┌─── Broker 1 ────┐  ┌─── Broker 2 ────┐   │
│  │                 │  │                 │   │
│  │  Topic "orders" │  │  Topic "orders" │   │
│  │  Partition 0    │  │  Partition 1    │   │
│  │  (leader)       │  │  (leader)       │   │
│  │                 │  │                 │   │
│  │  Partition 1    │  │  Partition 0    │   │
│  │  (replica)      │  │  (replica)      │   │
│  └─────────────────┘  └─────────────────┘   │
│                                              │
│  ┌─── Broker 3 ────┐                        │
│  │  Partition 0    │                        │
│  │  (replica)      │                        │
│  │  Partition 1    │                        │
│  │  (replica)      │                        │
│  └─────────────────┘                        │
└──────────────────────────────────────────────┘
```

Broker это один процесс Kafka. Cluster обычно 3-9 brokers для HA plus scalability.

Topic это логическая категория сообщений. Например orders, payments, user-events. Producer пишет в конкретный topic. Consumer subscribes на конкретные topics.

Topic делится на partitions для parallelism. Каждая partition это упорядоченный log сообщений:
```
Partition 0:
[msg0] [msg1] [msg2] [msg3] [msg4] [msg5]
 offset 0-5
```

Каждое сообщение имеет offset — монотонно растущий integer identifier position в partition. Offset уникален per partition — разные partitions могут иметь same offset numbers для разных messages.

Гарантии ordering. Внутри одной partition Kafka guarantees strict ordering — messages появляются в consumer в том же order что written. Между partitions ordering НЕ guaranteed — reader может видеть messages в разном order чем publisher intended if reading multiple partitions.

Больше partitions равно больше parallelism (больше consumers могут работать одновременно). Но также больше overhead — metadata management, replication overhead, resource usage.

Offset это позиция message в partition. Монотонно растёт с каждым new message. Consumer хранит свой offset — до какой позиции прочитал. Начинает с этой позиции при рестарте.

Offset не удаляется до retention limit. Отсюда возможность replay — consumer может seek to любой offset и перечитать messages.

Replica это копия partition на другом broker. Replication factor определяет сколько replicas каждой partition (стандарт 3 для production).

Один broker это leader для каждой partition — обслуживает reads и writes. Остальные — followers — тянут данные с leader и держат synchronized copy. При падении leader — один из followers выбирается новым leader.

ISR (In-Sync Replicas) это replicas догоняющие leader (лаг меньше replica.lag.time.max.ms configuration). Только из ISR может быть выбран новый leader — гарантия что новый leader имеет самые recent committed данные.

min.insync.replicas настройка определяет сколько ISR должно подтвердить write при acks=all. Обычно 2 (leader plus один follower) для баланса reliability и availability.

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

Partitioner определяет в какую partition отправить message.

С ключом — hash(key) modulo partitions. Тот же ключ маршрутизируется в ту же partition обеспечивая ordering per key. Например все events one customer идут в one partition.

Без ключа — round-robin или sticky partitioning. Sticky (Kafka 2.4+) собирает multiple messages в one partition для batch efficiency потом switches.

Правило — если важен ordering для чего-то (userId, orderId) — использовать этот id как message key для guaranteed same-partition routing.

acks setting определяет level reliability.

acks=0 fire-and-forget. Не ждём подтверждения от broker. Максимальная скорость, потери возможны при broker unavailability или crashes.

acks=1 leader подтвердил. Быстро но если leader падает до replication на followers — потеря message.

acks=all все ISR подтвердили. Максимальная reliability, медленнее из-за replication wait.

Правило для важных данных — acks=all plus min.insync.replicas=2 для guaranteed durability.

Batching для throughput. Producer собирает messages в batch перед отправкой. linger.ms контролирует сколько ждать до отправки (default 0). batch.size максимальный размер batch (default 16 KB).

Увеличение linger.ms до 10-100 миллисекунд plus batch.size даёт значительно лучшую throughput ценой небольшой latency. Trade-off для high-throughput workloads обычно worth it.

Compression через snappy, lz4, gzip, zstd algorithms. zstd лучший баланс compression ratio и CPU cost. Больше compression меньше network usage но больше CPU для encoding/decoding.

Idempotent producer через enable.idempotence=true. Гарантирует что при retry не будет duplicates. Kafka присваивает producer ID plus sequence number на каждое message. Broker дедуплицирует базируясь на этих identifiers.

Обязательно для критичных данных. Zero cost overhead — always turn on.

Delivery guarantees через комбинации settings.

At-most-once — acks=0 или 1 plus retries=0. Может потерять, не задваивает.

At-least-once — acks=all plus retries>0. Не потеряет, может задвоить (при retry без idempotence).

Exactly-once в one producer — acks=all plus enable.idempotence=true. Не потеряет, не задваивает (в рамках одной partition).

Exactly-once transactional требует transactional.id plus producer.initTransactions. Для multiple partitions atomic writes plus consumer offset commits.

## Consumer детали

Consumer читает сообщения из topic:
```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("group.id", "order-processor");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

Consumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("orders"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
    for (ConsumerRecord<String, String> record : records) {
        process(record);
    }
    consumer.commitSync();
}
```

Detailed consumer discussion в следующем файле.

## Retention

Kafka хранит сообщения согласно retention policies.

Time-based retention. retention.ms=604800000 (7 дней по default). Через 7 дней старые segments удаляются.

Size-based retention. retention.bytes=1073741824 (1 GB per partition). При превышении — старые segments удаляются.

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

.log это сам message data. .index maps offset to byte position в .log. .timeindex maps timestamp to offset для поиска по времени.

Новый segment открывается по достижении segment.bytes (default 1 GB) или segment.ms (default 7 days).

Retention удаляет segments целиком, не отдельные messages. Это делает retention operation быстрой — просто delete files.

## ZooKeeper vs KRaft

Historical model. До Kafka 2.8 требовался Zookeeper для metadata (topics, partitions, replicas), controller election, ACLs, consumer offsets (до 0.10 — теперь в special topic).

Проблемы Zookeeper approach. Ещё одна distributed system для support и operations. Ограничение scale — around 200k partitions практический limit. Разная consistency model между Kafka и Zookeeper — sync issues возможны.

KRaft (KIP-500) mode начиная с Kafka 2.8 preview, production ready с 3.3+. Internal Raft consensus в Kafka вместо Zookeeper.

Плюсы. Одна distributed system для management. Больше scale — миллионы partitions поддерживаются. Быстрее startup и controller failover.

В Kafka 3.x можно выбирать между Zookeeper и KRaft. В 4.x (2024+) только KRaft — Zookeeper support полностью удалён.

## Controller

Один broker выбран controller (координатор cluster). Ответственности controller. Отслеживает состояние других brokers. Управляет partition leader election при failures. Обрабатывает administrative операции — создание topics, изменение partition count.

В classical Kafka с Zookeeper controller выбирается через Zookeeper election. В KRaft mode — через internal Raft consensus.

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

Consume flow:
```
1. Consumer.subscribe("orders")
2. Присоединяется к consumer group
3. Group coordinator назначает partitions
4. Consumer.poll(timeout)
5. Broker отправляет batch сообщений начиная с последнего offset
6. Consumer обрабатывает
7. Consumer.commitSync() → offset сохраняется в __consumer_offsets
```

## Deployment типичный

Kafka cluster 3-5 brokers для standard HA setup.

Zookeeper ensemble 3-5 nodes если использется классический mode.

Kafka Connect для integration с databases и other systems. Optional но common.

Schema Registry (Confluent) для Avro schemas. Ensures schema evolution compatibility.

ksqlDB или Kafka Streams для stream processing поверх Kafka data.

Monitoring. JMX metrics для broker health. Prometheus scraping. Grafana dashboards. kafka-consumer-groups.sh script для consumer lag monitoring.

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

Kafka это distributed log не queue. Persistent log storage с offset-based access.

Kafka vs RabbitMQ разные модели. Kafka log-based для streaming, high throughput, replay. Rabbit queue-based для task queues, RPC, complex routing.

Основные концепты. Broker — процесс. Topic — категория. Partition — упорядоченный log. Offset — позиция message в partition. Replica — копия для HA. ISR — replicas догоняющие leader.

Producer plus partitioner routing. Key hash для same-partition ordering. Round-robin или sticky для without key. acks levels balance reliability и speed. Idempotent producer для no duplicates.

Consumer pull based через poll. Consumer groups для parallelism. Обработка после poll plus commit offsets.

Retention time-based или size-based. Log compaction для KV stores поверх Kafka.

Segments — physical файлы. Retention deletes целые segments.

KRaft заменяет Zookeeper начиная с 3.3+. В 4.x only KRaft.

Controller координирует cluster — election в KRaft mode.

Дальше — детальный разбор producer, consumer, offsets, consumer groups как practical work with Kafka.
