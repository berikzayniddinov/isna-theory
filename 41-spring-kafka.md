# 41. Spring Kafka: @KafkaListener, KafkaTemplate

Как работать с Kafka в Spring Boot. Прод-грейд конфиг.

---

## 1. Зависимости

```gradle
implementation 'org.springframework.kafka:spring-kafka'
```

Автоматически подключается через Spring Boot starter если Kafka на classpath.

---

## 2. Конфигурация

```yaml
spring:
  kafka:
    bootstrap-servers: kafka:9092
    client-id: isna-knp

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
        linger.ms: 10
        compression.type: zstd

    consumer:
      group-id: isna-knp-processor
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: kz.gov.kgd.isna.*
        max.poll.records: 100
        max.poll.interval.ms: 300000

    listener:
      ack-mode: manual_immediate       # manual/manual_immediate/batch/record/time/count
      concurrency: 3                    # threads per @KafkaListener
      poll-timeout: 500
      type: single                      # single / batch
```

---

## 3. KafkaTemplate — публикация

### 3.1 Простая публикация

```java
@Autowired KafkaTemplate<String, OrderEvent> template;

public void publish(OrderEvent event) {
    template.send("orders", event.getCustomerId(), event);
}
```

Ключ — второй аргумент. Определит partition.

### 3.2 С callback

```java
public void publish(OrderEvent event) {
    template.send("orders", event.getCustomerId(), event)
        .whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Send failed", ex);
            } else {
                log.debug("Sent to partition={} offset={}",
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            }
        });
}
```

Async (в Spring Kafka 3+ — `CompletableFuture`). Не блокирует.

### 3.3 Sync (не рекомендуется)

```java
SendResult<String, OrderEvent> result = template.send(...).get();   // блокирует
```

Только если реально нужно подтверждение перед следующим действием.

### 3.4 Producer transactions

Для exactly-once:
```yaml
spring.kafka.producer:
  transaction-id-prefix: tx-isna-knp-
```

```java
@Transactional("kafkaTransactionManager")
public void publishAtomic(OrderEvent e1, PaymentEvent e2) {
    template.send("orders", e1);
    template.send("payments", e2);
    // если бросит — оба aborted
}
```

---

## 4. @KafkaListener — приём

### 4.1 Простой

```java
@Component
class OrderListener {

    @KafkaListener(topics = "orders", groupId = "isna-knp-processor")
    public void handle(OrderEvent event) {
        log.info("Received: {}", event);
        process(event);
        // success → auto ack
    }
}
```

Spring:
- Создаёт `ConcurrentMessageListenerContainer`.
- Подписывается на topic.
- Poll'ит, десериализует, вызывает.
- Auto ack при success; retry / DLT при exception (по настройкам).

### 4.2 С полными аргументами

```java
@KafkaListener(topics = "orders")
public void handle(@Payload OrderEvent event,
                   @Header(KafkaHeaders.RECEIVED_KEY) String key,
                   @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                   @Header(KafkaHeaders.OFFSET) long offset,
                   @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long timestamp,
                   ConsumerRecord<String, OrderEvent> record,
                   Acknowledgment ack) {

    log.info("Received key={} part={} offset={}", key, partition, offset);
    try {
        process(event);
        ack.acknowledge();
    } catch (Exception e) {
        // handling
    }
}
```

### 4.3 Manual ack

Требует `ack-mode: manual` или `manual_immediate`:
```yaml
spring.kafka.listener:
  ack-mode: manual_immediate
```

```java
@KafkaListener(...)
public void handle(OrderEvent event, Acknowledgment ack) {
    process(event);
    ack.acknowledge();   // явный commit
}
```

Разница `manual` vs `manual_immediate`:
- `manual` — commit в конце batch.
- `manual_immediate` — commit сразу после acknowledge.

### 4.4 Batch listener

```yaml
spring.kafka.listener.type: batch
```

```java
@KafkaListener(...)
public void handleBatch(List<OrderEvent> batch) {
    processBatch(batch);
}
```

Быстрее для массовых.

### 4.5 Concurrency

```yaml
spring.kafka.listener.concurrency: 5
```

Или per listener:
```java
@KafkaListener(topics = "orders", concurrency = "5")
```

5 threads / consumer instances в одном приложении.

Правило: `concurrency <= partitions в topic`. Больше — часть потоков простаивает.

### 4.6 Отдельный containerFactory

