# 22. Spring AMQP: RabbitTemplate, @RabbitListener

Как работать с RabbitMQ в Spring Boot. Прод-грейд конфиг.

---

## 1. Зависимости

```gradle
implementation 'org.springframework.boot:spring-boot-starter-amqp'
```

Тянет за собой:
- `spring-rabbit` — Spring AMQP.
- `amqp-client` — низкоуровневая java-библиотека Rabbit.

---

## 2. Конфигурация подключения

```yaml
spring:
  rabbitmq:
    host: rabbit.isna.internal
    port: 5672
    virtual-host: /
    username: knp
    password: ${RABBITMQ_PASSWORD}
    connection-timeout: 10s

    publisher-confirm-type: correlated       # publisher confirms
    publisher-returns: true                   # mandatory + returns

    listener:
      simple:
        acknowledge-mode: manual              # или AUTO/NONE
        prefetch: 20
        concurrency: 5                         # мин. consumers
        max-concurrency: 20                    # макс.
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1s
          multiplier: 2
          max-interval: 10s
        default-requeue-rejected: false        # при exception → DLX
```

### 2.1 Publisher confirm types

- `none` — без подтверждений.
- `simple` — sync, `template.waitForConfirms()`.
- `correlated` — async, callback с корреляцией.

Всегда `correlated` для прод.

---

## 3. RabbitTemplate — публикация

### 3.1 Простая публикация

```java
@Autowired RabbitTemplate rabbit;

// как объект (сериализация Jackson)
rabbit.convertAndSend("knp.events", "fno.submitted",
    new FnoSubmittedEvent(fnoId, regNum));

// как byte[]
rabbit.convertAndSend("knp.events", "fno.submitted", "hello".getBytes());

// со свойствами
rabbit.convertAndSend("knp.events", "fno.submitted", event, msg -> {
    msg.getMessageProperties().setMessageId(UUID.randomUUID().toString());
    msg.getMessageProperties().setExpiration("60000");
    msg.getMessageProperties().setHeader("source", "knp");
    return msg;
});
```

### 3.2 Confirm callback

```java
@PostConstruct
void setup() {
    rabbit.setConfirmCallback((correlation, ack, cause) -> {
        if (!ack) {
            log.error("Publish nack for {}: {}", correlation, cause);
            // retry / alert
        }
    });

    rabbit.setReturnsCallback(returned -> {
        log.warn("Unroutable: {} exchange={} routingKey={}",
            returned.getMessage(), returned.getExchange(), returned.getRoutingKey());
    });
}

rabbit.convertAndSend("knp.events", "fno.submitted", event,
    new CorrelationData(fnoId.toString()));
```

### 3.3 Message converter

По умолчанию Spring использует `SimpleMessageConverter` (Java serialization). Для JSON:

```java
@Bean
Jackson2JsonMessageConverter jsonConverter() {
    return new Jackson2JsonMessageConverter();
}

@Bean
RabbitTemplate rabbitTemplate(ConnectionFactory cf, Jackson2JsonMessageConverter c) {
    RabbitTemplate t = new RabbitTemplate(cf);
    t.setMessageConverter(c);
    return t;
}
```

Теперь `convertAndSend(event)` → JSON body, `Content-Type: application/json`.

### 3.4 Retry для publishing

```yaml
spring.rabbitmq.template:
  retry:
    enabled: true
    initial-interval: 1s
    multiplier: 2
    max-attempts: 3
```

---

## 4. Объявление exchanges / queues / bindings

### 4.1 Через @Bean

