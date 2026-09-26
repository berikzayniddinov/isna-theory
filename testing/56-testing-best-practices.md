# 56. Testing best practices: TDD, BDD, mutation, chaos, contract

## Зачем нужны advanced testing practices

Разработчик овладевший unit и integration testing обычно считает — все proven techniques know. Пишет tests, achieves coverage, considers testing skills complete. Production reality приносит nuances. Coverage 80% но critical bug escapes — tests didn't verify edge cases actually catching it. Refactoring painful — tests brittle, breaking on implementation changes. E2E tests flaky — passes locally, fails intermittently в CI. Team habits — some write tests first, some after, no shared understanding practices.

Разница между разработчиком «знающим testing basics» и «понимающим testing practices» проявляется в team-level testing quality. Первый пишет tests как personal preference. Второй знает TDD как workflow discipline — writing tests first shapes design and prevents over-implementation. Знает BDD как communication tool — Given-When-Then bridge между business stakeholders и developers. Знает что coverage lies — mutation testing verifies actual test quality через code mutation. Знает contract testing как efficient way catch inter-service compatibility issues before deployment. Знает chaos engineering как proactive resilience validation. Знает test smells signaling problems requiring attention.

В этом файле разберём advanced practices глубоко. TDD как discipline. BDD и Cucumber. Test doubles углубленно с state vs behavior verification. Test isolation techniques. Flaky tests — diagnosis, fixing, prevention. Test data management strategies. Mutation testing как quality metric. Chaos engineering practices. Contract testing detailed workflow. Shift-left testing philosophy. Real ИСНА test practices. Test smells to avoid.

## TDD: Test-Driven Development

Test-Driven Development — писать тест до кода. Discipline reversing typical order. Not just testing after — driving design через tests.

Red-Green-Refactor cycle:
```
1. RED — написать тест, он падает (код ещё не написан)
2. GREEN — написать минимальный код чтобы тест прошёл
3. REFACTOR — улучшить код (сохраняя зелёный тест)
```

Затем следующий тест — новый цикл. Iteration through cycles builds up feature с coverage guaranteed.

Пример. Задача — add(a, b) method.

Шаг 1 — RED:
```java
@Test
void add() {
    assertThat(calc.add(1, 2)).isEqualTo(3);
}
```

Компилится? Нет — нет метода add. Red state. Test cannot pass because code doesn't exist.

Шаг 2 — GREEN:
```java
public int add(int a, int b) {
    return 3;   // минимум чтобы тест прошёл
}
```

Тест зелёный. Да, return 3 глупо, но это минимум. TDD encourages смallest possible implementation making current test pass. Overengineering prevented.

Шаг 3 — ещё RED:
```java
@Test
void addDifferent() {
    assertThat(calc.add(2, 3)).isEqualTo(5);
}
```

Падает — return 3 doesn't work для all cases.

Шаг 4 — GREEN:
```java
public int add(int a, int b) {
    return a + b;
}
```

Оба теста зелёные. General solution emerged через specific examples.

Шаг 5 — REFACTOR. Cleanup если нужно. Both tests remain green throughout refactoring — safety net enabled.

Плюсы TDD. Code coverage гарантирован — тесты пишутся первыми. Design feedback — если код сложно протестировать, плохо спроектирован. TDD стимулирует хороший design. Regression safety — тесты покрывают всё что делает код. Documentation — тесты показывают как использовать. Быстрая обратная связь — каждый цикл minutes not hours.

Минусы TDD. Кривая обучения — не всем интуитивно. Первые часы медленнее — писать тест plus код. Не работает если требования не ясны — иногда сначала прототип, потом тесты. Не всегда применимо — legacy код без tests, exploratory prototypes.

Realистично. Немногие команды делают строгое TDD. Обычно смесь. TDD для новой бизнес-логики. Post-hoc tests для boilerplate. Exploratory без тестов, потом фиксируешь. Pragmatism over dogma.

## BDD: Behavior-Driven Development

BDD — фокус на поведение системы глазами пользователя, не на реализацию. Extension of TDD emphasizing communication.

Given-When-Then structure:
```
Given некое начальное состояние
When действие
Then ожидаемый результат
```

