# 41. Spring Kafka: KafkaTemplate, @KafkaListener, error handling

## Зачем нужна abstraction над raw Kafka client

Использование raw Kafka clients (KafkaProducer plus KafkaConsumer от Apache Kafka) в Spring приложении технически возможно но неэффективно. Каждый developer будет создавать custom bean для KafkaProducer, custom infrastructure для consumer loop с proper polling plus commit strategy, custom error handling с retry logic, custom shutdown hooks для graceful termination. Всё это повторяется в каждом проекте. Плюс раздел ответственности между инфраструктурным кодом и business logic плохой — consumer loop смешивает threading concerns с business rules.

Spring Kafka решает это через high-level abstractions поверх raw clients. KafkaTemplate wraps producer предоставляя idiomatic Spring API. @KafkaListener предоставляет declarative consumer через annotation. ConcurrentMessageListenerContainer управляет threading, polling, commits, rebalancing behind the scenes. DefaultErrorHandler с DeadLetterPublishingRecoverer обеспечивает retry plus DLT patterns. Из этих абстракций формируется production-grade pipeline с minimum custom code.

Разница между разработчиком «использующим Kafka» и «понимающим Spring Kafka» проявляется в детях. Первый пишет @KafkaListener и надеется. Второй знает что concurrency в @KafkaListener фактически создаёт multiple consumers в одном приложении — каждый в отдельном thread — что effectively partitioning must быть sufficient чтобы избежать idle threads. Знает что default ack-mode «batch» коммитит после всего poll batch — meaning single failed message приводит к whole batch reprocessing at least once. Знает что ErrorHandlingDeserializer это wrapper который catches deserialization exceptions и attaches к record instead of blowing up polling entirely. Знает что @TransactionalEventListener AFTER_COMMIT plus KafkaTemplate published из listener гарантирует publish только после DB commit.

В этом файле разберём Spring Kafka с этой deep perspective. Configuration properties и что каждая делает. KafkaTemplate publish semantics — async CompletableFuture callback, sync waiting, producer transactions. @KafkaListener механика — container factory internals, threading model, ack modes deeply. Error handling — DefaultErrorHandler flow, DeadLetterPublishingRecoverer configuration, @RetryableTopic alternative. Serialization pipeline и ErrorHandlingDeserializer защита от poison pills. Producer transactions для atomicity. Complete production example с идемпотентностью на consumer.

## Configuration properties detailed

Spring Kafka configuration через application.yml. Structure отражает разделение producer, consumer, listener concerns:
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
      ack-mode: manual_immediate
      concurrency: 3
      poll-timeout: 500
      type: single
```

bootstrap-servers это entry point cluster. Client discovers full cluster через metadata request к любому из указанных. Обычно 2-3 addresses указывается для resilience.

client-id это identifier producer/consumer instance. Полезно для monitoring — shows в broker JMX metrics которое приложение делает что. Should be unique per instance для different services.

Producer section wraps Kafka producer settings. Spring translates эти properties к properties producer's underlying config. Не все Kafka producer settings имеют dedicated properties — extra через `properties:` map.

Consumer section идентично wraps consumer settings. group-id обязателен — group coordinator uses для partition assignment.

Listener section это Spring Kafka specific — controls behavior ConcurrentMessageListenerContainer wrapping consumer. ack-mode определяет offset commit timing (детально ниже). concurrency количество consumer threads в приложении. poll-timeout maximum wait в poll call.

## KafkaTemplate publish semantics

Основной API для publishing:
```java
@Autowired KafkaTemplate<String, OrderEvent> template;

public void publish(OrderEvent event) {
    template.send("orders", event.getCustomerId(), event);
}
```

Ключ (второй аргумент) определяет partition через hash routing. Same customerId идёт в same partition обеспечивая ordering per customer. Best practice — использовать key that reflects business ordering requirement.

send returns CompletableFuture в Spring Kafka 3+. По умолчанию async — не блокирует calling thread. Result available asynchronously через callback:
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

whenComplete callback вызывается в Kafka producer's I/O thread когда broker responds. Не блокировать callback — этот thread doing all producer I/O work.

Sync waiting иногда нужен но обычно anti-pattern:
```java
SendResult<String, OrderEvent> result = template.send(...).get();
```

Блокирует calling thread до получения confirmation. Использовать только когда really needed — например если следующая operation зависит от successful publish. Обычно async plus callback lifecycle правильный подход.

Producer transactions для exactly-once atomic writes. Configuration:
```yaml
spring.kafka.producer:
  transaction-id-prefix: tx-isna-knp-