```java
@Configuration
class RabbitConfig {

    public static final String EXCHANGE = "knp.events";
    public static final String QUEUE_FNO = "knp.fno.submitted";
    public static final String QUEUE_FNO_DLQ = "knp.fno.submitted.dlq";

    @Bean
    TopicExchange knpEventsExchange() {
        return new TopicExchange(EXCHANGE, true /* durable */, false);
    }

    @Bean
    Queue fnoQueue() {
        return QueueBuilder.durable(QUEUE_FNO)
            .withArgument("x-dead-letter-exchange", "")
            .withArgument("x-dead-letter-routing-key", QUEUE_FNO_DLQ)
            .withArgument("x-message-ttl", 3600000)   // 1 час
            .build();
    }

    @Bean
    Queue fnoDlq() {
        return QueueBuilder.durable(QUEUE_FNO_DLQ).build();
    }

    @Bean
    Binding fnoBinding(Queue fnoQueue, TopicExchange knpEventsExchange) {
        return BindingBuilder.bind(fnoQueue).to(knpEventsExchange).with("fno.submitted");
    }
}
```

Spring на старте автоматически создаст (`RabbitAdmin`) объявленные exchanges/queues/bindings. Если существуют — проверит совместимость.

### 4.2 Через @RabbitListener (queuesToDeclare)

Компактнее:
```java
@RabbitListener(queuesToDeclare = @Queue(name = "knp.fno.submitted", durable = "true"))
void listen(FnoSubmittedEvent e) { ... }
```

Но лучше через `@Bean` — контроль полный.

---

## 5. @RabbitListener — приём

### 5.1 Простой consumer

```java
@Component
class FnoListener {

    @RabbitListener(queues = RabbitConfig.QUEUE_FNO)
    void receive(FnoSubmittedEvent event) {
        log.info("Received {}", event);
        processFno(event);
        // success → auto ack
    }
}
```

Spring:
- Создаёт `SimpleMessageListenerContainer` под капотом.
- Подписывается на очередь.
- При получении → десериализует (по converter) → вызывает метод.
- При success → ack.
- При exception → nack (retry / DLX по настройкам).

### 5.2 С полными аргументами

```java
@RabbitListener(queues = QUEUE_FNO)
void receive(@Payload FnoSubmittedEvent event,
             @Header(AmqpHeaders.MESSAGE_ID) String messageId,
             @Header(name = "source", required = false) String source,
             Channel channel,
             Message message) {
    // полный контроль
}
```

### 5.3 Concurrency

```yaml
spring.rabbitmq.listener.simple:
  concurrency: 5
  max-concurrency: 20
```

Или per listener:
```java
@RabbitListener(queues = QUEUE_FNO, concurrency = "5-20")
```

5 consumers постоянно, до 20 при нагрузке.

### 5.4 Manual ack

```java
@RabbitListener(queues = QUEUE_FNO, ackMode = "MANUAL")
void receive(FnoSubmittedEvent event, Channel ch,
             @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
    try {
        process(event);
        ch.basicAck(tag, false);
    } catch (RetryableException e) {
        ch.basicNack(tag, false, true);   // requeue
    } catch (FatalException e) {
        ch.basicNack(tag, false, false);  // DLX
    }
}
```

Обычно достаточно AUTO ack + правильный error handler.

---

## 6. Retry и recovery

### 6.1 Автоматический retry (Spring AOP)

```yaml
spring.rabbitmq.listener.simple:
  retry:
    enabled: true
    max-attempts: 3
    initial-interval: 1s
    multiplier: 2
    max-interval: 10s
```

При exception:
1. Первая попытка → упало.
2. Ждёт 1 сек → вторая.
3. Ждёт 2 сек → третья.
4. Если и третья упала → recovery.

**Кавет**: retry держит поток consumer'а `Thread.sleep`. Для медленных retry — лучше через retry-topic (см. `21-rabbitmq-delivery-guarantees.md`).

### 6.2 Recoverer

Что делать когда все retries исчерпаны:

```java
@Bean
MessageRecoverer messageRecoverer(RabbitTemplate template) {
    return new RepublishMessageRecoverer(template, "knp.events.dlx", "knp.fno.failed");
}
```

- `RejectAndDontRequeueRecoverer` — просто nack без requeue → в DLX (если настроен на очереди).
- `RepublishMessageRecoverer` — публикует в другой exchange (для собственного DLQ handling).
- Кастомный — свой класс.

---

## 7. ConnectionFactory

Тонкая настройка соединения:

```java
@Bean
CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory f = new CachingConnectionFactory("rabbit.isna.internal");
    f.setUsername("knp");
    f.setPassword("...");
    f.setVirtualHost("/");
    f.setPort(5672);
    f.setConnectionTimeout(10000);
    f.setChannelCacheSize(25);
    f.setPublisherConfirmType(ConfirmType.CORRELATED);
    f.setPublisherReturns(true);
    return f;
}
```

`CachingConnectionFactory` — cache каналов внутри connection. Дефолт в Boot.

Альтернатива — `PooledChannelConnectionFactory` с пулом каналов (для очень высокой нагрузки).

---

## 8. Error handler

Если retry+recovery не хватает — свой error handler:

```java
@Bean
SimpleRabbitListenerContainerFactory containerFactory(ConnectionFactory cf) {
    var factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(cf);
    factory.setErrorHandler(new MyErrorHandler());
    return factory;
}

class MyErrorHandler implements ErrorHandler {
    public void handleError(Throwable t) {
        log.error("Listener failed", t);
        // metric, alert, ...
    }
}
```

---

## 9. Batch listeners

Для очень высокой throughput — обрабатывать сообщения пачками:

```java
@RabbitListener(queues = QUEUE_FNO, containerFactory = "batchContainerFactory")
void batch(List<FnoSubmittedEvent> batch) {
    processBatch(batch);
}

@Bean
SimpleRabbitListenerContainerFactory batchContainerFactory(ConnectionFactory cf) {
    var factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(cf);
    factory.setBatchListener(true);
    factory.setBatchSize(50);
    factory.setConsumerBatchEnabled(true);
    return factory;
}
```

---

## 10. Reply-to (RPC pattern)

RabbitMQ можно использовать для RPC (redko).

```java
// producer
FnoResult result = rabbit.convertSendAndReceiveAsType(
    "knp.exchange", "fno.query", request,
    new ParameterizedTypeReference<FnoResult>() {});
```

Под капотом: producer создаёт temporary queue, шлёт с `reply-to`, ждёт ответа.

Плохо для микросервисов в целом — синхронный RPC поверх async сложнее HTTP. Использовать редко.

---

## 11. Testing

### 11.1 С TestContainers

```java
@Testcontainers
@SpringBootTest
class RabbitIntegrationTest {
    @Container
    static RabbitMQContainer rabbit = new RabbitMQContainer("rabbitmq:3-management");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.rabbitmq.host", rabbit::getHost);
        r.add("spring.rabbitmq.port", rabbit::getAmqpPort);
    }

    @Autowired RabbitTemplate template;

    @Test
    void publishAndReceive() {
        template.convertAndSend("test.exchange", "key", "hello");
        // ...
    }
}
```

### 11.2 Без Docker (mock listener)

Spring AMQP имеет `TestRabbitTemplate` — записывает вызовы для проверки.

---

## 12. Пример полной прод-конфигурации

