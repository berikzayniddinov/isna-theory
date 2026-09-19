# 56. Testing best practices: TDD, BDD, mutation, chaos

Принципы и подходы к тестированию.

---

## 1. TDD — Test-Driven Development

**Test-Driven Development**: писать тест **до** кода.

### 1.1 Red-Green-Refactor cycle

```
1. RED — написать тест, он падает (код ещё не написан)
2. GREEN — написать минимальный код чтобы тест прошёл
3. REFACTOR — улучшить код (сохраняя зелёный тест)
```

Затем следующий тест — новый цикл.

### 1.2 Пример

Задача: `add(a, b)`.

**Шаг 1 — RED**:
```java
@Test
void add() {
    assertThat(calc.add(1, 2)).isEqualTo(3);
}
```
Компилится? Нет — нет метода `add`. → Red.

**Шаг 2 — GREEN**:
```java
public int add(int a, int b) {
    return 3;   // минимум чтобы тест прошёл
}
```
Тест зелёный. Да, `return 3` глупо, но это минимум.

**Шаг 3 — ещё RED**:
```java
@Test
void addNegative() {
    assertThat(calc.add(2, 3)).isEqualTo(5);
}
```
Падает.

**Шаг 4 — GREEN**:
```java
public int add(int a, int b) {
    return a + b;
}
```
Оба теста зелёные.

**Шаг 5 — REFACTOR**: чистка (если нужно).

### 1.3 Плюсы

- **Code coverage** гарантирован (тесты пишутся первыми).
- **Design feedback** — если код сложно протестировать → плохо спроектирован. TDD стимулирует хороший design.
- **Regression safety** — тесты покрывают всё, что делает код.
- **Documentation** — тесты показывают как использовать.
- **Быстрая обратная связь**.

### 1.4 Минусы

- **Кривая обучения** — не всем интуитивно.
- **Первые часы медленнее** — писать тест + код.
- **Не работает если требования не ясны** (иногда сначала прототип, потом тесты).
- **Не всегда применимо** — legacy код, exploratory prototypes.

### 1.5 Realистично

Немногие команды делают **строгое TDD**. Обычно смесь:
- TDD для новой бизнес-логики.
- Post-hoc tests для боилерплейта.
- Exploratory без тестов, потом фиксируешь.

---

## 2. BDD — Behavior-Driven Development

**BDD** — фокус на **поведение системы** глазами пользователя, не на реализацию.

### 2.1 Given-When-Then

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

### 2.2 Cucumber / Gherkin

**Gherkin** — язык описания сценариев. **Cucumber** — фреймворк для их выполнения.

Feature file `order.feature`:
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

### 2.3 Плюсы

- **Читабельно для бизнеса** — PM/QA могут читать/писать.
- **Живая документация**.
- **Один язык** между dev, QA, PM.

### 2.4 Минусы

- **Overhead** — писать fixtures + step definitions.
- **Fragile** — feature files длинные, легко изменить и сломать.
- **Не для unit** — для acceptance/integration.

### 2.5 Realистично

BDD **больше про подход**, чем про инструмент. Можно писать Given-When-Then в обычных JUnit тестах:
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

Cucumber полезен когда QA пишут scenarios без Java-кода.

---

## 3. Test doubles — детально

Из `53-testing-unit.md`, углубление.

### 3.1 Иерархия (Meszaros)

- **Dummy** — заглушка, значение не используется.
- **Stub** — возвращает заданные ответы (state verification).
- **Fake** — упрощённая реальная реализация (in-memory Repository).
- **Spy** — как stub + записывает вызовы.
- **Mock** — как stub + verify (behavior verification).

### 3.2 State vs Behavior verification

**State verification** — проверить что state объекта соответствует ожидаемому:
```java
customer.purchase(50);
assertThat(customer.getBalance()).isEqualTo(50);   // state
```

**Behavior verification** — проверить что нужный метод вызвался:
```java
service.notify(customer);
verify(notifier).send(any(Notification.class));    // behavior
```

