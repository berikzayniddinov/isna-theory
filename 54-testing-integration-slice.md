# 54. Integration и Slice testing в Spring Boot с Testcontainers

## Зачем идти выше unit tests

Unit tests обеспечивают fast feedback plus isolated verification одного класса. Хорошее покрытие unit tests catches many bugs. Но некоторые категории issues fundamentally beyond unit test scope. Как поведёт Spring wire beans при startup? Работает ли JPA mapping правильно с actual PostgreSQL? Правильно ли HTTP endpoints сериализуют/десериализуют JSON? Настроена ли Spring Security correctly? Что происходит при real network calls к real Kafka?

Разница между разработчиком «пишущим unit tests» и «понимающим integration testing» проявляется в incident diagnosis. Первый видит production bug где Spring @Transactional не rollback'ит properly через inheritance. Unit tests все зелёные. Обнаруживается only when hitting real database. Второй знает что integration tests с real database (Testcontainers) catches этот класс issues. Знает что @DataJpaTest slice test даёт real JPA behavior fast. Знает @WebMvcTest для web layer testing без full context. Знает Testcontainers для realistic external systems (PostgreSQL, Kafka, RabbitMQ, Keycloak) в tests. Знает что @Transactional in tests provides automatic rollback maintaining test isolation.

В этом файле разберём Spring Boot integration testing глубоко. Что такое integration test. @SpringBootTest — полный контекст, its variants. Slice tests — @WebMvcTest, @DataJpaTest, @JsonTest, @RestClientTest. MockMvc для HTTP testing без реального сервера. @MockBean для замены beans в context. Testcontainers detailed — postgresql, kafka, rabbit, keycloak. @DynamicPropertySource для dynamic properties. Test profiles. Test data management. Contract testing. Best practices. Реальные caveats.

## Что такое integration test

Тест взаимодействия компонентов. В отличие от unit — не мокаем всё, а тестируем реальные связки.

Виды integration testing. Component test — весь сервис, но external moks. Integration test — часть сервиса (controller plus service plus repo plus БД). System test — весь сервис plus реальные dependencies. Overlapping definitions в industry.

В Spring Boot чаще всего. Slice test — часть Spring контекста (@WebMvcTest, @DataJpaTest). Full context test — @SpringBootTest (весь контекст plus Testcontainers).

Trade-off. Больше real components — реалистичнее но медленнее. Slice tests balance realism с speed для focused testing specific concerns.

## @SpringBootTest — full context

Поднимает весь Spring контекст. Медленно, но реалистично:
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

Что делает. Загружает @SpringBootApplication. Auto-configuration triggers. Создаёт все bean. Готовит для теста.

Время старта. 5-30 секунд первый раз (depending on application size). Cached между тестами того же класса при right configuration.

WebEnvironment опции:
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

MOCK (default) — MockMvc, без реального сервера. Fastest option. HTTP interactions simulated в memory.

RANDOM_PORT — реальный Tomcat на случайном порту. Real HTTP stack. Tests real network behavior.

DEFINED_PORT — на указанном порту. Rare — random обычно предпочтителен.

NONE — не web application. Для non-web functionality.

@Transactional в тестах:
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

Rollback после теста — избегаешь пересечения тестов. Data changes visible during test но rolled back afterwards. Clean state for next test.

Caveat REQUIRES_NEW внутри не откатится с тестовой tx. Отдельная транзакция commits независимо. Другой поток тоже не увидит изменения — свой session.

@DirtiesContext. Если тест портит контекст (изменяет singleton bean state) — заставить пересоздать:
```java
@Test
@DirtiesContext
void breaksThings() { ... }
```

Медленнее (пересоздание контекста). Avoid если можно. Sometimes necessary для tests modifying singleton state.

## Slice tests

Загружают только часть контекста → быстрее.

@WebMvcTest — только web слой:
```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean OrderService svc;

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

Only. Controllers. @ControllerAdvice. Jackson. Validation. Filters. Security (если on classpath).

Не поднимает repositories, services, JPA. Тестирует HTTP-уровень plus JSON serialization plus validation.

MockMvc симулирует HTTP-запросы без реального сервера. Быстро:
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

Rich API для request building plus response assertions. jsonPath для JSON field verification.

@DataJpaTest — только JPA слой:
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

Загружает только. Repositories. EntityManager. DataSource. Liquibase / Flyway.

По умолчанию использует in-memory H2. Для реалистичности — Testcontainers (см. ниже).

@Transactional включён plus rollback автоматически. Fresh state per test.

TestEntityManager — helper для manipulating entities directly в tests. persist, find, flush, clear operations. Cleaner чем using repositories для test setup.

@JsonTest — только сериализация:
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

Для проверки Jackson-схемы. Rarely needed unless heavy JSON customization.

@RestClientTest — только HTTP-клиент:
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

MockRestServiceServer перехватывает исходящие HTTP-запросы, возвращает моки. Testing client-side HTTP integration.

## @MockBean

Заменить бин в контексте на mock:
```java
@SpringBootTest
class OrderIntegrationTest {