Для гибкой настройки нескольких listener'ов:

```java
@Bean
ConcurrentKafkaListenerContainerFactory<String, OrderEvent> orderKafkaFactory(
        ConsumerFactory<String, OrderEvent> cf) {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, OrderEvent>();
    factory.setConsumerFactory(cf);
    factory.setConcurrency(5);
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
    factory.setCommonErrorHandler(errorHandler());
    return factory;
}

@KafkaListener(topics = "orders", containerFactory = "orderKafkaFactory")
public void handle(OrderEvent event) { ... }
```

---

## 5. Error handling

### 5.1 DefaultErrorHandler (Spring Kafka 2.8+)

```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<String, ?> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));
    var handler = new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3));
    return handler;
}
```

Что делает:
1. При exception в listener — retry 3 раза с задержкой 1 сек.
2. Если все retry упали → отправить в DLT (Dead Letter Topic).
3. Continue с следующего сообщения.

### 5.2 Dead Letter Topic (DLT)

Отдельный topic для «плохих» сообщений. Обычно название `<original>.DLT`.

Формат: original message + headers с оригинальным topic/partition/offset/exception.

Consumer'ы DLT — обычно ручной разбор + повторная отправка после fix.

### 5.3 RetryTopicConfigurer (более гибкое)

```java
@RetryableTopic(attempts = "3", backoff = @Backoff(delay = 1000, multiplier = 2))
@KafkaListener(topics = "orders")
public void handle(OrderEvent event) { ... }
```

Автоматически создаёт retry-topics с задержками (`orders-retry-0`, `orders-retry-1`, `orders-dlt`).

Не блокирует основной consumer.

---

## 6. Serializer / Deserializer

### 6.1 String

Default простой:
```yaml
value-serializer: org.apache.kafka.common.serialization.StringSerializer
value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```

### 6.2 JSON (Spring Kafka)

```yaml
value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

properties:
  spring.json.trusted.packages: kz.gov.kgd.isna.*
  spring.json.value.default.type: kz.gov.kgd.isna.knp.OrderEvent
```

Сериализует объект в JSON. Deserializer читает `__TypeId__` header для определения типа.

**`spring.json.trusted.packages`** — safety: не десериализовать в любой класс (RCE защита).

### 6.3 Avro (Confluent)

Schema-based, компактный, evolution-friendly.

```yaml
value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
schema.registry.url: http://schema-registry:8081
```

Требует Schema Registry (Confluent).

### 6.4 Protobuf

Аналогично Avro, через Protobuf Serializer.

### 6.5 ErrorHandlingDeserializer

Poison pill (некорректный JSON) → deserializer throws → poll break.

Fix: обёртка:
```yaml
value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
properties:
  spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
```

При ошибке — сообщение помечается, попадает в error handler → DLT.

---

## 7. Полная прод-конфигурация

```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Bean
    ProducerFactory<String, Object> producerFactory(KafkaProperties props) {
        var config = new HashMap<String, Object>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, props.getBootstrapServers());
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.LINGER_MS_CONFIG, 10);
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "zstd");
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> pf) {
        return new KafkaTemplate<>(pf);
    }

    @Bean
    ConsumerFactory<String, Object> consumerFactory(KafkaProperties props) {
        var config = new HashMap<String, Object>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, props.getBootstrapServers());
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "isna-knp-processor");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        config.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "kz.gov.kgd.isna.*");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> cf,
            KafkaTemplate<String, Object> template) {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, Object>();
        factory.setConsumerFactory(cf);
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        var recoverer = new DeadLetterPublishingRecoverer(template);
        var errorHandler = new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3));
        factory.setCommonErrorHandler(errorHandler);

        return factory;
    }
}

@Component
class OrderListener {

    @KafkaListener(topics = "orders")
    public void handle(OrderEvent event, Acknowledgment ack,
                       @Header(KafkaHeaders.RECEIVED_KEY) String key) {
        try {
            processIdempotent(event, key);
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Failed to process order {}", key, e);
            throw e;  // → error handler → retry → DLT
        }
    }
}

@Service
class OrderPublisher {
    private final KafkaTemplate<String, OrderEvent> template;

    public void publish(OrderEvent event) {
        template.send("orders", event.getCustomerId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) log.error("Publish failed", ex);
            });
    }
}
```

---

## 8. Testing

### 8.1 EmbeddedKafka