```java
@Configuration
public class RabbitConfig {

    public static final String EX = "knp.events";
    public static final String Q = "knp.fno.submitted";
    public static final String Q_DLQ = "knp.fno.submitted.dlq";
    public static final String EX_DLX = "knp.dlx";

    @Bean TopicExchange events() { return new TopicExchange(EX, true, false); }
    @Bean DirectExchange dlx() { return new DirectExchange(EX_DLX, true, false); }

    @Bean
    Queue fnoQueue() {
        return QueueBuilder.durable(Q)
            .withArgument("x-dead-letter-exchange", EX_DLX)
            .withArgument("x-dead-letter-routing-key", Q_DLQ)
            .build();
    }

    @Bean Queue fnoDlq() { return QueueBuilder.durable(Q_DLQ).build(); }

    @Bean Binding fnoBind() {
        return BindingBuilder.bind(fnoQueue()).to(events()).with("fno.submitted");
    }
    @Bean Binding fnoDlqBind() {
        return BindingBuilder.bind(fnoDlq()).to(dlx()).with(Q_DLQ);
    }

    @Bean
    Jackson2JsonMessageConverter converter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    RabbitTemplate rabbitTemplate(ConnectionFactory cf, Jackson2JsonMessageConverter c) {
        var t = new RabbitTemplate(cf);
        t.setMessageConverter(c);
        t.setMandatory(true);
        t.setConfirmCallback((corr, ack, cause) -> {
            if (!ack) log.error("Publish nack: {}", cause);
        });
        t.setReturnsCallback(r -> log.warn("Unroutable: {}", r));
        return t;
    }

    @Bean
    SimpleRabbitListenerContainerFactory listenerFactory(
            ConnectionFactory cf, Jackson2JsonMessageConverter c) {
        var f = new SimpleRabbitListenerContainerFactory();
        f.setConnectionFactory(cf);
        f.setMessageConverter(c);
        f.setPrefetchCount(20);
        f.setConcurrentConsumers(5);
        f.setMaxConcurrentConsumers(20);
        f.setDefaultRequeueRejected(false);
        f.setAdviceChain(retryInterceptor());
        return f;
    }

    @Bean
    RetryOperationsInterceptor retryInterceptor() {
        return RetryInterceptorBuilder.stateless()
            .maxAttempts(3)
            .backOffOptions(1000, 2.0, 10000)
            .recoverer(new RejectAndDontRequeueRecoverer())
            .build();
    }
}

@Component
class FnoListener {
    @RabbitListener(queues = RabbitConfig.Q)
    public void handle(FnoSubmittedEvent event,
                        @Header(AmqpHeaders.MESSAGE_ID) String messageId) {
        if (isDuplicate(messageId)) return;
        processFno(event);
        markProcessed(messageId);
    }
}

@Service
class FnoPublisher {
    private final RabbitTemplate rabbit;

    void publish(FnoSubmittedEvent event) {
        rabbit.convertAndSend(RabbitConfig.EX, "fno.submitted", event,
            msg -> {
                msg.getMessageProperties().setMessageId(UUID.randomUUID().toString());
                msg.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                return msg;
            },
            new CorrelationData(event.getFnoId().toString()));
    }
}
```

---

## 13. Собесные вопросы

1. **Что такое RabbitTemplate?** — Основной класс для публикации + утилиты для приёма (RPC).
2. **Как объявить очередь в Spring?** — @Bean с Queue/QueueBuilder; Spring RabbitAdmin создаст при старте.
3. **Что такое @RabbitListener?** — Аннотация на метод-consumer; Spring создаёт listener container.
4. **Что даёт `publisher-confirm-type: correlated`?** — Async подтверждения с корреляцией, callback.
5. **Как настроить retry?** — `spring.rabbitmq.listener.simple.retry.*`.
6. **Что такое message recoverer?** — Что делать после исчерпания retry: RejectAndDontRequeue / RepublishToOther / custom.
7. **CachingConnectionFactory — что кэширует?** — Каналы внутри connection (default в Boot).
8. **Что такое prefetch, где настроить?** — Сколько unacked; `spring.rabbitmq.listener.simple.prefetch`.
9. **Concurrency vs max-concurrency?** — Минимум и максимум количество consumer'ов на очередь.
10. **Что делает `mandatory=true` в RabbitTemplate?** — Возвращает сообщение через returnsCallback если не сматчилось.
11. **Как использовать Jackson в Rabbit?** — Bean `Jackson2JsonMessageConverter`, устанавливать в template + listener factory.
12. **Batch listener — когда?** — Очень высокая throughput, независимая обработка сообщений; `setBatchListener(true)`.

---

## Итог

- **RabbitTemplate** — публикация.
- **@RabbitListener** — приём (SimpleMessageListenerContainer под капотом).
- **Publisher confirms + returns** — надёжность на producer'е.
- **Prefetch + concurrency** — контроль consumer.
- **Retry + Recoverer** — обработка ошибок.
- **Jackson converter** — JSON.
- **@Bean-декларация exchanges/queues/bindings** — рекомендуемый способ.
- **manual/auto ack** — auto достаточно с правильными retry+DLX.

Следующий — `23-rabbitmq-prod-patterns.md`.