Оба нужны. State — для outputs, behavior — для side effects.

### 3.3 Классическая vs Мокита-школы

- **Classic (state-based)** — минимум моков; тестируй state.
- **Mockist (behavior-based)** — мокай всё; verify.

Разные lagera. Реальность — **combination**.

Правило: **не мокай то что владеешь**. Мокай external boundaries.

### 3.4 Кавет мокинга

Много моков = тестируешь моки, не код. При рефакторинге ломается всё.

Правильно — **integration tests с real dependencies** (Testcontainers).

---

## 4. Test isolation

### 4.1 Каждый тест независим

Не полагайся на порядок. Не полагайся на state от предыдущих.

Плохо:
```java
@Test @Order(1) void createOrder() { orderId = svc.create(...); }
@Test @Order(2) void updateOrder() { svc.update(orderId, ...); }   // ← зависит от порядка
```

Хорошо:
```java
@Test void updateOrder() {
    Long orderId = createTestOrder();   // setup внутри теста
    svc.update(orderId, ...);
}
```

### 4.2 Cleanup

- **@Transactional + rollback** — самый простой (JPA).
- **@Sql cleanup** — SQL скрипт после.
- **Unique IDs** — избежать конфликта (`UUID.randomUUID()`).
- **Testcontainers reset** — новая БД на класс.

### 4.3 Parallel execution

JUnit 5 поддерживает:
```
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
```

Плюс: быстрее.
Минусы: тесты должны быть **thread-safe** (никакого shared state).

---

## 5. Flaky tests

Тест иногда падает, иногда — нет. **Настоящая боль**.

### 5.1 Причины

- **Race conditions** — async без proper wait.
- **Time-dependent** — `Instant.now()` без freeze.
- **Shared state** между тестами.
- **Network** — external calls.
- **Ordering** — полагание на order.
- **Non-deterministic data** — random IDs, HashMap iteration.
- **UI timing** — Selenium без explicit waits.

### 5.2 Как чинить

**Repro**: запусти тест 100 раз локально:
```bash
./gradlew test --tests OrderTest.flakyTest --rerun-tasks
```

Или циклом bash. Если 1/100 падает → flaky.

**Диагностика**:
- Логи.
- Screenshots (для UI).
- Thread dumps.

**Fix**:
- Explicit waits (Awaitility).
- Freeze time (`Clock` injection).
- Isolate data (unique IDs).
- Mock external (WireMock).
- Fix race conditions (mutex, `CompletableFuture.get()`).

### 5.3 Не игнорируй

Flaky test = **broken test**. Игнор → перестанешь верить тестам вообще.

**Retry как workaround** только временно:
```java
@Test
@RepeatedTest(3)
void flakyTest() { ... }
```

Или в CI:
```yaml
retries: 2
```

Но параллельно — чинить причину.

---

## 6. Test data management

### 6.1 Стратегии

**A) Fixed test data** — известные записи в БД (BOOK-1, USER-1).

Плюсы: предсказуемо.
Минусы: shared state, сложность добавления новых тестов.

**B) Create in test** — каждый тест создаёт свои.

Плюсы: изоляция.
Минусы: медленнее.

**C) Random data** — Faker, EasyRandom.

Плюсы: unbiased.
Минусы: непредсказуемо, hard to debug.

**D) Fixtures / builders** — `anOrder().build()`.

Плюсы: читаемо, DRY.
Минусы: другой код поддерживать.

### 6.2 Faker

```java
Faker faker = new Faker(new Locale("ru"));
String name = faker.name().fullName();
String email = faker.internet().emailAddress();
LocalDate date = faker.date().birthday().toInstant()...;
```

Полезно для performance / demo data.

### 6.3 Не путать unit и integration data

- Unit — минимум (только то что тест проверяет).
- Integration — реалистичные (для проверки JOIN, indexes).

---

## 7. Mutation testing

**Mutation testing** — проверка **качества** тестов.

### 7.1 Идея