```

Использование через Spring @Transactional:
```java
@Transactional("kafkaTransactionManager")
public void publishAtomic(OrderEvent e1, PaymentEvent e2) {
    template.send("orders", e1);
    template.send("payments", e2);
    // если бросит exception — оба aborted
}
```

Transactional producer заранее opens transaction через initTransactions при startup. Sends within @Transactional method wrapped в transaction context — beginTransaction перед first send, commit/abort в конце.

Multiple partition writes atomic — either все или ни одного visible к read_committed consumers. Trade-off performance ценой atomicity guarantees. Обычно overkill для simple scenarios — Outbox pattern часто simpler и flexible.

## @KafkaListener механика

Основная abstraction для consumers:
```java
@Component
class OrderListener {

    @KafkaListener(topics = "orders", groupId = "isna-knp-processor")
    public void handle(OrderEvent event) {
        log.info("Received: {}", event);
        process(event);
    }
}
```

Что происходит under the hood при @KafkaListener annotation processing.

При Spring context startup KafkaListenerAnnotationBeanPostProcessor scans beans looking для @KafkaListener annotations. Для каждой annotated method — создаётся MethodKafkaListenerEndpoint containing method metadata.

Endpoint регистрируется в KafkaListenerEndpointRegistry которая maintains все registered endpoints. Container factory (ConcurrentKafkaListenerContainerFactory) called для создания MessageListenerContainer per endpoint. ConcurrentMessageListenerContainer это main implementation.

ConcurrentMessageListenerContainer at startup — если concurrency=3, создаёт 3 KafkaMessageListenerContainer instances. Каждый содержит один Kafka consumer running в своём thread. Effectively three consumers в one application, all в one group.

Каждый KafkaMessageListenerContainer runs consumer loop internally. Polls Kafka broker. Deserializes records через configured deserializers. Invokes handler method с deserialized value. Handles exceptions through configured error handler. Commits offsets через configured ack-mode.

С полными аргументами method может получить всё context:
```java
@KafkaListener(topics = "orders")
public void handle(@Payload OrderEvent event,
                   @Header(KafkaHeaders.RECEIVED_KEY) String key,
                   @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                   @Header(KafkaHeaders.OFFSET) long offset,
                   @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long timestamp,
                   ConsumerRecord<String, OrderEvent> record,
                   Acknowledgment ack) {
    // full context available
}
```

@Payload marks основное содержимое. @Header extracts specific headers. ConsumerRecord даёт full raw record. Acknowledgment enables manual commit control.

## Ack modes глубже

ack-mode это Spring Kafka concept над raw Kafka offset commits. Multiple strategies available.

RECORD — commit после каждого record. Максимальная safety, минимальная performance (много Kafka commits). Rarely used в production.

BATCH — commit после всего poll batch (default). Standard behavior — process all records from poll, then commit. Single failed message в batch может приводить к whole batch reprocessing.

TIME — commit periodically по timer. Time-based batching offset commits.

COUNT — commit после N records regardless когда poll happened. Count-based batching.

COUNT_TIME — combination COUNT plus TIME (whichever triggers first).

MANUAL — commit only when Acknowledgment.acknowledge() called manually. But actual commit happens at end of poll batch если not yet committed.

MANUAL_IMMEDIATE — commit immediately when Acknowledgment.acknowledge() called synchronously. Different от MANUAL — не waits для end of batch.

Configuration:
```yaml
spring.kafka.listener:
  ack-mode: manual_immediate