Пример:
```
Given customer с балансом $100
When он покупает товар за $50
Then баланс становится $50
And появляется notification "Purchase successful"
```

Business language. No implementation details. Anyone can understand — developer, tester, product owner.

Cucumber plus Gherkin. Gherkin — язык описания сценариев. Cucumber — фреймворк для их выполнения.

Feature file order.feature:
```gherkin
Feature: Order creation

  Scenario: Create valid order
    Given customer "berik" is registered
    And catalog has product "Book" with price 100
    When customer creates order for "Book" with quantity 2
    Then order is created with total 200
    And order status is "NEW"

  Scenario Outline: Various quantities
    Given catalog has product "<product>" with price <price>
    When customer creates order for "<product>" with quantity <qty>
    Then order total is <total>

    Examples:
      | product | price | qty | total |
      | Book    | 100   | 2   | 200   |
      | Pen     | 5     | 10  | 50    |
```

Step definitions (Java):
```java
public class OrderSteps {

    @Given("customer {string} is registered")
    public void customer_registered(String name) { ... }

    @When("customer creates order for {string} with quantity {int}")
    public void create_order(String product, int qty) { ... }

    @Then("order total is {int}")
    public void check_total(int expected) {
        assertThat(order.getTotal()).isEqualTo(expected);
    }
}
```

Feature files написаны на natural language. Step definitions map их к executable code. Cucumber runs feature files matching steps.

Плюсы. Читабельно для бизнеса — PM/QA могут читать/писать. Живая документация — always current, executed against actual code. Один язык между dev, QA, PM.

Минусы. Overhead — писать fixtures plus step definitions. Fragile — feature files длинные, легко изменить и сломать. Не для unit — для acceptance/integration только.

Realистично. BDD больше про подход, чем про инструмент. Можно писать Given-When-Then в обычных JUnit тестах:
```java
@Test
@DisplayName("Given customer with balance 100, When purchase 50, Then balance 50")
void purchaseReducesBalance() {
    // Given
    Customer c = new Customer(balance(100));
    // When
    c.purchase(50);
    // Then
    assertThat(c.getBalance()).isEqualTo(50);
}
```

Cucumber полезен когда QA пишут scenarios без Java-кода. Overhead worthwhile когда team includes non-programmers writing tests.

## Test doubles глубже

Из файла 53. Углубление здесь.

Иерархия Meszaros. Dummy — заглушка, значение не используется. Stub — возвращает заданные ответы (state verification). Fake — упрощённая реальная реализация (in-memory Repository). Spy — как stub plus записывает вызовы. Mock — как stub plus verify (behavior verification).

Каждый имеет specific role. Not just terminology — different testing purposes.

State vs Behavior verification. Two schools of thought.

State verification — проверить что state объекта соответствует ожидаемому:
```java
customer.purchase(50);
assertThat(customer.getBalance()).isEqualTo(50);   // state
```

Проверяем что operation resulted в expected state. Outputs matter.

Behavior verification — проверить что нужный метод вызвался:
```java
service.notify(customer);
verify(notifier).send(any(Notification.class));    // behavior
```

Проверяем что operation triggered expected interactions. Side effects matter.

Оба нужны. State — для outputs. Behavior — для side effects. Different scenarios warrant different verification styles.

Классическая vs Мокита-школы. Classic (state-based) — минимум моков, тестируй state. Mockist (behavior-based) — мокай всё, verify.

Разные lagera in industry debate. Реальность — combination. Neither extreme optimal.

Правило — не мокай то что владеешь. Мокай external boundaries. Own code should not need mocking usually — either directly testable или design refactoring needed.

Кавет мокинга. Много моков = тестируешь моки, не код. При рефакторинге ломается всё. Testing implementation через excessive mocking creates brittle test suite.

Правильно — integration tests с real dependencies (Testcontainers) для realistic behavior verification. Не replaces unit tests но complements them.

## Test isolation techniques

Каждый тест независим. Не полагайся на порядок. Не полагайся на state от предыдущих tests.

Плохо:
```java
@Test @Order(1) void createOrder() { orderId = svc.create(...); }
@Test @Order(2) void updateOrder() { svc.update(orderId, ...); }
// зависит от порядка выполнения
```

