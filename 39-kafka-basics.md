# 39. Kafka основы: broker, topic, partition, offset

Что такое Kafka, чем отличается от Rabbit, как устроена.

---

## 1. Что такое Kafka

**Apache Kafka** — distributed streaming platform. Не «просто очередь» — это **распределённый лог** (log-based storage).

Идея:
- События (messages) пишутся в **log** (append-only).
- Consumer'ы **читают** log с любой позиции.
- Log **не удаляется после чтения** (в отличие от Rabbit).
- Хранится долго (дни, недели, годы).

Создан в LinkedIn (2011), open-source.

---

## 2. Kafka vs RabbitMQ

Классический вопрос.

| | Kafka | RabbitMQ |
|---|---|---|
| Модель | Log-based (persistent) | Queue-based (transient) |
| Delivery | Consumer pull | Broker push |
| Ordering | В partition — да | В queue — да (1 consumer) |
| Multi-consumer | Consumer groups (одно сообщение — одному в группе) | Fanout exchange / много queues |
| Throughput | Очень высокий (millions/sec) | Высокий (~50k/sec) |
| Retention | По времени/размеру, не завист от чтения | Удаляется после ack |
| Replay | Да (перечитать с любого offset) | Нет (DLQ архив) |
| Priority | Нет | Да |
| Route-логика | Только по partition | Богатая (topic exchange, headers) |
| Транзакции | Да (transactional producer) | Ограниченно |

**Когда Kafka**:
- Event streaming / event sourcing.
- Log agregation.
- Change data capture (CDC).
- Metrics ingest.
- High throughput.
- Требуется replay.

**Когда Rabbit**:
- Task queues.
- RPC-style.
- Priority messages.
- Complex routing.
- Ниже требования к throughput.

---

## 3. Основные концепты

```
┌─── Kafka Cluster ────────────────────────────┐
│                                              │
│  ┌─── Broker 1 ────┐  ┌─── Broker 2 ────┐  │
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

Разберём каждый.

### 3.1 Broker

Один процесс Kafka. Кластер обычно 3-9 broker'ов.

### 3.2 Topic

Логическая категория сообщений. Например: `orders`, `payments`, `user-events`.

Topic делится на **partitions** для параллелизма.

### 3.3 Partition

**Упорядоченный лог сообщений**. Каждое сообщение имеет **offset** — позицию в partition.

```
Partition 0:
[msg0] [msg1] [msg2] [msg3] [msg4] [msg5]
 offset 0-5