```

Usage manual ack:
```java
@KafkaListener(...)
public void handle(OrderEvent event, Acknowledgment ack) {
    process(event);
    ack.acknowledge();
}
```

Choice depends на requirements. MANUAL_IMMEDIATE дает максимальный control но повышает overhead commits. BATCH обычно best balance для standard scenarios. RECORD для extreme reliability requirements.

## Batch listener

Для massive scale processing multiple records в one method call:
```yaml
spring.kafka.listener.type: batch
```

```java
@KafkaListener(...)
public void handleBatch(List<OrderEvent> batch) {
    processBatch(batch);
}
```

Method receives entire poll batch как List. Processing может leverage bulk operations — bulk database inserts, bulk external API calls — reducing overhead per-message.

Trade-off — error handling более сложный. Single failed record в batch — как обрабатывать? Retry whole batch (retry successful ones тоже — duplicates). Skip whole batch (lose successful ones). Partial handling — split successful и failed, more complex logic. DefaultErrorHandler имеет support для batch но requires careful configuration.

## Concurrency и partitions relationship

concurrency в @KafkaListener создаёт multiple consumer instances в одном приложении:
```yaml
spring.kafka.listener.concurrency: 5
```

Или per listener:
```java
@KafkaListener(topics = "orders", concurrency = "5")
```

5 threads с 5 KafkaMessageListenerContainer instances. Each container имеет свой Kafka consumer. Все в one group (group-id from listener config).

Key rule — concurrency should be <= partitions в topic. Больше — extra consumers будут idle потому что каждая partition assigned to exactly one consumer в group. Waste ресурсов.

Practical values. concurrency=1 для development или low-volume topics. concurrency=3-5 для medium throughput. Higher only если partitions counts support.

Consideration — если несколько @KafkaListener в приложении используют same group-id, concurrency counts combined. Two listeners each with concurrency=3 in same group = 6 total consumers. Ensure sufficient partitions.

## Отдельный containerFactory для гибкости

Default single containerFactory сhared для всех @KafkaListener. Для different scenarios требуется different configuration:
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

Named containerFactory injected по name в @KafkaListener. Позволяет different topics/scenarios have different behaviors — different error handling, ack modes, concurrency, deserializers.

## Error handling: DefaultErrorHandler flow

DefaultErrorHandler (Spring Kafka 2.8+) это main error handling mechanism:
```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<String, ?> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));
    var handler = new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3));
    return handler;
}
```

Что делает при exception в listener. Records retried согласно BackOff strategy. FixedBackOff(1000, 3) — 3 retry attempts с 1 second delay между ними. При исчерпании retries — recoverer called с failed record. DeadLetterPublishingRecoverer publishes к DLT topic. После recovery — continue с following records.

BackOff implementations вариируются. FixedBackOff одинаковая delay. ExponentialBackOff увеличивающаяся delay (1s, 2s, 4s, ...). Custom implementations для specific scenarios.

Дополнительная configuration — classify exceptions. Some exceptions retriable (network glitches). Others fatal (validation errors — retry не поможет). handler.addNotRetryableExceptions marks specific types as skip-retry, go directly к recoverer. handler.addRetryableExceptions ограничивает what to retry.

Recovery через DeadLetterPublishingRecoverer publishes к DLT topic. Format — original message plus headers indicating original topic/partition/offset/exception. DLT это regular Kafka topic subject к all standard Kafka behaviors.

## Dead Letter Topic pattern

DLT это отдельный topic для «плохих» сообщений которые consumer не может обработать. Обычно название `<original>.DLT`.

Message format в DLT — original message content plus enrichment headers. `kafka_original-topic`, `kafka_original-partition`, `kafka_original-offset` — где message originally был. `kafka_exception-fqcn` — full class name exception. `kafka_exception-message` — exception message. `kafka_exception-stacktrace` — full stack trace.

DLT consumption обычно manual. Human reviews failed messages, decides — bug fix in consumer, then reprocess. Or malformed message that should be discarded. Или systematic issue requiring escalation.

Some implementations DLT auto-reprocess после certain time — retry hoping transient issue resolved. Advanced patterns — retry topics with delays (0s, 30s, 5min, 30min) forming retry chain перед DLT.

## @RetryableTopic — альтернатива

Spring Kafka также предоставляет @RetryableTopic для declarative retry configuration:
```java
@RetryableTopic(attempts = "3", backoff = @Backoff(delay = 1000, multiplier = 2))
@KafkaListener(topics = "orders")
public void handle(OrderEvent event) { ... }
```

Автоматически создаёт retry-topics с задержками. При failure — message goes to retry topic с timestamp indicating when re-process. Separate consumer of retry topic picks up когда time comes. Через exhaustion attempts — DLT.

Advantage vs DefaultErrorHandler — не блокирует main consumer. Retries happen через separate consumers of retry topics. Main topic continues processing new messages. Long backoffs не hold up throughput.

Disadvantage — more complex topic structure. Multiple retry topics per business topic. Requires understanding retry topic ecosystem.

Choice между DefaultErrorHandler и @RetryableTopic зависит от use case. DefaultErrorHandler proще для quick retry. @RetryableTopic лучше для scenarios с long backoffs или high volume.

## Serialization pipeline и защита от poison pill

Serializers и deserializers key components. Configuration через Spring properties или explicit beans.

String simple:
```yaml
value-serializer: org.apache.kafka.common.serialization.StringSerializer
value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```

JSON через Spring Kafka support:
```yaml
value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

