# 42. Kafka в production: transactions, exactly-once, monitoring, распространённые issues

## Зачем углубляться в production paradigms

В предыдущих файлах разобрали Kafka как distributed log с brokers, topics, partitions, offsets. Разобрали Spring Kafka abstractions с KafkaTemplate, @KafkaListener, error handling. Этой глубины достаточно чтобы поднять working Kafka pipeline. Но production reality приносит classes проблем которые не проявляются в testing environments. Что делать если producer нужно писать в несколько partitions atomically? Как реально достичь exactly-once semantics когда tools promise это в specific circumstances only? Что происходит когда consumer suddenly starts lagging и alerts fire — как diagnose и fix? Как настроить monitoring чтобы problem detected before customer notices?

Разница между разработчиком «использующим Kafka» и «понимающим production Kafka» очень заметна именно здесь. Первый deploys Kafka, sees green metrics, considers done. Второй знает что consumer lag это key operational metric — growing lag indicates capacity problem или downstream issue requiring action. Знает что UnderReplicatedPartitions > 0 warning об imminent risk of data loss. Знает что exactly-once semantics в Kafka работают только within Kafka boundaries — writing к external database требует Outbox pattern не Kafka transactions. Знает что rebalancing storms usually cause slow consumer processing exceeding max.poll.interval.ms — fix через adjusting timeout или reducing batch size.

В этом файле разберём production concerns деятельностью. Kafka transactions detailed — что реально делают, requirements, semantics с consumers. Exactly-once semantics — три подхода (idempotent producer, transactional producer, EOS в streams) plus fundamental limitations на external systems. Comprehensive monitoring — что мониторить на broker/topic/consumer/producer levels. Consumer lag deeper investigation. Common issues и их fixes — rebalancing storms, hot partitions, poison pills, duplicate messages, missing messages. Kafka Streams и Connect как higher-level patterns. Schema Registry для schema evolution. Best practices deployment.

## Kafka transactions detailed

Проблема которую решают Kafka transactions. Producer sends multiple messages в разные partitions. Между sends может произойти crash. Atomicity нарушена — часть sent, часть нет. Downstream systems see partial state.

Пример scenario. Order service publishes OrderCreated к «orders» topic plus PaymentRequested к «payments» topic per order. Consumer processing orders must correlate с payments. Если сrash после «orders» send но перед «payments» — payment service никогда не knows about order. Data inconsistency.

Kafka transactions предоставляют atomic multi-partition writes. Всё что происходит внутри beginTransaction/commitTransaction — либо celiком visible к read_committed consumers, либо не visible вовсе. Either both partitions get their messages, either neither does.

Setup requires transactional producer configuration. transactional.id это identifier — уникальный per producer instance, stable across restarts. При startup producer.initTransactions() coordinates с broker — resumes any pending transactions from previous incarnation, aborts им если needed.

Configuration in Spring Kafka:
```yaml
spring.kafka.producer:
  transaction-id-prefix: tx-orders-
```

Spring generates unique IDs based on prefix. Stability across restarts requires deterministic ID generation — usually based on host/pod identity.

Или manual configuration:
```properties
enable.idempotence=true
acks=all
transactional.id=tx-orders-{host}-{pid}
```