Хорошо:
```java
@Test void updateOrder() {
    Long orderId = createTestOrder();   // setup внутри теста
    svc.update(orderId, ...);
}
```

Каждый тест self-contained. Order-independence enables parallel execution plus reliable single test execution.

Cleanup mechanisms. @Transactional plus rollback — самый простой (JPA tests). @Sql cleanup — SQL скрипт после test. Unique IDs — избежать конфликта через UUID.randomUUID. Testcontainers reset — новая БД на класс.

Parallel execution. JUnit 5 supports:
```
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
```

Плюс — быстрее. Test suite runs в parallel.

Минусы. Тесты должны быть thread-safe. Никакого shared state. Careful when parallelizing tests с side effects.

## Flaky tests: diagnosis и prevention

Тест иногда падает, иногда — нет. Настоящая боль. Trust destroying — team stops taking test failures seriously if often flaky.

Причины. Race conditions — async без proper wait. Time-dependent — Instant.now() без freeze. Shared state между тестами. Network — external calls. Ordering — полагание на order. Non-deterministic data — random IDs, HashMap iteration. UI timing — Selenium без explicit waits.

Как чинить. Repro — запустить тест 100 раз локально:
```bash
./gradlew test --tests OrderTest.flakyTest --rerun-tasks
```

Или циклом bash. Если 1/100 падает — flaky. Reliable reproduction critical для fixing.

Диагностика. Логи (comprehensive logging). Screenshots (для UI failures). Thread dumps (для concurrency issues).

Fix strategies. Explicit waits (Awaitility) вместо fixed delays. Freeze time (Clock injection) вместо Instant.now(). Isolate data (unique IDs) вместо shared. Mock external (WireMock) вместо real dependencies. Fix race conditions (mutex, CompletableFuture.get()) вместо relying on timing.

Не игнорируй. Flaky test = broken test. Ignoring leads to trust erosion. Real failures dismissed as flakiness. Debugging real bugs impossible.

Retry как workaround только временно:
```java
@Test
@RepeatedTest(3)
void flakyTest() { ... }
```

Или в CI:
```yaml
retries: 2
```

Но параллельно — чинить причину. Retry hides symptoms без addressing root cause.

## Test data management

Стратегии. Каждая с plusами и minuses.

Fixed test data — известные записи в БД (BOOK-1, USER-1). Плюсы — предсказуемо. Минусы — shared state, сложность добавления новых тестов без conflicts.

Create in test — каждый тест создаёт свои данные. Плюсы — изоляция. Минусы — медленнее (setup per test).

Random data — Faker, EasyRandom. Плюсы — unbiased (no coincidental fixed values missing edge cases). Минусы — непредсказуемо, hard to debug when random values matter.

Fixtures / builders — anOrder().build(). Плюсы — читаемо, DRY. Минусы — extra code to maintain.

Faker для generating realistic test data:
```java
Faker faker = new Faker(new Locale("ru"));
String name = faker.name().fullName();
String email = faker.internet().emailAddress();
LocalDate date = faker.date().birthday().toInstant()...;
```

Полезно для performance test data, demo environments, general-purpose realistic data.

Не путать unit и integration data. Unit — минимум (только то что тест проверяет). Integration — реалистичные (для проверки JOIN, indexes). Different quantities appropriate для different test types.

## Mutation testing

Проверка качества тестов. Beyond coverage — actual verification effectiveness.

Идея. Инструмент мутирует твой код (изменяет одну строку). Common mutations. > становится >=. + становится -. true становится false. return x становится return null.

Если тесты всё равно проходят — тесты не ловят эту ошибку — survived mutation — плохо покрытие.

Каждая survived mutation indicates test suite could not detect subtle bug. High kill rate (mutations detected) indicates strong test suite.

Pitest (PIT) — main Java tool:
```gradle
plugins {
    id 'info.solidsoft.pitest' version '1.15.0'
}

pitest {
    junit5PluginVersion = '1.2.1'
    targetClasses = ['com.example.*']
    threads = 4
}
```

Запуск:
```
./gradlew pitest
```

Отчёт в build/reports/pitest — сколько mutations survived.

Metric — mutation score = killed / total. Хорошо — 70%+. Отлично — 90%+. Отражает real качество тестов, не просто coverage.