```java
@SpringBootTest
@EmbeddedKafka(partitions = 3, topics = {"orders"})
class KafkaIntegrationTest {

    @Autowired KafkaTemplate<String, Object> template;
    @Autowired OrderListener listener;

    @Test
    void publishAndConsume() {
        template.send("orders", new OrderEvent(...));
        await().atMost(5, SECONDS).until(() -> listener.getReceived().size() == 1);
    }
}
```

Быстро, без Docker. Но не всё покрывает (real broker specifics).

### 8.2 Testcontainers

```java
@Testcontainers
@SpringBootTest
class KafkaIntegrationTest {
    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
}
```

Реалистичнее, медленнее.

---

## 9. Пример полного flow

Publisher:
```java
@Service
@Slf4j
class OrderService {

    @Autowired KafkaTemplate<String, OrderEvent> template;

    @Transactional
    public void createOrder(OrderRequest req) {
        Order o = new Order(req);
        repo.save(o);
        // публикация после commit — @TransactionalEventListener
        events.publishEvent(new OrderCreatedEvent(o));
    }
}

@Component
class OrderEventPublisher {

    @Autowired KafkaTemplate<String, OrderEvent> template;

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void publish(OrderCreatedEvent e) {
        template.send("orders", e.getOrder().getCustomerId(),
            OrderEvent.from(e.getOrder()));
    }
}
```

Consumer:
```java
@Component
@Slf4j
class OrderProcessor {

    @Autowired ProcessedRepo processed;

    @KafkaListener(topics = "orders")
    @Transactional
    public void handle(OrderEvent event, Acknowledgment ack,
                       @Header(KafkaHeaders.RECEIVED_KEY) String key,
                       @Header(KafkaHeaders.OFFSET) long offset) {
        // idempotency check
        if (processed.existsById(event.getOrderId())) {
            log.debug("Skip duplicate {}", event.getOrderId());
            ack.acknowledge();
            return;
        }

        try {
            doProcess(event);
            processed.save(new Processed(event.getOrderId()));
            ack.acknowledge();
        } catch (RetryableException e) {
            log.warn("Retry", e);
            throw e;  // error handler → retry
        } catch (FatalException e) {
            log.error("Fatal — will go to DLT", e);
            throw e;
        }
    }
}
```

---

## 10. Собесные вопросы

1. **Как публиковать в Spring Kafka?** — `KafkaTemplate.send(topic, key, value)`; async с CompletableFuture callback.
2. **Как подписаться?** — `@KafkaListener(topics = "...")` на методе.
3. **Что такое ack-mode?** — Как Kafka commit'ит offsets: auto, manual, batch, record, time, count.
4. **Как обработать batch?** — `spring.kafka.listener.type: batch` + `List<T>` параметр.
5. **Concurrency в @KafkaListener?** — Threads per listener; <= partitions.
6. **Что такое DLT?** — Dead Letter Topic для необработанных сообщений; DefaultErrorHandler + DeadLetterPublishingRecoverer.
7. **Как настроить retry?** — DefaultErrorHandler с BackOff или @RetryableTopic.
8. **Разница @RetryableTopic и DefaultErrorHandler?** — @RetryableTopic делает retry в отдельных topics (не блокирует consumer); DefaultErrorHandler retry в том же consumer.
9. **Как избежать poison pill?** — ErrorHandlingDeserializer оборачивает JsonDeserializer.
10. **Как публиковать после commit tx?** — @TransactionalEventListener(AFTER_COMMIT).
11. **Producer transactions в Spring Kafka?** — `transaction-id-prefix` + @Transactional("kafkaTransactionManager").
12. **Что делает Acknowledgment.acknowledge?** — Commit offset (при manual ack mode).
13. **manual vs manual_immediate?** — Manual в конце batch; immediate сразу.
14. **Как тестировать Kafka?** — EmbeddedKafka (быстро) или Testcontainers (реалистично).

---

## Итог

- **KafkaTemplate** для publish, **@KafkaListener** для consume.
- **acks=all + idempotence** обязательно.
- **Manual ack** для контроля.
- **DefaultErrorHandler + DLT** для error handling.
- **ErrorHandlingDeserializer** от poison pill.
- **@TransactionalEventListener(AFTER_COMMIT)** для publish после DB commit.
- **Concurrency** <= partitions.
- **Идемпотентный consumer** обязательно.

Следующий — `42-kafka-prod.md`.