properties:
  spring.json.trusted.packages: kz.gov.kgd.isna.*
  spring.json.value.default.type: kz.gov.kgd.isna.knp.OrderEvent
```

JsonSerializer serializes object via Jackson. Adds __TypeId__ header identifying class. JsonDeserializer reads header, deserializes to identified type.

Critical security setting — spring.json.trusted.packages. Prevents deserialization в arbitrary classes что могло бы enable RCE attacks через crafted messages. Restrictive by default — must explicitly whitelist trusted packages.

Avro через Confluent Schema Registry — compact binary format с schema evolution support:
```yaml
value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
schema.registry.url: http://schema-registry:8081
```

Requires Schema Registry deployment. Schemas registered centrally. Producers include schema ID in messages. Consumers fetch schemas by ID. Enables schema evolution — backwards и forwards compatibility rules enforced.

ErrorHandlingDeserializer защита от poison pill. Message with invalid format — regular deserializer throws exception at poll level — poll cannot return records — infinite retry без progress:
```yaml
value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
properties:
  spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
```

ErrorHandlingDeserializer wraps actual deserializer. При exception во время deserialization — не throws. Instead attaches exception к record. Record delivered к listener with null payload plus exception header. Listener можно проверить и route к DLT explicitly, или default DefaultErrorHandler catches null payload и routes to DLT.

## Producer transactions

Для exactly-once atomic multi-partition writes. Producer transactions coordinate multiple sends в one atomic operation.

Requirements. transactional.id уникальный per producer instance plus stable across restarts. Consumer с isolation.level=read_committed для reading only committed. enable.idempotence=true plus acks=all.

Spring configuration:
```yaml
spring.kafka.producer:
  transaction-id-prefix: tx-isna-knp-
```

Spring generates unique transactional.id per producer using prefix plus counter. Stable across restarts если same prefix used.

Usage через @Transactional с KafkaTransactionManager:
```java
@Transactional("kafkaTransactionManager")
public void publishAtomic(OrderEvent e1, PaymentEvent e2) {
    template.send("orders", e1);
    template.send("payments", e2);
    // exception → abort
}
```

Both sends part of one transaction. Either both visible к read_committed consumers или neither. Atomic guarantees across partitions.

Consumer side с isolation.level=read_committed shows only committed messages. Aborted transactions skipped. Adds latency потому что must wait для transaction completion signal.

Practical limitation — transactions only across Kafka. Не extends к external databases или other systems. Для DB plus Kafka atomicity — Outbox pattern preferred.

## Полная production конфигурация

Собранная воедино:
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
```

Configuration components. Producer factory с idempotence, acks=all, zstd compression. Consumer factory с ErrorHandlingDeserializer wrapping JsonDeserializer, manual commits, trusted packages. Listener container factory с concurrency 3, manual_immediate ack, retry error handler plus DLT recoverer.

## Idempotent consumer pattern