Cost. Медленно (много вариантов mutations to try). Обычно в nightly build, не на каждый commit. Trade-off worthwhile для critical code paths.

## Chaos engineering

Уже упоминал в файле 52. Здесь подробнее принципы.

Ломать намеренно — проверить что система переживает.

Netflix pioneered concept. Chaos Monkey случайно убивает production под каждый день. Applications must быть fault-tolerant. Continuous chaos verifies resilience.

Уровни chaos. Kill pod — самое простое. Slow network — latency / packet loss injection. DNS failure. CPU 100% на ноде. Disk full. Kill datacenter — extreme.

Инструменты. Chaos Monkey — оригинал (Java). Chaos Mesh — K8s native. Litmus — K8s. Gremlin — SaaS.

GameDay — планированная сессия. Разработчики plus операторы plus бизнес — намеренная поломка — как система реагирует, время до detection, до fix.

Учит команду отвечать на incidents. Training exercise. Real problem scenarios без real consequences.

## Contract testing detailed

Из файла 54. Углубление здесь.

Проблема. Producer меняет API — consumer ломается на проде. E2E ловят но поздно и медленно. Need faster feedback loop.

Consumer-Driven Contracts. Consumer описывает что ожидает — generates contract — Producer verifies.

Pact — популярная framework:

Consumer тест:
```java
@ExtendWith(PactConsumerTestExt.class)
class OrderClientTest {

    @Pact(consumer = "order-service", provider = "payment-service")
    RequestResponsePact pact(PactDslWithProvider builder) {
        return builder
            .given("customer c1 exists")
            .uponReceiving("charge request")
            .path("/api/charge")
            .method("POST")
            .body("{\"customerId\":\"c1\",\"amount\":100}")
            .willRespondWith()
            .status(200)
            .body("{\"transactionId\":\"tx-1\"}")
            .toPact();
    }

    @Test
    @PactTestFor(providerName = "payment-service", pactMethod = "pact")
    void charge(MockServer mockServer) {
        // тест использует mock server, генерирует contract
    }
}
```

Consumer defines expectations. Runs test against mock. Generates contract file describing interactions.

Contract publish в Pact Broker — centralized repository.

Producer verify:
```java
@Provider("payment-service")
@PactBroker(url = "http://pact-broker")
class PaymentProviderTest {

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verify(PactVerificationContext context) {
        context.verifyInteraction();
    }
}
```

Real service запускается. Pact вызывает по contract. Verifies response matches expectations. If provider changed API breaking contract — test fails.

Spring Cloud Contract — Java/Groovy DSL alternative:
```yaml
description: "Get order"
request:
  method: GET
  url: /api/orders/1
response:
  status: 200
  body:
    id: 1
    status: NEW
```

Автогенерируется producer test plus consumer stub. Producer test verifies API. Consumer stub used в consumer's tests.

Плюсы contract testing. Ловим breaking changes до deploy. Быстрее E2E — no full deployment needed. Legacy compatibility — protects existing consumers.

Enable evolution APIs safely. Producer knows exact expectations. Consumer knows exact guarantees.

## Shift-left testing

Идея — тестировать раньше в pipeline.

Раньше:
```
Dev → QA (тестирует) → prod
```

Shift-left:
```
Dev (unit tests + linter + SAST во время написания)
   → CI (integration + contract)
      → deploy
         → post-deploy smoke
```

Плюсы. Bugs ловятся раньше — дешевле fix. QA освобождается для exploratory / user testing. Быстрее feedback.

Обязательно для CI/CD культуры. Fast iteration requires fast feedback.

Testing shifted левее в timeline — from post-development QA к inline during development.

## Правила ИСНА

Из memory. Практика в реальном проекте.

Прод-смок объём. Memory knp-e2e-prod-smoke-scope-rules. Не смокать мутирующие API. Только gap-эндпоинты (что сейчас не покрыто), потом safe GET.

e2e-gate. knp-e2e-gate-check-master-race — гейт можно retry, scope — только изменённый сервис. Optimization avoiding unnecessary reruns.

Feign smoke. knp-e2e-feign-smoke — синтетический смок всех Feign-клиентов. Verifies all internal service integrations working.