Usage паттерн. Каждая transaction — beginTransaction, sends, commit or abort:
```java
KafkaProducer<String, Object> producer = ...;
producer.initTransactions();   // один раз при startup

try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("orders", "key1", value1));
    producer.send(new ProducerRecord<>("payments", "key1", value2));
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

В Spring через @Transactional:
```java
@Transactional("kafkaTransactionManager")
public void publishAtomic() {
    template.send("orders", event1);
    template.send("payments", event2);
    // если бросит exception → abort
}
```

Consumer side для видимости committed transactions:
```properties
isolation.level=read_committed
```

Read_committed consumer видит только committed transactions. Aborted — completely invisible. Uncommitted — пока не visible до commit signal.

Trade-off. Adds latency — consumer ждёт transaction completion before delivering messages. Aborted transactions increase overhead. Coordination between transactional producer и consumer requires additional protocol messages.

Read_uncommitted (default) видит всё включая aborted transactions. Faster because no waiting. But receives messages that will be aborted — downstream processing must handle этого. Rarely appropriate когда transactions used.

## Exactly-once semantics полностью

EOS (Exactly-Once Semantics) наиболее confusing part Kafka semantics. Requires understanding what exactly is guaranteed and what isn't.

Что нужно для EOS. Idempotent producer через enable.idempotence=true — no duplicates в single partition при retries. Transactional producer — atomic writes в multiple partitions. Consumer с isolation.level=read_committed — reading only committed transactions. Consume plus process plus produce plus commit offset в one transaction.

Classical stream processing pattern для EOS. Read from input topic — process — write to output topic — commit input offset. All в one transaction:
```java
producer.beginTransaction();
try {
    for (record : consumer.poll(...)) {
        Result r = process(record);
        producer.send(new ProducerRecord<>("output", r));
    }
    producer.sendOffsetsToTransaction(
        currentOffsets(consumer.assignment()),
        consumer.groupMetadata());
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

Что происходит atomically. Все send calls к output topic. Consumer offset commit для input topic. Either все committed together, либо все aborted. Exactly-once semantics — каждый input record produces exactly one output record even under failures.

Kafka Streams делает это автоматически с processing.guarantee=exactly_once_v2. Framework обеспечивает EOS transparently для stream applications. Recommended approach для complex stream processing needing EOS.

Fundamental limitation — EOS работает только внутри Kafka. Если consumer:
- Пишет в внешнюю БД (PostgreSQL, MongoDB, etc)
- Или вызывает внешний API
- Или отправляет email/SMS

— это НЕ покрывается Kafka-транзакцией. Kafka transaction commits после DB write — если между DB write и Kafka commit crash — DB has data, Kafka message uncommitted, on retry re-executed DB write создаёт duplicate.

Для БД plus Kafka правильно — Outbox pattern. В одной DB transaction: сохранить business data plus запись в outbox таблицу. Отдельный producer читает outbox, публикует в Kafka, marks record as published. Atomicity через DB transaction. Publish eventually happens after successful commit.

Outbox provides at-least-once delivery к Kafka. Combined с идемпотентностью consumer — effective exactly-once для business perspective. Simpler than trying stretch Kafka transactions к external systems.

## Comprehensive monitoring

Broker level metrics критичны для operational awareness cluster health.

UnderReplicatedPartitions — сколько partitions currently not fully replicated. Some replicas lagging behind leader beyond threshold. > 0 warning об imminent data loss risk. Investigation required — replica broker health, network between brokers, disk capacity на affected brokers.

OfflinePartitionsCount — partition без leader. > 0 критично — эти partitions completely unavailable for read/write. Usually indicates severe broker failures или configuration issue. Data unavailable to consumers, producers cannot write.

ActiveControllerCount — должен быть exactly 1 в cluster. Multiple controllers indicates split-brain situation. Zero controllers indicates cluster без coordinator — administrative operations fail.

NetworkProcessorAvgIdlePercent — если <30% network processors перегружены — brokers cannot handle traffic. Scale-up needed. RequestHandlerAvgIdlePercent similar но для request handling threads.

BytesInPerSec/BytesOutPerSec — network traffic per broker. Uneven distribution indicates hot brokers requiring rebalancing.

Topic level metrics per-topic insights. MessagesInPerSec — production rate. BytesInPerSec/BytesOutPerSec — data volume. PartitionCount — configuration verification. LogSize — disk usage на broker. Alerting on LogSize approaching capacity предотвращает disk full incidents.

Consumer level metrics критично для health processing pipeline.

Lag — сколько отстал consumer от producer. Most important operational metric. Growing lag indicates consumer cannot keep up с production rate. Detailed section below.

CommitLatency — время commit operations. Growing latency indicates coordinator issues или network problems.

Producer level metrics. RecordSendRate — sending rate. RecordErrorRate — errors per second. Growing errors indicate configuration issues или broker problems. RequestLatency — время responses от brokers.

## Tools для monitoring

JMX это Kafka's native monitoring interface. All metrics exposed через JMX MBeans. Standard Java Management Extensions.

JMX Exporter конвертирует JMX metrics в Prometheus format. Deployed as sidecar или agent. Configures which JMX metrics to expose plus how to rename/label.

Confluent Control Center — commercial GUI от Confluent (создатели Kafka). Rich monitoring, alerting, cluster management. License required.

Kafka Manager (CMAK) — open source management UI. Community-maintained. Basic topic/broker/consumer group operations. Free alternative.

Cruise Control — automated rebalancing. Detects broker imbalance, generates reassignment plans, executes gradually. Reduces operational burden manual rebalancing.

## Consumer lag investigation

Lag = latest_offset (producer) - current_offset (consumer). Как sees command-line:
```bash
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
    --describe --group my-group

GROUP           TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
my-group        orders  0          10500           10520           20
my-group        orders  1          10480           10500           20
```

CURRENT-OFFSET consumer's current position. LOG-END-OFFSET latest position в topic. LAG difference.

Growing lag = consumer не догоняет. Investigation.

Медленный процессинг. Thread dump consumer showing where time spent. Обычно external calls внутри processing loop — database queries, HTTP calls. Optimize downstream или add async processing.

Мало consumers. Increase concurrency до количества partitions. Beyond partition count no benefit — extra consumers idle.

Мало partitions. Cannot добавить больше consumers чем partitions. Requires topic reconfiguration — create new topic с more partitions plus migrate data. Cannot change partition count on existing topic without data reshuffling.

Downstream tormoзит. Database saturated. External API rate-limited. Cache misses. Investigation goes down to root cause.

Rebalance loop. Consumers unstable — coordinator constantly rebalancing. Fix через увеличения max.poll.interval.ms или switching к CooperativeStickyAssignor.

## Prometheus plus Grafana

Standard monitoring stack. JMX Exporter scrapes metrics from brokers. Prometheus scrapes JMX Exporter endpoint. Grafana visualizes.

Grafana community dashboards доступны для Kafka. Kafka Overview показывает broker metrics. Kafka Consumer per-group lag с partition-level detail. Kafka Producer send rates и errors.

Alerts критичные для operational awareness. Lag больше 10000 на любую partition — investigate. UnderReplicatedPartitions > 0 — investigate replication. OfflinePartitionsCount > 0 — critical, immediate action. Broker down — page on-call. Disk usage approaching capacity — expand storage.

## Rebalancing storm

Постоянные rebalance — группа не потребляет. Diagnostic pattern.

Причины обычно взаимосвязанные. Consumer processes batch too long — exceeds max.poll.interval.ms default 5 minutes. Coordinator marks consumer dead. Rebalance triggered. Consumer reassigned partitions. Continues slow processing. Repeat.

Fix strategies. Уменьшить max.poll.records с default 500 к 100 или 50. Smaller batches complete faster. Увеличить max.poll.interval.ms если processing genuinely slow. But not indefinitely — too high delays failure detection.

CooperativeStickyAssignor incremental rebalance не полный freeze всей группы. Consumers gradually reassigned один за одним. Significantly reduces impact rebalance events.

Stabilize consumers. Investigate crashes causing false failures. Fix underlying processing issues. Ensure consumer application stable.

## Hot partition

Одна partition принимает большинство трафика — перегружена, другие пустуют. Common issue при uneven key distribution.

Причины. Плохой ключ — например country_code где 90% "KZ". Skewed data distribution — some users значительно more active than others.

Impact. One partition consumer overloaded. Other consumers idle. Effective throughput limited by hot partition capacity даже если infrastructure could handle much more.

Fix strategies. Sharding ключа — вместо country_code use country_code plus hash(user_id) % 10. Splits KZ traffic across 10 partitions.

Custom partitioner для complex routing logic. Sometimes business need routing patterns не conveniently expressible через key hashing.

Sometimes redesign fundamental. Не use natural id как key if distribution uneven. Composite keys balancing distribution vs ordering requirements.

## Slow consumer

Consumer сильно медленнее producer. Lag растёт. Manifested через increasing lag metric.

Fix strategies проходят escalation ladder.

Больше consumers. Up to partition count. Simple первый шаг.

Batch-обработка. Process multiple records в one operation. Bulk database inserts, bulk API calls. Reduces per-message overhead.

Async processing внутри consumer. Consumer thread only polls plus dispatches. Actual work в thread pool. Кавет — теряется partition ordering.

Оптимизация downstream. Slow database? Add indexes, tuning queries. Slow API? Investigate downstream provider capacity.

## Poison pill

Некорректное сообщение — deserializer fails на каждый poll. Consumer stuck — cannot progress past bad message. Same offset polled forever.

Fix через ErrorHandlingDeserializer как обсуждали в файле 41. Wraps actual deserializer. Deserialization exceptions attached к record. Record delivered к listener with null payload plus exception header. DefaultErrorHandler routes to DLT.

Bad message safely goes to DLT для manual review. Consumer continues с next message. No stuck.

## Duplicate messages

Consumer crash между process и commit — сообщение обрабатывается снова. Fundamental characteristic at-least-once semantics.

Fix mandatory — идемпотентность consumer'а. Processed table checking messageId. UPSERT operations. Conditional UPDATE clauses. Each processed message idempotent — repeated execution not changing state beyond first execution.

Не workaround через complex retry logic. Fundamental design pattern — accepting duplicates possible, ensuring processing safe under duplicates.

## Missing messages

Two main causes.

acks=0 или acks=1 plus leader crashes до replication на followers. Message accepted by leader, follower replication pending, leader crashes. New leader elected from followers doesn't have message. Message lost.

Fix — acks=all plus min.insync.replicas=2. Producer waits для replication к at least 2 ISR before считает send successful.

Consumer commit'нул offset до обработки — crash — пропуск. Auto-commit или incorrect ordering в manual commit code.

Fix — always commit после processing, not before. Manual commit configuration.

Combined solution — acks=all + min.insync.replicas=2 + commit after processing plus idempotent consumer. Complete reliability guarantee.

## Disk full

Kafka заполнил диск — broker падает. Common issue при high volume plus slow retention cleanup.

Причины. Retention периоды больше чем available disk allows. Producer accelerated без reviewing capacity. Failed retention cleanup из-за bug или incorrect configuration.

Fix. Retention меньше — быстрее удаление старых messages. Больше дисков — expand storage. Monitoring с alert на free space threshold — action taken перед hitting full.

Preventive — capacity planning based on ожидаемого throughput plus retention. Regular monitoring disk usage trends.

## Networked issue

Broker down — controller election — пауза — пока новый leader elected для partitions на dead broker. Consumers/producers experience temporary unavailability.

Monitoring LeaderElectionRateAndTimeMs shows how quickly recoveries happen. Long times indicate network issues или broker misconfiguration.

## Kafka Streams обзор

Библиотека для stream processing поверх Kafka. Declarative API для transformations.

Возможности. Filter, map, flatMap — basic transformations. Joins — streams-to-streams, streams-to-table. Aggregations — count, sum, custom. Windowing — tumbling (fixed non-overlapping), hopping (overlapping), session (dynamic based on activity). Stateful processing — RocksDB как local state store. Exactly-once semantics built-in.

Пример:
```java
StreamsBuilder builder = new StreamsBuilder();
KStream<String, Order> orders = builder.stream("orders");

KTable<String, Long> ordersPerCustomer = orders
    .groupBy((k, v) -> v.getCustomerId())
    .count();

ordersPerCustomer.toStream().to("customer-orders-count");

new KafkaStreams(builder.build(), config).start();
```

Использование. Real-time aggregations — computing metrics from event stream. ETL — transforming events between different formats. Event sourcing — computing state from event history.

Альтернативы. Apache Flink мощнее, сложнее. Spark Streaming для batch-like processing. ksqlDB — SQL over Kafka для non-programmers.

## Kafka Connect обзор

Framework для интеграции Kafka с external systems. No custom code required для common integrations.

Source connectors — читают из DB/файлов/API — пишут в Kafka. Пример Debezium — CDC (Change Data Capture) из PostgreSQL/MySQL к Kafka. Real-time propagation database changes.

Sink connectors — читают из Kafka — пишут в DB/Elastic/S3. Various pre-built connectors. Elasticsearch sink для indexing events. JDBC sink для writing back к databases. S3 sink для archival.

Config через REST API. Deploy как отдельный process или distributed cluster. Fault-tolerant — connector restarts on failures.

Использование. Интеграция без своего кода — common patterns supported ready-made connectors. Reduce custom development for standard integration patterns.

## Schema Registry

Централизованное хранилище schemas (Avro/Protobuf/JSON Schema). Confluent product though open source version available.

Producer публикует schema в registry — каждое сообщение содержит schema ID. Consumer читает schema ID — получает schema — десериализует.

Плюсы. Schema evolution — backward/forward compatibility rules enforced. Adding fields safe. Removing fields может ломать consumers depending on compatibility mode. Валидация schemas prevent malformed data entering topics. Компактные сообщения — schema один раз в registry, не в каждом message.

Compatibility modes определяют allowed changes. BACKWARD — new consumer can read old data. FORWARD — old consumer can read new data. FULL — both directions. NONE — no compatibility guarantees.

Обязателен для Avro в prod. Managing schemas manually нереально в scale.

## Deployment best practices

Cluster sizing. Минимум 3 broker (replication factor 3). Минимум 3 Zookeeper или KRaft controllers. Больше при нужде для throughput scaling или fault tolerance.

Retention. Обычные topics 7-30 дней. Компакция для KV forever plus periodic compaction. Мониторить disk usage.

Topics. Названия по конвенции — `<domain>.<entity>.<action>` например `knp.order.created`. Consistent naming позволяет easier discovery и filtering. Создание с partitions=X replication-factor=3 min.insync.replicas=2 для production reliability.

Producer settings production:
```
acks=all
enable.idempotence=true
compression.type=zstd
linger.ms=10
batch.size=32768
```

Consumer settings production:
```
enable.auto.commit=false
isolation.level=read_committed          # если producer transactional
max.poll.records=100                    # tune под processing time
session.timeout.ms=30000
heartbeat.interval.ms=10000
```

JVM tuning. Kafka use OS page cache extensively. Не давать всё memory к heap — leave for page cache. Heap 6 GB usually enough. Остальная RAM — OS cache.

```
-Xms6g -Xmx6g -XX:+UseG1GC -XX:MaxGCPauseMillis=20
```

G1GC preferred over CMS/Parallel для Kafka. Consistent short pauses better than throughput-optimized collectors.

## Когда Kafka vs Rabbit vs alternatives

Kafka — high throughput, event streaming, replay, exactly-once, log compaction. Wide adoption для event-driven architectures.

RabbitMQ — task queues, complex routing, priority. Better для command-style messaging.

SQS — managed AWS, простые очереди, exactly-once (FIFO). Cloud-native choice.

NATS — легковесный, low-latency messaging. Alternative для simple scenarios не needing Kafka's features.

Redis Streams — простой event log поверх Redis. Convenient если Redis already deployed.

Pulsar — Kafka++ (tiered storage, multi-tenancy), сложнее. Newer alternative gaining adoption.

Для типового КНП scenario — Rabbit уже используется. Introduction Kafka не мотивирован если Rabbit satisfies needs.

## Итоги

Kafka transactions provide atomic multi-partition writes plus atomic offset commits — только внутри Kafka boundaries.

Exactly-once semantics — idempotent producer plus transactional producer plus read_committed consumer plus consume/process/produce/commit в one transaction. Only полно works внутри Kafka.

Для DB plus Kafka atomicity — Outbox pattern. Не Kafka transactions across systems.

Consumer lag это main operational metric. Growing lag indicates capacity problem — investigation через thread dumps, consumer/partition scaling, downstream optimization.

CooperativeStickyAssignor против rebalancing storm. Incremental rebalance не freeze всей группы.

Sharding ключа против hot partition. Composite keys distributing load.

acks=all plus min.insync.replicas=2 plus idempotent producer plus replication factor 3 — no data loss guarantee.

JMX через Prometheus через Grafana — стандартный monitoring stack для Kafka.

Alerting на UnderReplicatedPartitions, OfflinePartitions, growing consumer lag, disk usage, broker down.

Kafka Streams для сложных stream processing scenarios с EOS. Kafka Connect для integration с external systems без custom code. Schema Registry для schema evolution в Avro/Protobuf.

Deployment best practices — sizing, retention, topic naming conventions, production-tuned producer/consumer settings, JVM tuning leaving OS page cache room.

Итог блока Kafka. Файлы 39-42 покрыли основы, producer/consumer/offsets, Spring Kafka abstractions, production patterns. Комплексное understanding для reliable Kafka usage в enterprise.

Дальше — Java основы: примитивы, объекты, память с deep dive в memory layout, GC roots, off-heap patterns.