Even с exactly-once producer duplicates возможны при crashes between processing и commit. Consumer idempotency mandatory:
```java
@Component
class OrderListener {

    @Autowired ProcessedRepo processed;

    @KafkaListener(topics = "orders")
    @Transactional
    public void handle(OrderEvent event, Acknowledgment ack,
                       @Header(KafkaHeaders.RECEIVED_KEY) String key) {
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
            throw e;
        } catch (FatalException e) {
            log.error("Fatal — will go to DLT", e);
            throw e;
        }
    }
}
```

Pattern implementation. Check processed table для message ID. If already processed — skip и acknowledge (duplicate). Process message. Record processed ID. Acknowledge. All в single database transaction — либо все либо ни одного (avoids partial state).

Retryable exceptions rethrown — DefaultErrorHandler retries. Fatal exceptions rethrown — DefaultErrorHandler classified as not-retryable, goes to DLT immediately.

## Publishing после DB commit

Common scenario — save entity в DB, then publish event. Naive approach:
```java
@Transactional
public void createOrder(OrderRequest req) {
    Order o = new Order(req);
    repo.save(o);
    template.send("orders", OrderEvent.from(o));
}
```

Problem — publish happens внутри transaction. Если transaction rollbacks — event уже published. Downstream consumers act on nonexistent order.

Right approach — publish только после commit через @TransactionalEventListener:
```java
@Service
class OrderService {
    @Autowired ApplicationEventPublisher events;

    @Transactional
    public void createOrder(OrderRequest req) {
        Order o = new Order(req);
        repo.save(o);
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

@TransactionalEventListener AFTER_COMMIT phase гарантирует что listener invoked только после successful DB commit. Rollback — event not published. Consistency ensured.

Trade-off — если Kafka publish fails after DB commit, DB state inconsistent с Kafka. Order created but event lost. Outbox pattern eliminates this window через persistent outbox table processed by separate publisher — гарантирует eventual publish.

## Testing

EmbeddedKafka для integration testing без external Kafka:
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

EmbeddedKafka starts Kafka in-process в JVM before tests. Fast, no Docker required. Real Kafka behavior для most testing needs.

Testcontainers для even more realistic testing:
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

Real Kafka in Docker container. More authentic testing но slower — container startup adds seconds к test run time. Right choice для critical integration tests.

## Итоги

Spring Kafka предоставляет high-level abstractions над raw Kafka clients. KafkaTemplate для publishing plus @KafkaListener для consuming plus supporting infrastructure.

Configuration через application.yml с producer/consumer/listener sections. Extra properties через nested map для settings without dedicated properties.

KafkaTemplate.send returns CompletableFuture default async. whenComplete callback в producer's I/O thread. Sync waiting doable но anti-pattern usually.

@KafkaListener creates ConcurrentMessageListenerContainer wrapping consumer. Concurrency creates multiple consumer instances в one app — up to partitions count.

Ack modes control offset commit timing. MANUAL_IMMEDIATE для explicit control. BATCH default. Choice affects reliability vs performance trade-off.

Batch listener processes multiple records в one method invocation. Bulk operations leverage. Error handling more complex.

Container factories для different scenarios. Named factories injected через @KafkaListener(containerFactory=...).

DefaultErrorHandler с DeadLetterPublishingRecoverer для retry plus DLT pattern. BackOff strategy determines retry timing. addNotRetryableExceptions для fast fail на fatal errors.

@RetryableTopic alternative через separate retry topics. Не блокирует main consumer. Simpler для complex retry chains.

ErrorHandlingDeserializer защита от poison pill. Wraps actual deserializer. Deserialization exceptions attached к record instead of blowing polling.

JSON serialization через Spring Kafka JsonSerializer/JsonDeserializer. spring.json.trusted.packages critical security setting.

Producer transactions для atomic multi-partition writes. transaction-id-prefix in config. @Transactional("kafkaTransactionManager") для usage.

@TransactionalEventListener AFTER_COMMIT для publishing после DB commit. Гарантирует consistency между DB state и Kafka messages.

Idempotent consumer через processed table или conditional updates. Mandatory для reliable operation.

Testing через EmbeddedKafka (fast) или Testcontainers (realistic).

Дальше — Kafka в production с transactions, exactly-once semantics, monitoring и распространёнными issues.
