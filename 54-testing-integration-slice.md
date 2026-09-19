# 54. Integration и Slice тесты

Spring Boot тесты уровня выше unit. `@SpringBootTest`, slice tests, Testcontainers.

---

## 1. Что такое integration test

Тест **взаимодействия компонентов**. В отличие от unit — не мокаем всё, а тестируем реальные связки.

Виды:
- **Component test** — весь сервис, но external моки.
- **Integration test** — часть сервиса (controller + service + repo + БД).
- **System test** — весь сервис + реальные dependencies.

В Spring Boot чаще всего:
- **Slice test** — часть Spring контекста (`@WebMvcTest`, `@DataJpaTest`).
- **Full context test** — `@SpringBootTest` (весь контекст + Testcontainers).

---

## 2. @SpringBootTest — полный контекст

Поднимает весь Spring контекст. Медленно, но реалистично.

```java
@SpringBootTest
class OrderIntegrationTest {

    @Autowired OrderService svc;
    @Autowired OrderRepository repo;

    @Test
    void createOrder_persists() {
        Order o = svc.create(new OrderRequest(...));
        assertThat(repo.findById(o.getId())).isPresent();
    }
}
```

### 2.1 Что делает

1. Загружает `@SpringBootApplication`.
2. Auto-configuration.
3. Создаёт все бины.
4. Готовит для теста.

Время старта: **5-30 секунд** первый раз, кэшируется между тестами того же класса.