Реальная топология. knp-e2e-trigger-topology — как модули триггерят раннер: inline / E2E_LIST / include / E2E_SCOPE. Different modes for different scenarios.

## Test smells

Anti-patterns to avoid.

Excessive setup. Тест начинается со 100 строк setup. Класс делает слишком много. Solution — refactor class into smaller pieces.

Fragile tests. Изменение implementation ломает тесты. Тестируешь implementation, не behavior. Solution — test через public interface, focus on outputs.

Slow tests. Unit тест > 100 мс. Something needs mocking или refactoring. Slow unit tests undermine feedback loop.

Overspecification. Тест проверяет всё. Изменения безобидные — тест падает. Правило — тест проверяет одно поведение.

Assertion roulette. Много assertions без сообщений. Тест падает — не понятно на каком. Fix — одно логическое утверждение per тест или AssertJ assertThat().satisfies() для группировки.

Magic numbers:
```java
assertThat(order.getTotal()).isEqualTo(542.35);   // 542.35 откуда?
```

Правильно:
```java
BigDecimal expected = PRICE.multiply(QTY).add(TAX);
assertThat(order.getTotal()).isEqualTo(expected);
```

Named constants explaining где values come from. Business logic transparent.

Ignored tests. @Disabled без комментария — потерялись, вечно ignored. Или чинить, или удалить. Zombie tests providing no value.

## Метрики качества тестов

Что отслеживать за пределами passes/fails.

Coverage (line, branch). Basic metric.

Mutation score. Real quality indicator.

Test execution time — pipeline должен быть быстрым. Slow tests kill feedback loop.

Flaky rate — процент падений при retry. High rate indicates trust problem.

Test:code ratio — 1:1 to 3:1 обычно. Wildly outside range indicates либо undertested либо overtested.

Все нужно отслеживать. Metrics-driven test improvement.

## Best practices summary

Пирамида тестов. Distribution matters.

AAA / GWT structure. Arrange-Act-Assert или Given-When-Then.

F.I.R.S.T принципы. Fast, Isolated, Repeatable, Self-validating, Timely.

AssertJ plus Mockito стандартный стек Java.

Testcontainers для integration. Real dependencies preferred over mocks для integration testing.

TDD для новой логики, post-hoc для остального. Pragmatic mix.

BDD — если QA пишет тесты. Communication tool.

Contract testing для микросервисов. Faster feedback than E2E.

Explicit waits в UI тестах. Never Thread.sleep.

Fix flaky immediately. Don't tolerate.

Mutation testing периодически. Nightly rather than every commit.

Chaos в staging. Production chaos only когда mature system.

Shift-left — тесты рано в pipeline.

Test independence — auto-rollback / unique data.

Не мокай то что владеешь — только external boundaries.

## Итоги

TDD — Red-Green-Refactor cycle. Discipline shaping design plus guaranteeing coverage. Не dogma но valuable для новой логики.

BDD — Given-When-Then. Communication tool bridging developers plus business stakeholders. Cucumber for non-programmer scenarios.

Test doubles hierarchy. State vs behavior verification. Balanced approach preferred over extremes.

Test isolation через unique data, cleanup, @Transactional rollback. Parallel execution requires thread-safety.

Flaky tests fundamental problem. Diagnose through repeated runs. Fix через explicit waits, deterministic data, mocked external. Never ignore.

Mutation testing verifies test quality beyond coverage. Pitest tool для Java. Nightly builds appropriate.

Chaos engineering proactive resilience validation. Netflix Chaos Monkey inspired. GameDay training exercise.

Contract testing (Pact, Spring Cloud Contract) fast alternative к E2E для API compatibility. Consumer-driven approach.

Shift-left — тесты рано в development pipeline. Fast feedback loop foundation CI/CD culture.

Real ИСНА practices show discipline через specific rules — не смокать мутирующие, gap-endpoints only, scope изменённые services.

Test smells indicate design problems. Excessive setup, fragile tests, magic numbers, ignored tests. Address underlying issues.

Metrics — coverage, mutation score, execution time, flaky rate, test-to-code ratio. Metrics-driven test improvement.

Дальше — Spring Boot configuration comprehensively с PropertySource hierarchy, profiles, ConfigurationProperties, secrets management.
