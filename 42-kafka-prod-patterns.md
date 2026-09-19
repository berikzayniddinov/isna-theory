# 42. Kafka в проде: transactions, exactly-once, monitoring

Distributed transactions в Kafka, exactly-once, мониторинг, типовые проблемы.

---

## 1. Kafka transactions

### 1.1 Проблема

Producer шлёт много сообщений в разные partitions. Между ними может упасть → **atomicity нарушена** (часть отправлена, часть нет).

Consumer читает → обрабатывает → write в другой topic → commit offset. Между шагами может упасть → **дубли или потери**.

### 1.2 Идея транзакций

Всё что происходит внутри `beginTransaction ... commitTransaction` — **атомарно**:
- Все sends в разные partitions.
- Commit offsets для нескольких partitions.

Транзакция либо целиком видна consumer'ам (с `isolation.level=read_committed`), либо не видна.

### 1.3 Настройка producer

```yaml
spring.kafka.producer:
  transaction-id-prefix: tx-orders-
```

Или руками:
```properties
enable.idempotence=true
acks=all
transactional.id=tx-orders-{host}-{pid}
```

**`transactional.id`** должен быть уникальный per producer instance и **стабильный между рестартами** (для recovery).

### 1.4 Использование

```java
KafkaProducer<String, Object> producer = ...;
producer.initTransactions();   // один раз при старте

try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("orders", "key1", value1));
    producer.send(new ProducerRecord<>("payments", "key1", value2));
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

В Spring:
```java
@Autowired KafkaTemplate<String, Object> template;

@Transactional("kafkaTransactionManager")
public void publishAtomic() {
    template.send("orders", event1);
    template.send("payments", event2);
    // если бросит → abortTransaction
}
```

### 1.5 Consumer read_committed

```properties
isolation.level=read_committed
```

Consumer видит только commit'нутые транзакции. Aborted — пропускаются. Uncommitted — пока не видит.

**Замедляет latency** — consumer ждёт commit чтобы отдать.

`isolation.level=read_uncommitted` (default) — видит всё, включая aborted (нет транзакций → быстрее).

---

## 2. Exactly-Once Semantics (EOS)

### 2.1 Что нужно

- **Idempotent producer** (`enable.idempotence=true`).
- **Transactional producer** — atomic writes в несколько partitions.
- **Consumer с `read_committed`**.
- **Consume + process + produce + commit offset** в одной транзакции.

### 2.2 Stream processing (Kafka Streams)

Классический pattern для EOS:
```
Read from topic A → process → Write to topic B → Commit offset A
```

Всё в одной Kafka-транзакции:
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

Kafka Streams делает это автоматически с `processing.guarantee=exactly_once_v2`.

### 2.3 Кавет — не всё exactly-once

EOS работает **только внутри Kafka**. Если consumer:
- Пишет в внешнюю БД,
- Или вызывает внешний API,

— это не покрывается Kafka-транзакцией → **идемпотентность обязательна**.

Для БД + Kafka правильно — **outbox pattern**:
1. В одной DB-tx: сохранить + запись в outbox.
2. Отдельный producer читает outbox → публикует в Kafka.

---

## 3. Мониторинг

### 3.1 Что мониторить

**Broker level**:
- **UnderReplicatedPartitions** — сколько partitions не полностью реплицированы. > 0 → тревога.
- **OfflinePartitionsCount** — partition без leader. > 0 → критично.
- **ActiveControllerCount** — должен быть 1 в кластере.
- **NetworkProcessorAvgIdlePercent** — если <30% → перегружен.
- **RequestHandlerAvgIdlePercent** — то же для request handler.
- **BytesInPerSec / BytesOutPerSec** — трафик.

**Topic level**:
- **MessagesInPerSec**.
- **BytesInPerSec / BytesOutPerSec**.
- **PartitionCount**.
- **LogSize** — размер на диске.

**Consumer level**:
- **Lag** — сколько отстал consumer от producer (см. §3.3).
- **CommitLatency**.

**Producer level**:
- **RecordSendRate**.
- **RecordErrorRate**.
- **RequestLatency**.

### 3.2 Инструменты

- **JMX** — Kafka экспортирует метрики через JMX.
- **JMX Exporter** для Prometheus.
- **Confluent Control Center** (commercial).
- **Kafka Manager / CMAK** (UI).
- **Cruise Control** — automated rebalancing.

### 3.3 Consumer lag

**Lag** = `latest_offset (producer) - current_offset (consumer)`.

```bash
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
    --describe --group my-group