### 2.2 WebEnvironment

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class HttpIntegrationTest {

    @LocalServerPort int port;

    @Autowired TestRestTemplate rest;

    @Test
    void getOrder() {
        ResponseEntity<Order> resp = rest.getForEntity(
            "http://localhost:" + port + "/api/orders/1", Order.class);
        assertThat(resp.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

Опции:
- **`MOCK`** (default) — MockMvc, без реального сервера.
- **`RANDOM_PORT`** — реальный Tomcat на случайном порту.
- **`DEFINED_PORT`** — на указанном.
- **`NONE`** — не web.

### 2.3 @Transactional в тестах

```java
@SpringBootTest
@Transactional
class OrderTest {

    @Autowired OrderService svc;

    @Test
    void createOrder() {
        svc.create(new OrderRequest(...));
        // после теста — ROLLBACK автоматически
        // база чистая для следующего теста
    }
}
```

**Rollback после теста** — избегаешь пересечения тестов.

Кавет: REQUIRES_NEW внутри не откатится с тестовой tx. Другой поток тоже не увидит изменения (свой session).

### 2.4 @DirtiesContext

Если тест **портит** контекст (изменяет singleton bean state) — заставить пересоздать:
```java
@Test
@DirtiesContext
void breaksThings() { ... }
```

Медленнее (пересоздание контекста). Избегай если можно.

---

## 3. Slice tests

Загружают **только часть** контекста → быстрее.

### 3.1 @WebMvcTest — только web слой

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean OrderService svc;   // ← service мокается

    @Test
    void getOrder() throws Exception {
        when(svc.get(1L)).thenReturn(new Order(1L, "customer-1"));

        mockMvc.perform(get("/api/orders/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.customerId").value("customer-1"));
    }

    @Test
    void createOrder() throws Exception {
        Order created = new Order(42L, "customer-1");
        when(svc.create(any())).thenReturn(created);

        mockMvc.perform(post("/api/orders")
            .contentType(APPLICATION_JSON)
            .content("{\"customerId\":\"customer-1\",\"amount\":100}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(42));
    }
}
```

Только:
- Controllers.
- `@ControllerAdvice`.
- Jackson.
- Validation.
- Filters.
- Security (если on classpath).

Не поднимает repositories, services, JPA.

Тестирует **HTTP-уровень** + JSON serialization + validation.

### 3.2 MockMvc

Симулирует HTTP-запросы без реального сервера. Быстро.

```java
mockMvc.perform(get("/api/orders/1")
        .param("include", "items")
        .header("X-Trace-Id", "abc"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.total").value(100.0))
    .andExpect(jsonPath("$.items", hasSize(2)))
    .andExpect(header().string("Cache-Control", "no-cache"));

mockMvc.perform(post("/api/orders")
        .contentType(APPLICATION_JSON)
        .content("""
            {
              "customerId": "c1",
              "amount": 100
            }
            """))
    .andExpect(status().isCreated())
    .andExpect(header().exists("Location"));
```

### 3.3 @DataJpaTest — только JPA слой

```java
@DataJpaTest
class OrderRepositoryTest {

    @Autowired OrderRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void findByStatus() {
        em.persist(new Order("c1", NEW));
        em.persist(new Order("c2", CANCELLED));
        em.flush();

        List<Order> newOrders = repo.findByStatus(NEW);

        assertThat(newOrders).hasSize(1);
        assertThat(newOrders.get(0).getCustomerId()).isEqualTo("c1");
    }
}
```

Загружает только:
- Repositories.
- EntityManager.
- DataSource.
- Liquibase / Flyway.

**По умолчанию использует in-memory H2**. Для реалистичности — Testcontainers (см. §5).

`@Transactional` включён + rollback автоматически.

### 3.4 @JsonTest — только сериализация

```java
@JsonTest
class OrderJsonTest {

    @Autowired JacksonTester<Order> json;

    @Test
    void serialize() throws Exception {
        Order o = new Order(1L, "customer-1", NEW);

        assertThat(json.write(o))
            .hasJsonPath("$.id")
            .extractingJsonPathStringValue("$.customerId").isEqualTo("customer-1");
    }

    @Test
    void deserialize() throws Exception {
        String content = "{\"id\":1,\"customerId\":\"customer-1\"}";
        assertThat(json.parse(content).getObject().getId()).isEqualTo(1L);
    }
}
```

Для проверки Jackson-схемы.

### 3.5 @RestClientTest — только HTTP-клиент

```java
@RestClientTest(PaymentClient.class)
class PaymentClientTest {

    @Autowired PaymentClient client;
    @Autowired MockRestServiceServer server;

    @Test
    void charge() {
        server.expect(requestTo("/api/payments"))
            .andExpect(method(HttpMethod.POST))
            .andRespond(withSuccess("{\"status\":\"OK\"}", APPLICATION_JSON));

        PaymentResult result = client.charge(new PaymentRequest(...));

        assertThat(result.getStatus()).isEqualTo("OK");
    }
}
```

MockRestServiceServer перехватывает исходящие HTTP-запросы, возвращает моки.

---

## 4. @MockBean

Заменить бин в контексте на mock:

```java
@SpringBootTest
class OrderIntegrationTest {

    @MockBean PaymentClient paymentClient;    // ← мок вместо реального
    @Autowired OrderService svc;               // ← реальный, получит mock

    @Test
    void test() {
        when(paymentClient.charge(any())).thenReturn(...);
        svc.createOrder(...);
    }
}
```

Полезно когда хочешь реальный сервис + БД, но мокнуть **внешний API**.

**Кавет**: `@MockBean` **пересоздаёт контекст** для каждого теста → медленно, если много тестов с разными моками.

Альтернатива: **`@SpyBean`** — обёртка над реальным.

---

## 5. Testcontainers

Docker контейнеры в тестах. Реальные PostgreSQL, RabbitMQ, Kafka, Redis, Elasticsearch, Keycloak.

### 5.1 Зависимости

```gradle
testImplementation 'org.testcontainers:testcontainers'
testImplementation 'org.testcontainers:postgresql'
testImplementation 'org.testcontainers:junit-jupiter'
```

### 5.2 Базовое использование

```java
@Testcontainers
@SpringBootTest
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("test")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }

    @Autowired OrderRepository repo;

    @Test
    void savesToRealPg() {
        Order o = new Order("c1", NEW);
        repo.save(o);

        assertThat(repo.findAll()).hasSize(1);
    }
}
```

При запуске:
1. Testcontainers pull image `postgres:15`.
2. Запускает контейнер на случайном порту.
3. Spring подхватывает через `@DynamicPropertySource`.
4. Тесты бегут против реального PG.
5. Контейнер убивается после.

### 5.3 Reuse

Между тест-классами — можно переиспользовать контейнер:
```java
static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:15")
    .withReuse(true);
```

Плюс `~/.testcontainers.properties`:
```
testcontainers.reuse.enable=true
```

Первый запуск — медленно; последующие — секунды.

### 5.4 Готовые модули

- `org.testcontainers:postgresql` — PG.
- `org.testcontainers:kafka` — Kafka.
- `org.testcontainers:rabbitmq` — Rabbit.
- `org.testcontainers:elasticsearch`.
- `com.github.dasniko:testcontainers-keycloak` — Keycloak.
- `org.testcontainers:localstack` — AWS emulator.

### 5.5 GenericContainer для custom

```java
GenericContainer<?> app = new GenericContainer<>("my/image:latest")
    .withExposedPorts(8080)
    .waitingFor(Wait.forHttp("/actuator/health").forPort(8080));
```

### 5.6 Docker Compose

```java
@Container
static DockerComposeContainer compose = new DockerComposeContainer(
    new File("src/test/resources/docker-compose.yml"))
    .withExposedService("db_1", 5432)
    .withExposedService("rabbit_1", 5672);
```

---

## 6. @DynamicPropertySource

Динамически регистрирует properties **до** старта контекста:

```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("kafka.bootstrap-servers", kafka::getBootstrapServers);
}
```

Метод — **static**, вызывается **один раз** перед контекстом.

Заменил старый `@TestPropertySource(properties = "...")` для динамических значений.

---

## 7. Test profiles

Отдельная конфигурация для тестов:

`src/test/resources/application-test.yml`:
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:test
  jpa:
    hibernate.ddl-auto: create-drop
```

```java
@SpringBootTest
@ActiveProfiles("test")
class MyTest { ... }
```

Или через свойство:
```
mvn test -Dspring.profiles.active=test
```

---

## 8. Test data management

### 8.1 @Sql — SQL скрипты

```java
@Test
@Sql("/test-data/orders.sql")
void test() { ... }

@Test
@Sql(scripts = "/setup.sql", executionPhase = BEFORE_TEST_METHOD)
@Sql(scripts = "/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
void test2() { ... }
```

### 8.2 Builder pattern / Fixtures

```java
public class OrderFixture {
    public static OrderBuilder anOrder() {
        return new OrderBuilder()
            .withCustomerId("customer-1")
            .withStatus(NEW)
            .withAmount(BigDecimal.valueOf(100));
    }
}

@Test
void test() {
    Order o = anOrder().withAmount(BigDecimal.valueOf(500)).build();
    // ...
}
```

Плюсы:
- Читаемо.
- Легко изменять defaults.
- Меньше boilerplate.

### 8.3 @DataSet (DBUnit)

XML/YAML fixture для БД. Использовать редко (сложно поддерживать).

---

## 9. Testing Kafka с Testcontainers

```java
@Testcontainers
@SpringBootTest
class KafkaIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired KafkaTemplate<String, Order> template;
    @Autowired OrderListener listener;

    @Test
    void publishAndConsume() {
        template.send("orders", new Order("c1", NEW));

        await().atMost(5, SECONDS)
            .until(() -> listener.getReceivedOrders().size() == 1);
    }
}
```

`await()` из **Awaitility** — ждать async condition.

---

## 10. Testing Rabbit

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
        template.convertAndSend("orders", new OrderEvent(...));
        // ...
    }
}
```

---

## 11. Testing Keycloak

```java
@Container
static KeycloakContainer keycloak = new KeycloakContainer("quay.io/keycloak/keycloak:24.0")
    .withRealmImportFile("test-realm.json");

@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    r.add("spring.security.oauth2.resourceserver.jwt.issuer-uri",
        () -> keycloak.getAuthServerUrl() + "/realms/test");
}
```

+ JSON exported из production Keycloak (users, clients, roles).

---

## 12. Contract testing

### 12.1 Проблема

Producer меняет API → consumer ломается. Как узнать заранее?

### 12.2 Consumer-Driven Contracts

Consumer описывает какой response ожидает. Producer тестируется против этого contract.

### 12.3 Spring Cloud Contract

Producer определяет:
```yaml
description: Should return order
request:
  method: GET
  url: /api/orders/1