```

Гарантии:
- Порядок **внутри partition** — да.
- Между partitions — **нет** гарантии.

Больше partitions = больше параллелизм. Но также overhead.

### 3.4 Offset

Позиция сообщения в partition. Монотонно растёт.

Consumer хранит **свой offset** — до какой позиции прочитал. Начинает с этой позиции при рестарте.

Offset **не удаляется** до retention limit — можно перечитать.

### 3.5 Replica

Партиция реплицируется на **replication factor** брокеров. Стандарт — 3.

Один broker — **leader** partition (обслуживает read/write). Остальные — **followers** (тянут данные с leader).

При падении leader — один из followers становится новым leader.

### 3.6 ISR (In-Sync Replicas)

Replicas, догоняющие leader (лаг < `replica.lag.time.max.ms`).

Только из ISR может быть выбран новый leader.

**`min.insync.replicas`** — сколько ISR должно подтвердить write (при `acks=all`). Обычно 2 (leader + 1 follower).

---

## 4. Producer

Пишет сообщения в topic.

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

### 4.1 Partitioner

Producer решает в какую partition отправить:
- **С ключом** — hash(key) % partitions. Тот же ключ → та же partition → тот же порядок.
- **Без ключа** — round-robin / sticky partitioning.

Правило: **если важен порядок для чего-то (userId, orderId) — использовать этот id как ключ**.

### 4.2 acks

Уровень надёжности:
- **acks=0** — fire-and-forget. Не ждём подтверждения. Максимальная скорость, потери возможны.
- **acks=1** — leader подтвердил. Быстро, но если leader упадёт до replication → потеря.
- **acks=all** — все ISR подтвердили. Максимальная надёжность, медленнее.

Правило: **`acks=all` + `min.insync.replicas=2`** для важных данных.

### 4.3 Batching

Producer собирает сообщения в batch перед отправкой:
- **linger.ms** — сколько ждать до отправки (default 0).
- **batch.size** — max размер batch (default 16 KB).

Увеличить `linger.ms` до 10-100 мс + batch.size = хорошая пропускная способность за счёт небольшой latency.

### 4.4 Compression

- `snappy`, `lz4`, `gzip`, `zstd`.
- `zstd` — лучший баланс.
- Больше compression = меньше сеть, больше CPU.

### 4.5 Idempotent producer

`enable.idempotence=true` — гарантирует что при retry не будет дублей.

Внутри Kafka присваивает каждому producer'у ID + sequence number на каждое сообщение → broker дедуплицирует.

**Обязательно** для критичных данных.

---

## 5. Consumer

Читает сообщения из topic.

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

Подробно про consumer — в следующем файле.

---

## 6. Retention

Kafka хранит сообщения по политикам:

### 6.1 Time-based

```
retention.ms = 604800000   # 7 days (default)
```

Через 7 дней сегменты старше — удаляются.

### 6.2 Size-based

```
retention.bytes = 1073741824   # 1 GB per partition
```

При превышении — старые сегменты удаляются.

### 6.3 Log compaction

**Log compaction** — не удалять по времени, а держать **последнее значение для каждого key**.

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

Использование: KV-store поверх Kafka (event sourcing snapshots, Kafka Streams state).

Настройка: `cleanup.policy=compact` (или `delete,compact` для гибрида).

### 6.4 Правильные значения

Для типового topic с событиями: 7-30 дней retention.

Для аналитики/replay: месяцы.

Для compaction (KV): вечно + периодический compaction.

---

## 7. Segments и файлы

Partition на диске = набор **segments**:

```
/var/lib/kafka/data/orders-0/
├── 00000000000000000000.log       ← сегмент 1 (offset 0 - 999)
├── 00000000000000000000.index
├── 00000000000000000000.timeindex
├── 00000000000000001000.log       ← сегмент 2 (offset 1000+)
├── 00000000000000001000.index
└── 00000000000000001000.timeindex
```

- **.log** — сам данные.
- **.index** — offset → байтовый offset.
- **.timeindex** — timestamp → offset (для поиска по времени).

Новый сегмент открывается по достижении `segment.bytes` (1 GB) или `segment.ms` (7 days).

Retention удаляет сегменты **целиком**, не отдельные сообщения.

---

## 8. ZooKeeper vs KRaft

### 8.1 Раньше: Kafka + Zookeeper

До 2.8 Kafka требовал Zookeeper для:
- Metadata (topics, partitions, replicas).
- Controller election.
- ACLs.
- Consumer offsets (до 0.10 — потом в спец. topic).

Проблемы:
- Ещё одна распределённая система для support.
- Ограничение масштаба (~200k partitions).
- Разная модель consistency.

### 8.2 KRaft (KIP-500)

С Kafka 2.8 (preview) / 3.3+ (production) — **KRaft mode**: внутренний Raft, без Zookeeper.

Плюсы:
- Одна система.
- Больше масштаб (миллионы partitions).
- Быстрее startup, controller failover.

В 3.x — можно выбирать. В 4.x (2024) — только KRaft, Zookeeper удалён.

---

## 9. Controller

Один broker выбран **controller** (координатор):
- Отслеживает состояние других broker'ов.
- Управляет partition leader election.
- Обрабатывает административные операции.

В классической Kafka выбирается через Zookeeper. В KRaft — через внутренний Raft.

---

## 10. Публикация и чтение — happy path

Publish:
```
1. Producer.send(record)
2. Partitioner выбирает partition (по ключу или round-robin)
3. Записывается в batch
4. При заполнении batch или linger.ms → отправка на broker (leader partition)
5. Leader записывает в log (append-only)
6. Followers тянут → записывают
7. acks=all → все ISR подтвердили → producer получает ack
```

Consume:
```
1. Consumer.subscribe("orders")
2. Присоединяется к consumer group
3. Group coordinator назначает partitions
4. Consumer.poll(timeout)
5. Broker отправляет batch сообщений начиная с последнего offset
6. Consumer обрабатывает
7. Consumer.commitSync() → offset сохраняется в __consumer_offsets
```

---

## 11. Deployment

Типичный:
- **Kafka**: 3-5 broker.
- **Zookeeper** (если не KRaft): 3-5 ensemble.
- **Kafka Connect** (для integration): опционально.
- **Schema Registry** (Confluent) — для Avro.
- **ksqlDB / Kafka Streams** — для stream processing.

Мониторинг:
- **JMX metrics** → Prometheus.
- **kafka-consumer-groups.sh** — consumer lag.

---

## 12. Terminология

- **Topic** — категория.
- **Partition** — упорядоченный лог.
- **Offset** — позиция в partition.
- **Broker** — один узел Kafka.
- **Producer** — пишет.
- **Consumer** — читает.
- **Consumer group** — группа consumer'ов, делят partitions.
- **Rebalance** — перераспределение partitions между consumer'ами группы.
- **Replica** — копия partition на другом broker.
- **ISR** — In-Sync Replicas.
- **Leader / Follower** — роли replica.
- **Controller** — координатор кластера.
- **Retention** — политика хранения.
- **Log compaction** — держать только последнее по ключу.

---

## 13. Собесные вопросы

1. **Kafka vs RabbitMQ?** — Kafka log-based (persistent, replay), Rabbit queue-based (transient); Kafka для high throughput + streaming, Rabbit для task queues.
2. **Что такое topic и partition?** — Topic = категория; partition = упорядоченный лог внутри topic для параллелизма.
3. **Что такое offset?** — Позиция сообщения в partition; consumer хранит свою.
4. **Как гарантируется порядок?** — Внутри partition — да; между partitions — нет.
5. **Как обеспечить порядок для конкретного ключа?** — Использовать этот ключ как message key → hash → та же partition.
6. **Что такое ISR?** — In-Sync Replicas, догоняющие leader (лаг < threshold).
7. **Разница acks=0/1/all?** — 0 fire-and-forget; 1 leader ack; all все ISR ack.
8. **Что такое idempotent producer?** — Retry не даёт дублей (внутренний sequence number).
9. **Что такое log compaction?** — Хранить только последнее сообщение для каждого key.
10. **Retention типы?** — Time-based, size-based, log compaction.
11. **Что такое Zookeeper в Kafka?** — Metadata + coordination (устарел, замена — KRaft).
12. **Что делает controller?** — Координирует состояние broker'ов, leader election.
13. **Что такое replication factor?** — Сколько replica у каждой partition (стандарт 3).
14. **Broker упал — что происходит?** — Followers → новый leader выбирается; при падении controller — новый выбирается.
15. **Как обеспечить no data loss?** — acks=all + min.insync.replicas=2 + idempotent producer + replication.factor=3.

---

## Итог

- **Kafka** = distributed **log**, не очередь.
- **Topic** → **Partitions** → **Segments** → messages с **offset**.
- **Broker** — узел; кластер 3-5.
- **Replication factor** = 3, `acks=all`, `min.insync.replicas=2` — надёжный setup.
- **Idempotent producer** для no-duplicates.
- **Consumer group** делит partitions.
- **Retention** по времени/размеру или **log compaction**.
- **KRaft** заменяет Zookeeper (3.3+).

Следующий — `40-kafka-producer-consumer-offsets.md`.