    @MockBean PaymentClient paymentClient;    // мок вместо реального
    @Autowired OrderService svc;               // реальный, получит mock

    @Test
    void test() {
        when(paymentClient.charge(any())).thenReturn(...);
        svc.createOrder(...);
    }
}
```

Полезно когда хочешь реальный сервис plus БД, но мокнуть внешний API. Common pattern — real internal, mocked external.

Caveat @MockBean пересоздаёт контекст для каждого теста. Медленно, если много тестов с разными моками. Spring caching optimizes but @MockBean invalidates cache.

Альтернатива @SpyBean — обёртка над реальным. Real bean underlying, methods can be spied without full replacement.

## Testcontainers

Docker контейнеры в тестах. Реальные PostgreSQL, RabbitMQ, Kafka, Redis, Elasticsearch, Keycloak в isolated containers.

Зависимости:
```gradle
testImplementation 'org.testcontainers:testcontainers'
testImplementation 'org.testcontainers:postgresql'
testImplementation 'org.testcontainers:junit-jupiter'
```

Базовое использование:
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

Что происходит. Testcontainers pull image postgres:15 при first run. Запускает контейнер на случайном порту. Spring подхватывает URL через @DynamicPropertySource. Тесты бегут против реального PostgreSQL. Контейнер убивается после.

Real PostgreSQL vs H2. H2 быстрее старт но different dialect. Behaviors различаются. Some Postgres features (JSONB, arrays, specific SQL) не работают в H2. Bugs found в H2 tests не reproduce в prod. Testcontainers обеспечивает realism.

Reuse containers между тестовыми классами:
```java
static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:15")
    .withReuse(true);
```

Плюс ~/.testcontainers.properties:
```
testcontainers.reuse.enable=true
```

Первый запуск — медленно (image pull, startup). Последующие — секунды (reuse existing container). Substantial speed improvement для test suites с many test classes.

Готовые модули для common systems. org.testcontainers:postgresql — PostgreSQL. org.testcontainers:kafka — Kafka. org.testcontainers:rabbitmq — RabbitMQ. org.testcontainers:elasticsearch. com.github.dasniko:testcontainers-keycloak — Keycloak. org.testcontainers:localstack — AWS emulator.

GenericContainer для custom images:
```java
GenericContainer<?> app = new GenericContainer<>("my/image:latest")
    .withExposedPorts(8080)
    .waitingFor(Wait.forHttp("/actuator/health").forPort(8080));
```

Waits ensures container ready before tests run. Various wait strategies — port available, HTTP endpoint responds, log message appears.

Docker Compose support:
```java
@Container
static DockerComposeContainer compose = new DockerComposeContainer(
    new File("src/test/resources/docker-compose.yml"))
    .withExposedService("db_1", 5432)
    .withExposedService("rabbit_1", 5672);
```

Multi-container test environments. Reuse existing compose files.

## @DynamicPropertySource

Динамически регистрирует properties до старта контекста:
```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("kafka.bootstrap-servers", kafka::getBootstrapServers);
}
```

Метод — static, вызывается один раз перед контекстом.

Заменил старый @TestPropertySource(properties = "...") для динамических значений. TestPropertySource requires constant values known at compile time. DynamicPropertySource supports dynamic values (например ports of containers).

Standard mechanism для integrating Testcontainers с Spring configuration.

## Test profiles

Отдельная конфигурация для тестов через application-test.yml:
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:test
  jpa:
    hibernate.ddl-auto: create-drop
```

Activation:
```java
@SpringBootTest
@ActiveProfiles("test")
class MyTest { ... }
```

Или через свойство:
```
mvn test -Dspring.profiles.active=test
```

Different configuration для test environment без polluting main application.yml.

## Test data management

@Sql — SQL скрипты:
```java
@Test
@Sql("/test-data/orders.sql")
void test() { ... }

@Test
@Sql(scripts = "/setup.sql", executionPhase = BEFORE_TEST_METHOD)
@Sql(scripts = "/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
void test2() { ... }
```

Standard Spring approach. SQL executes automatically. Convenient для complex data setup.

Builder pattern или Fixtures:
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
}
```

Плюсы. Читаемо. Легко изменять defaults. Меньше boilerplate. Business language в tests.

@DataSet (DBUnit). XML/YAML fixture для БД. Использовать редко (сложно поддерживать when schema changes).

## Testing Kafka с Testcontainers

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

await() из Awaitility — ждать async condition. Standard для testing async operations. Prevents Thread.sleep anti-pattern.

## Testing RabbitMQ

Similar pattern с RabbitMQContainer:
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
        // await for consumer processing
    }
}
```