response:
  status: 200
  body: {"id": 1, "status": "NEW"}
```

Автоматически генерируется:
- Тест для producer (реальный сервис должен вернуть указанный response).
- Stub jar для consumer (mock server отвечает так).

Consumer в своих тестах использует stub:
```java
@AutoConfigureStubRunner(ids = "com.example:order-service:+:stubs:8080")
```

### 12.4 Pact

Аналог, более популярен вне JVM (JS, Python).

Consumer описывает expected → генерируется contract → Pact Broker → Producer verify против contract.

---

## 13. Best practices

### 13.1 Категории

- **Unit** — 70% (быстро).
- **Slice** — 20% (@WebMvcTest, @DataJpaTest).
- **Integration** — 5-10% (@SpringBootTest + Testcontainers).
- **E2E** — 5%.

### 13.2 Тестовая пирамида в Spring

```
                 /  \      E2E (Selenium, real deployment)
                /____\
               /      \    Integration (@SpringBootTest + Testcontainers)
              /________\
             /          \  Slice (@WebMvcTest, @DataJpaTest)
            /____________\
           /              \ Unit (JUnit + Mockito)
          /________________\
```

### 13.3 Скорость

- Unit — миллисекунды.
- Slice — секунды.
- Integration — 10-60 секунд.
- E2E — минуты.

### 13.4 CI/CD

- **PR pipeline** — unit + slice + быстрые integration (< 5 мин).
- **Nightly** — полный integration + E2E.
- **On demand** — chaos, performance.

### 13.5 Изоляция

- Каждый тест — свои данные.
- Не полагаться на порядок.
- `@Transactional` + rollback.
- Уникальные IDs (`UUID.randomUUID()`).

### 13.6 Быстрая обратная связь

**Fail fast**: тесты запускать при каждом commit. IDE — прогонять всё при save. Медленные тесты — только на CI.

---

## 14. Реальные кейсы

### 14.1 Slow tests

`@SpringBootTest` на каждом тесте — 30 секунд старт. 100 тестов = 50 минут только startup.

**Fix**:
- Меньше `@MockBean` (пересоздаёт контекст).
- `@DirtiesContext` — избегать.
- Context caching (по default) — использовать.
- Разделить: 90% unit + 10% integration.

### 14.2 Flaky tests

Тесты иногда падают, иногда нет.

Причины:
- Race conditions (async без Awaitility).
- Полагание на порядок / общее state.
- Time (использовать `Clock` инъекцию, не `Instant.now()`).
- Внешние ресурсы (сеть).

**Fix**: чинить сразу, не игнорировать. Flaky test = broken test.

### 14.3 БД в тестах

- **H2 in-memory** — быстро, но диалект другой чем PG → баги в проде.
- **Testcontainers PG** — реалистично, медленнее.
- **Shared PG в CI** — быстро, но общее state → изоляция сложнее.

Правильно — **Testcontainers** для интеграционных.

---

## 15. Собесные вопросы

1. **Что такое integration test?** — Тест взаимодействия компонентов (не мокаем всё).
2. **@SpringBootTest — что делает?** — Загружает весь Spring контекст.
3. **@WebMvcTest — когда?** — Только web слой (controllers, JSON, validation), сервис мокается.
4. **@DataJpaTest — когда?** — Только repository + JPA, in-memory H2 по default.
5. **Что такое MockMvc?** — Симулятор HTTP запросов без реального сервера; быстрее чем TestRestTemplate.
6. **@MockBean vs @Mock?** — @MockBean заменяет bean в Spring контексте; @Mock — обычный Mockito mock без контекста.
7. **Что такое Testcontainers?** — Docker контейнеры (PG, Kafka, ...) в тестах для реалистичности.
8. **@DynamicPropertySource — зачем?** — Регистрирует properties (URLs Testcontainers) до старта контекста.
9. **@Transactional в тестах?** — Auto-rollback после теста; изоляция.
10. **Что такое @DirtiesContext?** — Пересоздать контекст (когда тест портит state); медленно, избегай.
11. **Contract testing?** — Consumer описывает expected; producer тестируется против contract (Pact, Spring Cloud Contract).
12. **@RestClientTest — что?** — Slice для HTTP-клиента; MockRestServiceServer.
13. **Awaitility — зачем?** — Ждать async condition в тестах (`await().until(...)`).
14. **Testcontainers reuse — как?** — `.withReuse(true)` + `~/.testcontainers.properties`; переиспользуется между тестами.
15. **In-memory H2 vs Testcontainers PG — выбор?** — H2 быстро, но диалект другой; PG реалистично, медленнее. Testcontainers для integration.

---

## Итог

- **@SpringBootTest** — весь контекст, реалистично, медленно.
- **Slice tests** (`@WebMvcTest`, `@DataJpaTest`) — быстро, часть контекста.
- **MockMvc** для HTTP без реального сервера.
- **@MockBean** для замены bean в контексте.
- **Testcontainers** для реалистичных БД, Kafka, Rabbit, Keycloak.
- **@Transactional** + auto-rollback = чистая БД.
- **Contract testing** (Pact / Spring Cloud Contract) для API stability.
- **Awaitility** для async.

Следующий — `55-testing-e2e-smoke-selenium.md`.