Инструмент **мутирует** твой код (изменяет одну строку):
- `>` → `>=`
- `+` → `-`
- `true` → `false`
- `return x` → `return null`

Если тесты **всё равно проходят** → тесты не ловят эту ошибку → **survived mutation** → плохо покрытие.

### 7.2 Pitest (PIT)

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

Отчёт в `build/reports/pitest/` — сколько mutations survived.

### 7.3 Metric

- **Mutation score** = killed / total.
- Хорошо: 70%+.
- Отлично: 90%+.

Отражает **real** качество тестов, не просто coverage.

### 7.4 Cost

Медленно (много вариантов). Обычно в **nightly**, не на каждый commit.

---

## 8. Chaos engineering

Уже упоминал в `52-microservices-resilience.md`.

### 8.1 Идея

Ломать намеренно — проверить что система переживает.

Netflix: **Chaos Monkey** случайно убивает production под каждый день. Приложения должны быть fault-tolerant.

### 8.2 Уровни

- **Kill pod** — самое простое.
- **Slow network** — latency / packet loss.
- **DNS failure**.
- **CPU 100%** на ноде.
- **Disk full**.
- **Kill datacenter**.

### 8.3 Инструменты

- **Chaos Monkey** — оригинал (Java).
- **Chaos Mesh** — K8s native.
- **Litmus** — K8s.
- **Gremlin** — SaaS.

### 8.4 GameDay

Планированная сессия: разработчики + операторы + бизнес → намеренная поломка → как система реагирует, время до detection, до fix.

Учит команду отвечать на incidents.

---

## 9. Contract testing (детально)

Из `54-testing-integration-slice.md`.

### 9.1 Проблема

Producer меняет API → consumer ломается **на проде**.

E2E ловят, но поздно и медленно.

### 9.2 Consumer-Driven Contracts

Consumer описывает **что ожидает** → generates contract → Producer verify.

### 9.3 Pact

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

Contract publish в **Pact Broker**.

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

Real service запускается → Pact вызывает по contract → verify response matches.

### 9.4 Spring Cloud Contract

Аналог, Java/Groovy DSL:
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

Автогенерируется producer test + consumer stub.

### 9.5 Плюсы

- **Ловим breaking changes до deploy**.
- **Быстрее E2E**.
- **Legacy compatibility**.

---

## 10. Shift-left testing

**Идея**: тестировать **раньше** в pipeline.

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

Плюсы:
- Bugs ловятся раньше = дешевле fix.
- QA освобождается для exploratory / user testing.
- Быстрее feedback.

**Обязательно** для CI/CD культуры.

---

## 11. Правила ИСНА (из memory)

### 11.1 Прод-смок объём

Memory `knp-e2e-prod-smoke-scope-rules`:
1. **Не смокать мутирующие API**.
2. **Только gap-эндпоинты** (что сейчас не покрыто), потом safe GET.

### 11.2 e2e-gate

`knp-e2e-gate-check-master-race` — гейт можно retry, scope = только изменённый сервис.

### 11.3 Feign smoke

`knp-e2e-feign-smoke` — синтетический смок всех Feign-клиентов.

### 11.4 Реальная топология

`knp-e2e-trigger-topology` — как модули триггерят раннер: inline / E2E_LIST / include / E2E_SCOPE.

---

## 12. Test smells

### 12.1 Excessive setup

Тест начинается со 100 строк setup. → Класс делает слишком много.

### 12.2 Fragile tests

Изменение implementation → ломает тесты. → Тестируешь implementation, не behavior.

### 12.3 Slow tests

Unit тест > 100 мс. → Что-то мокать / рефакторить.

### 12.4 Overspecification

Тест проверяет **всё**. Изменения безобидные → тест падает.

Правило: тест проверяет **одно поведение**.

### 12.5 Assertion roulette

Много assertions без сообщений. Тест падает — не понятно на каком.

Fix: одно логическое утверждение per тест или AssertJ `assertThat().satisfies(...)` для группировки.