Real RabbitMQ behavior in tests. Publisher/consumer flow verified end-to-end.

## Testing Keycloak

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

Real Keycloak instance для authentication testing. JSON exported from production Keycloak (users, clients, roles) как fixture. Full OAuth2/OIDC flow testable.

## Contract testing

Проблема. Producer меняет API — consumer ломается. Как узнать заранее?

Consumer-Driven Contracts. Consumer describes expected response. Producer tested against этот contract. Ensures API stability across services.

Spring Cloud Contract. Producer определяет:
```yaml
description: Should return order
request:
  method: GET
  url: /api/orders/1
response:
  status: 200
  body: {"id": 1, "status": "NEW"}
```

Автоматически генерируется. Test для producer (реальный сервис должен вернуть указанный response). Stub jar для consumer (mock server отвечает так).

Consumer в своих тестах использует stub:
```java
@AutoConfigureStubRunner(ids = "com.example:order-service:+:stubs:8080")
```

Pact — analog. Более популярен вне JVM (JS, Python, .NET).

Consumer описывает expected → генерируется contract → Pact Broker → Producer verify против contract. Different workflow но same concept.

Contract testing catches API breaking changes early в CI pipeline. Prevents runtime failures когда changes deployed.

## Best practices

Категории тестов и распределение.

Unit — 70% (быстро). Slice — 20% (@WebMvcTest, @DataJpaTest). Integration — 5-10% (@SpringBootTest plus Testcontainers). E2E — 5%.

Testovaya пирамида в Spring:
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

Скорость benchmark. Unit — миллисекунды. Slice — секунды. Integration — 10-60 seconds. E2E — минуты.

CI/CD stages. PR pipeline — unit plus slice plus быстрые integration (менее 5 min). Nightly — полный integration plus E2E. On demand — chaos, performance.

Изоляция tests. Каждый тест — свои данные. Не полагаться на порядок. @Transactional plus rollback. Уникальные IDs (UUID.randomUUID()).

Fast feedback. Тесты запускать при каждом commit. IDE — прогонять всё при save. Медленные тесты — только на CI.

## Realistic caveats

Slow tests. @SpringBootTest на каждом тесте — 30 seconds startup. 100 тестов = 50 минут только startup времени.

Fix. Меньше @MockBean (пересоздаёт контекст). @DirtiesContext — избегать. Context caching (по default) — использовать. Разделить 90% unit plus 10% integration.

Flaky tests. Тесты иногда падают, иногда нет. Frustrating и trust destroying.

Причины. Race conditions (async без Awaitility). Полагание на порядок или общее state. Time (использовать Clock injection, не Instant.now()). Внешние ресурсы (сеть).

Fix. Чинить сразу, не игнорировать. Flaky test = broken test. Ignored flaky tests eventually accepted as normal.

БД в тестах. Options plus trade-offs.

H2 in-memory. Быстро, но диалект другой чем PostgreSQL. Bugs в prod не catchable в tests.

Testcontainers PostgreSQL. Реалистично, медленнее. Real behavior. Recommended для integration tests.

Shared PG в CI. Быстро, но общее state — изоляция сложнее. Not recommended для parallel tests.

Правильно — Testcontainers для интеграционных tests.

## Итоги

Integration testing complements unit testing. Different concerns. Both необходимы для comprehensive coverage.

@SpringBootTest — весь контекст, реалистично, медленно. Web environment options MOCK/RANDOM_PORT/DEFINED_PORT/NONE.

Slice tests. @WebMvcTest для web слоя. @DataJpaTest для JPA. @JsonTest для serialization. @RestClientTest для HTTP client. Fast focused testing.

MockMvc для HTTP без реального сервера. Fluent API для requests plus response assertions.

@MockBean заменяет beans в Spring context. Slower через context recreation. @SpyBean для partial spy.

Testcontainers для реалистичных external systems. PostgreSQL, Kafka, RabbitMQ, Keycloak — modules ready. Reuse capability для speed.

@DynamicPropertySource для integrating Testcontainers ports plus URLs в Spring configuration.

Test profiles через @ActiveProfiles. Separate configuration для test environment.

Test data management. @Sql для scripts. Builder patterns для readable fixtures.

Contract testing (Spring Cloud Contract, Pact) для API stability. Consumer-driven contracts prevent breaking changes.

Best practices. Pyramid distribution. Isolated tests. Fast feedback. Fix flaky tests immediately. Testcontainers over H2 для realism.

Дальше — E2E, smoke tests, Selenium для UI, performance testing.