GROUP           TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
my-group        orders  0          10500           10520           20
my-group        orders  1          10480           10500           20
```

Растущий lag = consumer не догоняет.

Диагностика:
- Медленный процессинг → thread dump.
- Мало consumers → добавить (до max = partitions count).
- Downstream (БД, API) тормозит.
- Rebalance loop.

### 3.4 Prometheus + Grafana

Grafana dashboards (community):
- **Kafka Overview** — broker metrics.
- **Kafka Consumer** — per-group lag.
- **Kafka Producer** — send rates.

Alerts:
- Lag > 10000 на любой partition.
- UnderReplicatedPartitions > 0.
- OfflinePartitionsCount > 0.
- Broker down.

---

## 4. Типовые проблемы

### 4.1 Rebalancing storm

Постоянные rebalance → группа не потребляет.

Причины:
- Consumer долго обрабатывает batch → `max.poll.interval.ms` истёк → dead → rebalance → снова.
- Нестабильные consumers (падают).
- Cluster нестабилен.

Fix:
- Уменьшить `max.poll.records` или увеличить `max.poll.interval.ms`.
- **CooperativeStickyAssignor** — incremental rebalance без freeze всей группы.
- Стабилизировать consumers.

### 4.2 Hot partition

Одна partition принимает большинство трафика → перегружена, другие пусты.

Причины:
- Плохой ключ (например, `country_code` где 90% "KZ").
- Skewed data.

Fix:
- Sharding ключа (например, `country_code + hash(user_id) % 10`).
- Custom partitioner.

### 4.3 Slow consumer

Consumer сильно медленнее producer → lag растёт.

Fix:
- Больше consumers (до количества partitions).
- Batch-обработка.
- Async processing внутри consumer.
- Оптимизация downstream (БД, API).

### 4.4 Poison pill

Некорректное сообщение → deserializer fails → consumer stuck.

Fix: `ErrorHandlingDeserializer` (см. `41-spring-kafka.md`).

### 4.5 Duplicate messages

Consumer crash между process и commit → сообщение обрабатывается снова.

Fix: **идемпотентность consumer'а** (processed table, UPSERT, conditional UPDATE).

### 4.6 Missing messages

- `acks=0` или `acks=1` + leader упал до replication → потеря.
- Consumer commit'нул offset до обработки → crash → пропуск.

Fix: `acks=all` + `min.insync.replicas=2` + commit после обработки.

### 4.7 Disk full

Kafka заполнил диск → broker падает.

Fix:
- Retention меньше.
- Больше дисков.
- Мониторинг с alert на free space.

### 4.8 Networked issue

Broker down → controller election → пауза → пока новый leader.

Мониторить `LeaderElectionRateAndTimeMs`.

---

## 5. Kafka Streams (обзор)

Библиотека для stream processing.

Возможности:
- Filter, map, flatMap.
- Joins (streams-to-streams, streams-to-table).
- Aggregations (count, sum, custom).
- Windowing (tumbling, hopping, session).
- Stateful processing (RocksDB как local state).
- Exactly-once semantics.

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

Использование: real-time aggregations, ETL, event sourcing.

Альтернативы: Apache Flink (мощнее, сложнее), Spark Streaming, ksqlDB (SQL over Kafka).

---

## 6. Kafka Connect

Framework для интеграции Kafka с external systems.

**Source connectors** — читают из DB/файлов/API → пишут в Kafka.

Пример: **Debezium** — CDC (Change Data Capture) из PostgreSQL/MySQL → Kafka.

**Sink connectors** — читают из Kafka → пишут в DB/Elastic/S3/...

Config через REST. Deploy как отдельный process или distributed cluster.

Использование: интеграция без своего кода.

---

## 7. Schema Registry (Confluent)

Централизованное хранилище schemas (Avro/Protobuf/JSON Schema).

Producer публикует schema в registry → каждое сообщение содержит **schema ID**. Consumer читает schema ID → получает schema → десериализует.

Плюсы:
- Schema evolution (backward/forward compatibility).
- Валидация schemas.
- Компактные сообщения (schema один раз, не в каждом).

Обязателен для Avro в проде.

---

## 8. Deployment best practices

### 8.1 Cluster sizing

Минимум:
- 3 broker (replication factor 3).
- 3 Zookeeper / KRaft controllers.

Для нагрузки:
- +brokers для throughput.
- +partitions для параллелизма.

### 8.2 Retention

- Обычные topics: 7-30 дней.
- Компакция для KV: forever + compaction.
- Мониторить disk usage.

### 8.3 Topics

Названия по конвенции: `<domain>.<entity>.<action>` (например `knp.order.created`).

Создание с `partitions=X replication-factor=3 min.insync.replicas=2`.

### 8.4 Producer settings

Продовые:
```
acks=all
enable.idempotence=true
compression.type=zstd
linger.ms=10
batch.size=32768
```

### 8.5 Consumer settings

Продовые:
```
enable.auto.commit=false
isolation.level=read_committed          # если producer transactional
max.poll.records=100                    # tune под processing time
session.timeout.ms=30000
heartbeat.interval.ms=10000
```

### 8.6 JVM

Kafka на JVM: heap 6 GB (не больше — Kafka использует OS page cache).

```
-Xms6g -Xmx6g -XX:+UseG1GC -XX:MaxGCPauseMillis=20
```

Остальная RAM — OS кэш.

---

## 9. Когда Kafka vs Rabbit vs (SQS / NATS / ...)

- **Kafka** — high throughput, event streaming, replay, exactly-once, log compaction.
- **Rabbit** — task queues, complex routing, priority.
- **SQS** — managed, простые очереди, exactly-once (FIFO).
- **NATS** — легковесный, low-latency messaging.
- **Redis Streams** — простой event log поверх Redis.
- **Pulsar** — Kafka++ (tiered storage, multi-tenancy), сложнее.

Для типового ИСНА scenario — Rabbit (уже используется).

---

## 10. Собесные вопросы

1. **Что даёт Kafka transactions?** — Atomic writes в несколько partitions + atomic offset commit.
2. **Как настроить exactly-once producer?** — `enable.idempotence=true` + `transactional.id` + `initTransactions` + `beginTransaction/commit`.
3. **Что делает read_committed на consumer?** — Видит только commit'нутые транзакции; latency ↑, no duplicates от abort.
4. **Разница idempotent и transactional producer?** — Idempotent: no duplicates в одной partition; transactional: atomic writes в несколько partitions.
5. **Что такое consumer lag?** — Разница latest offset producer'а и current offset consumer'а; растущий = отстаёт.
6. **Как избежать rebalancing storm?** — CooperativeStickyAssignor, увеличить max.poll.interval.ms, стабильные consumers.
7. **Что такое hot partition?** — Одна partition принимает большую часть трафика; sharding ключа лечит.
8. **Как обеспечить no data loss в Kafka?** — acks=all + min.insync.replicas=2 + idempotent + replication.factor=3.
9. **Что такое Kafka Streams?** — Library для stream processing над Kafka; stateful, joins, aggregations, EOS.
10. **Что такое Kafka Connect?** — Framework для integration Kafka с external systems (Debezium CDC, S3 sink, ...).
11. **Что такое Schema Registry?** — Централизованное хранилище schemas (Avro), schema evolution.
12. **Ключевые метрики broker?** — UnderReplicatedPartitions, OfflinePartitions, ActiveControllerCount.
13. **Как обеспечить exactly-once с БД + Kafka?** — Outbox pattern (не Kafka transactions между БД и Kafka).
14. **Alerting на что?** — Consumer lag > threshold, UnderReplicated > 0, broker down, disk full.
15. **Kafka vs Rabbit — когда что?** — Kafka: high throughput, streaming, replay; Rabbit: task queues, complex routing.

---

## Итог

- **Transactions в Kafka** = atomic writes в partitions + atomic offset commit; **только внутри Kafka**.
- **Exactly-once** = idempotent + transactional producer + read_committed consumer.
- **Для БД + Kafka** — outbox pattern (не Kafka tx).
- **Consumer lag** — главная метрика мониторинга.
- **CooperativeStickyAssignor** против rebalancing storm.
- **Sharding ключа** против hot partition.
- **acks=all + idempotence + replication 3 + min.insync 2** = no data loss.
- **JMX → Prometheus → Grafana** для мониторинга.

---

## Итог блока Kafka

- 39 — основы (broker, topic, partition, offset, replication).
- 40 — producer / consumer / offsets / consumer groups / rebalancing.
- 41 — Spring Kafka (KafkaTemplate, @KafkaListener, error handling, DLT).
- 42 — прод-паттерны (transactions, EOS, monitoring, common issues).

Следующий — новые темы. `43-java-basics-primitives-memory.md`.