### 12.6 Magic numbers

```java
assertThat(order.getTotal()).isEqualTo(542.35);   // 542.35 откуда?
```

Правильно:
```java
BigDecimal expected = PRICE.multiply(QTY).add(TAX);
assertThat(order.getTotal()).isEqualTo(expected);
```

### 12.7 Ignored tests

`@Disabled` без комментария → потерялись, вечно ignored. → Или чинить, или удалить.

---

## 13. Метрики качества тестов

- **Coverage** (line, branch).
- **Mutation score**.
- **Test execution time** — pipeline должен быть быстрым.
- **Flaky rate** — % падений при retry.
- **Test:code ratio** — 1:1 - 3:1 обычно.

Все нужно отслеживать.

---

## 14. Best practices итого

1. **Пирамида** тестов.
2. **AAA / GWT** structure.
3. **F.I.R.S.T** принципы.
4. **AssertJ + Mockito** стандартный стек.
5. **Testcontainers** для integration.
6. **TDD для новой логики**, post-hoc для остального.
7. **BDD** — если QA пишет тесты.
8. **Contract testing** для микросервисов.
9. **Explicit waits** в UI тестах.
10. **Fix flaky immediately**.
11. **Mutation testing** периодически.
12. **Chaos** в staging.
13. **Shift-left** — тесты рано.
14. **Test independence** — auto-rollback / unique data.
15. **Не мокай то что владеешь** — только external.

---

## 15. Собесные вопросы

1. **Что такое TDD?** — Test-Driven Development: тест до кода; Red-Green-Refactor cycle.
2. **Плюсы TDD?** — Coverage, дизайн feedback, regression safety, documentation.
3. **Что такое BDD?** — Behavior-Driven Development; Given-When-Then; Cucumber/Gherkin.
4. **Test doubles — 5 видов?** — Dummy, Stub, Fake, Spy, Mock.
5. **State vs behavior verification?** — State: проверить state; behavior: verify вызовов.
6. **Что такое flaky test?** — Тест иногда падает, иногда — нет; race conditions, timing, shared state.
7. **Как чинить flaky?** — Repro (запустить много раз), explicit waits, isolate data, mock external.
8. **Что такое mutation testing?** — Инструмент мутирует код; если тесты проходят → плохо покрыто.
9. **Pitest — что?** — Java mutation testing tool.
10. **Contract testing — зачем?** — Ловить breaking API changes до deploy; Pact / Spring Cloud Contract.
11. **Что такое chaos engineering?** — Намеренная поломка prod для проверки resilience.
12. **Shift-left testing — что?** — Тестировать раньше в pipeline (unit → CI → post-deploy).
13. **Test smells?** — Excessive setup, fragile, slow, magic numbers, ignored.
14. **Как ускорить тесты?** — Меньше `@SpringBootTest`, parallel execution, mock external, Testcontainers reuse.
15. **Правило «не мокай то что владеешь»?** — Мокай external boundaries; own logic — реальные объекты.

---

## Итог

- **TDD** — тест до кода; Red-Green-Refactor.
- **BDD** — Given-When-Then; Cucumber для QA-friendly.
- **Test doubles**: Dummy/Stub/Fake/Spy/Mock.
- **Isolate tests**, **fix flaky immediately**.
- **Mutation testing** для качества тестов.
- **Contract testing** для микросервисов.
- **Chaos engineering** для resilience validation.
- **Shift-left** — тестировать рано.

---

## Итог блока testing

- 53 — Unit тесты (JUnit + Mockito + AssertJ).
- 54 — Integration + Slice (@SpringBootTest, Testcontainers, contract).
- 55 — E2E + Smoke + Selenium + Performance (Gatling, k6).
- 56 — Best practices (TDD, BDD, mutation, chaos, flaky).

Всего 4 файла тестирования. Следующий блок — новые темы (Spring config, Helm, Consul глубже, аннотации, controllers, REST vs SOAP). Начинаю с `57-spring-boot-config-detailed.md`.
