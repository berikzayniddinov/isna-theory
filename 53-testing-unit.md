# 53. Тестирование: пирамида, unit tests с JUnit, Mockito, AssertJ

## Зачем понимать тесты глубоко

Разработчик который недавно в профессии часто рассматривает тесты как «должны быть» — обязательство писать, а не инструмент для эффективного разработки. Пишет тесты чтобы удовлетворить coverage metric в CI. Тесты медленные, flaky, тесно связанные с implementation details — при рефакторинге ломаются, требуют переписывания. Тесты становятся burden а не asset.

Разница между разработчиком «пишущим тесты» и «понимающим тестирование» очевидна в скорости работы через кодовую базу. Первый боится менять существующий код — тесты падают из-за implementation coupling. Второй знает что тесты должны быть fast, isolated, repeatable, self-validating, timely (F.I.R.S.T principles). Знает что unit tests should test behavior не implementation — refactoring should not break tests. Знает что pyramid pattern (many unit, some integration, few E2E) provides good coverage при maintainable cost. Знает Mockito idioms — how mocks differ от stubs vs spies, when ArgumentCaptor нужен, avoiding brittle test setups.

В этом файле разберём testing глубоко. Зачем реально писать тесты (beyond obligation). Test pyramid как model. Levels tests. JUnit 5 mechanics — annotations, lifecycle, parameterized tests, nested tests, extensions. AssertJ fluent assertions — почему лучше JUnit assertions. Mockito comprehensively — mocks, verification, argument matchers, ArgumentCaptor, spy. Test doubles terminology (dummy, stub, spy, mock, fake). F.I.R.S.T principles applied. AAA pattern. Naming conventions. Code coverage tools plus limitations. Что тестировать и что не тестировать.

## Реальные reasons писать тесты

Regression prevention — правишь одно, ломаешь другое. Test catches. Without tests — bugs escape к production. Users find them. Cost multiplies.

Documentation. Тест показывает как код should be used. Executable documentation что stays current (unlike text docs that rot).

Confidence при refactoring. Переписал, тесты зелёные — работает. Without tests — every refactor risky. Developers become afraid to change code — codebase stagnates.

Design feedback. Если код сложно тестировать — плохо спроектирован. High coupling, hidden dependencies, side effects — all make testing hard. Testing pain forces better design.

Faster development. Counter-intuitive but true — ловля багов раньше = дешевле. Bug fixed при написании тестового case takes 5 minutes. Same bug fixed через customer support через 6 months takes hours or days.

Без тестов через 6 месяцев страшно менять код. Fear leads к workarounds (adding parallel functionality вместо fixing existing). Codebase entropy accelerates.

## Пирамида тестов

Classic model от Mike Cohn. Different levels tests в correct proportions:
```
                  ▲
                  │
                  │      /  \       ← E2E / UI (10%)
                  │     /    \        медленные, дорогие, flaky
                  │    /______\
                  │   /        \    ← Integration (20%)
                  │  /          \     средние, реалистичнее
                  │ /____________\
                  │/              \  ← Unit (70%)
                  /                \   быстрые, дешёвые, изолированные
                 /__________________\
```

Правило — много unit, меньше integration, ещё меньше E2E. Обеспечивает balance между coverage, speed, maintenance cost.

Обратная пирамида (много E2E, мало unit) — anti-pattern. Медленно (feedback slow). Flaky (many dependencies fail intermittently). Дорого поддерживать (complex setup).

Ice cream cone — anti-pattern. Много manual plus E2E, мало unit. Classical legacy без тестов. Common если tests added late.

Diamond shape — middle-heavy. Много integration, средне unit plus E2E. Sometimes OK для микросервисов (много interaction тesting warranted). Not standard rec but not wrong per se.

## Уровни тестов

Unit tests. Тестируют один класс / метод в изоляции. Мокают всё external. Быстро (milliseconds). Много (thousands). Легко писать. Не проверяют интеграцию.

Integration tests. Тестируют связку компонентов (класс plus БД, класс plus Rabbit). Медленнее (seconds). Меньше (hundreds). Реалистичнее. Часто с Testcontainers для real dependencies.

Component tests. Один микросервис целиком, но external dependencies моканы (WireMock для HTTP downstreams). Boundary тестирование one service.

Contract tests. Тестируют контракт между сервисами (Spring Cloud Contract, Pact). Provider generates stubs — consumer tests against them. Ensures API stability across services.

E2E tests. Полная цепочка через все сервисы plus БД plus broker. Медленно (minutes). Мало (dozens). Flaky. Ловят integration bugs missed elsewhere.

Smoke tests. Быстрая проверка — система жива? Основные операции работают? Обычно после deploy — прогон 5-20 критичных сценариев за 1-5 минут.

Каждый level имеет свое место. Full coverage requires combination.

## JUnit 5 mechanics

Стандарт в Java. JUnit 5 (Jupiter) replaced JUnit 4 около 2017.

Базовый тест:
```java
class OrderServiceTest {

    @Test
    void createOrder_savesToDb() {
        OrderRepository repo = mock(OrderRepository.class);
        OrderService svc = new OrderService(repo);

        svc.createOrder(new OrderRequest("customer-1", 100));

        verify(repo).save(any());
    }
}
```

@Test — метод-тест. Тестовый класс — по конвенции ClassNameTest. Метод — glagol_состояние format (createOrder_savesToDb).

Lifecycle annotations:
```java
class OrderServiceTest {

    @BeforeAll
    static void setupOnce() {
        // раз перед всеми тестами класса
    }

    @BeforeEach
    void setupEach() {
        // перед каждым тестом
    }

    @Test
    void test1() { ... }

    @Test
    void test2() { ... }

    @AfterEach
    void tearDownEach() { ... }

    @AfterAll
    static void tearDownAll() { ... }
}
```

@BeforeAll и @AfterAll — static, executed once per class. @BeforeEach и @AfterEach — instance methods, executed per test.

Assertions с AssertJ — намного лучше JUnit standard assertions:
```java
// JUnit standard
assertEquals(expected, actual);
assertTrue(condition);

// AssertJ (fluent, богаче)
assertThat(actual).isEqualTo(expected);
assertThat(list).hasSize(3).contains("a", "b").doesNotContain("c");
assertThat(order.getTotal()).isCloseTo(100.0, offset(0.01));
assertThat(map).containsEntry("key", "value");
assertThat(instant).isAfter(otherInstant).isBefore(now);
assertThat(exception)
    .isInstanceOf(BusinessException.class)
    .hasMessage("invalid order")
    .hasCauseInstanceOf(SQLException.class);
```

Более читаемо plus лучшие сообщения об ошибках. Стандарт де-факто.

Exception testing:
```java
@Test
void createOrder_throwsWhenInvalid() {
    OrderService svc = new OrderService(mock(OrderRepository.class));

    // AssertJ
    assertThatThrownBy(() -> svc.createOrder(null))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("request cannot be null");

    // Или JUnit
    IllegalArgumentException ex = assertThrows(
        IllegalArgumentException.class,
        () -> svc.createOrder(null));
    assertEquals("request cannot be null", ex.getMessage());
}
```

Nested tests для организации:
```java
class OrderServiceTest {

    @Nested
    class WhenOrderIsNew {
        @Test
        void savesToDb() { ... }
        @Test
        void publishesEvent() { ... }
    }

    @Nested
    class WhenOrderIsDuplicate {
        @Test
        void throwsException() { ... }
    }
}
```

Groups related tests. IDE display shows organized structure.

Parameterized tests:
```java
@ParameterizedTest
@ValueSource(strings = {"", " ", "\t"})
void isBlank_returnsTrueForWhitespace(String input) {
    assertThat(StringUtils.isBlank(input)).isTrue();
}

@ParameterizedTest
@CsvSource({
    "1, 1, 2",
    "2, 3, 5",
    "0, 0, 0"
})
void add(int a, int b, int expected) {
    assertThat(calc.add(a, b)).isEqualTo(expected);
}

@ParameterizedTest
@MethodSource("orderProvider")
void processOrder(OrderRequest req, OrderStatus expected) { ... }

static Stream<Arguments> orderProvider() {
    return Stream.of(
        arguments(new OrderRequest("A", 100), OrderStatus.NEW),
        arguments(new OrderRequest("B", 0), OrderStatus.INVALID)
    );
}
```

Один тест — много cases. Reduces copy-paste. Standard когда testing multiple inputs.

Display names:
```java
@Test
@DisplayName("Order с amount > 0 сохраняется в БД")
void createOrder_savesToDb() { ... }
```

Красиво в отчёте. Improves readability especially для business stakeholders reviewing test reports.

Disabled и conditional tests:
```java
@Test
@Disabled("флаки, чинить в JIRA-123")
void oldTest() { ... }

@Test
@EnabledOnOs(OS.LINUX)
void linuxOnly() { ... }

@Test
@EnabledIfEnvironmentVariable(named = "CI", matches = "true")
void onlyInCI() { ... }
```

Assumptions для skip logic:
```java
@Test
void test() {
    assumeTrue(dbAvailable(), "DB не доступна — skip");
    // если false → skipped, не failed
}
```

Different from assertion — assumption не failing test, marks as skipped.

## Mockito comprehensively

Библиотека моков. Стандарт в Java для test doubles.

Простой mock:
```java
OrderRepository repo = mock(OrderRepository.class);

when(repo.findById(1L)).thenReturn(Optional.of(new Order(...)));
when(repo.findById(999L)).thenReturn(Optional.empty());

Optional<Order> result = repo.findById(1L);
assertThat(result).isPresent();
```

Mockito.mock creates fake implementation. when.thenReturn programs behavior. Not doubling real object — full fake.

Verify (проверка что вызван):
```java
verify(repo).save(any());              // ровно 1 раз (default)
verify(repo, times(3)).save(any());    // ровно 3
verify(repo, atLeast(1)).save(any());
verify(repo, never()).delete(any());
verifyNoMoreInteractions(repo);
```

Ensures methods called as expected. Testing interactions between components. VerifyNoMoreInteractions catches unexpected calls.

Argument matchers:
```java
verify(repo).save(any(Order.class));
verify(repo).save(argThat(o -> o.getStatus() == NEW));
verify(repo).findById(eq(1L));
verify(repo).findByStatus(any(OrderStatus.class));
```

Правило — если один argument matcher, то все через matchers. Cannot mix concrete values с matchers.

ArgumentCaptor захватить аргумент для inspection:
```java
ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
verify(repo).save(captor.capture());

Order saved = captor.getValue();
assertThat(saved.getStatus()).isEqualTo(NEW);
assertThat(saved.getCreatedAt()).isCloseTo(now(), within(1, SECONDS));
```

More powerful than argThat lambdas — captured value inspectable через все assertions. Useful для complex object verification.

Throw exception:
```java
when(repo.findById(1L)).thenThrow(new RuntimeException("DB error"));
```

Simulates error scenarios. Testing error handling paths.

Void methods special syntax:
```java
doNothing().when(repo).delete(any());
doThrow(new RuntimeException()).when(repo).delete(any());
```

when.thenReturn не работает с void — использовать doNothing.when.

Spy — частичный mock:
```java
List<String> list = new ArrayList<>();
List<String> spy = spy(list);

spy.add("a");
verify(spy).add("a");
assertThat(spy).hasSize(1);   // реальный метод выполнился
```

Real object underneath. Methods actually execute unless stubbed. Verification works normally. Useful для partial mocking когда most behavior real, some parts stubbed. Использовать редко — обычно plain mock лучше (cleaner).

@Mock и @InjectMocks annotations:
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository repo;
    @Mock EventPublisher publisher;

    @InjectMocks OrderService svc;

    @Test
    void createOrder() {
        svc.createOrder(new OrderRequest(...));
        verify(repo).save(any());
    }
}
```

Mockito автоматически создаёт mocks plus inject в @InjectMocks через конструктор. Cleaner чем manual creation в @BeforeEach.

mockStatic для static методов:
```java
try (MockedStatic<Instant> mocked = mockStatic(Instant.class)) {
    mocked.when(Instant::now).thenReturn(Instant.parse("2026-09-05T10:00:00Z"));
    Instant result = Instant.now();
    assertThat(result).isEqualTo(Instant.parse("2026-09-05T10:00:00Z"));
}
```

Использовать редко. Static usage — плохой дизайн usually. Prefer clock injection для controlling time в tests.

## Test doubles terminology

Meszaros в «xUnit Test Patterns» defined precise terminology.

Dummy — объект-заглушка, значение не важно (для параметра метода что doesn't use it).

Stub — возвращает заранее заданные ответы. Simple canned responses.

Spy — записывает вызовы для проверки plus может выполнять реальную логику. Real object with additional recording.

Mock — как stub, но verify что вызван. Behavior expectations verified.

Fake — упрощённая реальная реализация. In-memory database instead of real one. Functional but simplified.

Mockito делает все категории, часто просто говорят «mock». Practical usage doesn't strictly distinguish типы часто. Общепринятое использование — mock covers все.

## F.I.R.S.T principles

Хороший unit test следует пяти principles.

Fast — миллисекунды. Slow tests не runable часто. Feedback loop broken. Developers stop running tests.

Isolated / Independent — тест не зависит от других. Order шouldn't matter. Test доменый state не affects другие.

Repeatable — одинаковый результат каждый раз. Not depending on time, random, environment. Deterministic outputs.

Self-validating — pass/fail автоматически. No manual output inspection required. Assertions clear yes/no.

Timely — писать одновременно с кодом (TDD) или сразу после. Retroactive testing painful. Design suffers.

## AAA pattern

Arrange / Act / Assert structure:
```java
@Test
void createOrder_savesToDb() {
    // Arrange
    OrderRepository repo = mock(OrderRepository.class);
    OrderService svc = new OrderService(repo);
    OrderRequest req = new OrderRequest("customer-1", 100);

    // Act
    svc.createOrder(req);

    // Assert
    verify(repo).save(argThat(o -> o.getCustomerId().equals("customer-1")));
}
```

Три секции. Читается сверху вниз. Standard structure понятная любому reader.

Sometimes called Given/When/Then (BDD style) — same three sections different names.

## Один assert per test

Идеал — один assert на тест. Reduces confusion при failure — know exactly what went wrong.

Но не dogma. Multiple assertions на одном объекте — норма:
```java
assertThat(order)
    .satisfies(o -> {
        assertThat(o.getStatus()).isEqualTo(NEW);
        assertThat(o.getTotal()).isEqualTo(100);
        assertThat(o.getCustomerId()).isEqualTo("customer-1");
    });
```

Several aspects of one object make sense в one test. Different behaviors обычно разные tests.

## Naming conventions

Format methodName_condition_expectedResult:
```
createOrder_withValidRequest_savesToDb
createOrder_withNullRequest_throwsException
findById_whenNotFound_returnsEmpty
```

Или BDD-style:
```
should_save_order_when_request_is_valid
```

Consistency важна. Team agreement — pick one style, use consistently. Test names read like specification.

## Что мокать

External. БД (unless using @DataJpaTest with H2), HTTP calls, files, sockets — mock. Real infrastructure = slow.

Slow. Сеть, GC-triggering operations — mock. Otherwise tests slow.

Non-deterministic. Time, random — mock или inject. Otherwise tests flaky.

Own logic — НЕ мокать. Иначе тестируешь mock, не код. Мoking your own code creates fake tests.

Правило не мокать value objects:
```java
// плохо
Instant now = mock(Instant.class);
when(now.isBefore(...)).thenReturn(true);

// хорошо
Instant now = Instant.parse("2026-09-05T10:00:00Z");
```

Instant, String, LocalDate — value objects. Просто создавай реальные. Cheap. Better test clarity.

## Code coverage

Мера сколько процентов кода покрыто тестами.

Инструменты. JaCoCo — стандарт в Java. Cobertura (старый).

Уровни coverage. Line coverage — процент строк выполнено. Branch coverage — процент условных ветвей (if branches). Method coverage. Class coverage.

Обычно смотрят line plus branch. Branch coverage more revealing — catches paths not exercised.

Setup через build:
```gradle
plugins {
    id 'jacoco'
}

test {
    finalizedBy jacocoTestReport
}

jacocoTestReport {
    reports {
        html.required = true
        xml.required = true
    }
}
```

`./gradlew test jacocoTestReport` → отчёт в build/jacoco/.

Coverage ВРЁТ важное caveat. 100% coverage ≠ 100% tested:
```java
@Test
void test() {
    calculator.add(1, 2);
    // никаких assertions!
    // 100% строк выполнено но ничего не проверено
}
```

Coverage говорит какой код выполнен, не правильно ли работает. Assertion quality matters more than code coverage.

Используй как guide (что НЕ покрыто), не как цель. Chasing 95%+ coverage everywhere leads к low-quality tests written just для coverage.

Разумные цели. 60-80% overall — норма. 90%+ для критичной бизнес-логики. 0-40% для boilerplate (DTOs, config). Focus coverage on important code.

## Что тестировать в unit

Тестируй. Бизнес-логика — расчёты, валидация, state transitions. Edge cases — null, empty, boundary (0, -1, MAX_VALUE). Exception paths — что выбрасывается на invalid input. Interaction — что нужный метод вызывается (verify).

Не тестируй. Getters/setters — не полезно. Framework code — Spring, Hibernate уже тестировали. Trivial code — return this.field. Реализацию — тестируй поведение, не implementation details (иначе refactoring = ломает тесты).

Testing behavior means tests pass после refactoring same behavior. Testing implementation means tests break when internals change even если behavior same. Brittleness от implementation testing painful для maintenance.

## Пример полного unit test

Comprehensive example demonstrating best practices:
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository repo;
    @Mock PaymentClient paymentClient;
    @Mock ApplicationEventPublisher events;

    @InjectMocks OrderService svc;

    @Test
    @DisplayName("createOrder с валидным request сохраняет Order и публикует событие")
    void createOrder_valid_savesAndPublishes() {
        // Arrange
        OrderRequest req = new OrderRequest("customer-1", BigDecimal.valueOf(100));
        when(paymentClient.validate(req.customerId())).thenReturn(true);

        // Act
        Order result = svc.createOrder(req);

        // Assert
        ArgumentCaptor<Order> orderCaptor = ArgumentCaptor.forClass(Order.class);
        verify(repo).save(orderCaptor.capture());
        Order saved = orderCaptor.getValue();
        assertThat(saved)
            .satisfies(o -> {
                assertThat(o.getCustomerId()).isEqualTo("customer-1");
                assertThat(o.getTotal()).isEqualTo(BigDecimal.valueOf(100));
                assertThat(o.getStatus()).isEqualTo(OrderStatus.NEW);
                assertThat(o.getCreatedAt()).isNotNull();
            });

        verify(events).publishEvent(any(OrderCreatedEvent.class));
    }

    @Test
    @DisplayName("createOrder с null throws IllegalArgumentException")
    void createOrder_null_throws() {
        assertThatThrownBy(() -> svc.createOrder(null))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("request cannot be null");

        verify(repo, never()).save(any());
        verify(events, never()).publishEvent(any());
    }

    @Test
    @DisplayName("createOrder когда customer не валидируется — throws + не сохраняет")
    void createOrder_invalidCustomer_throwsAndSkipsSave() {
        OrderRequest req = new OrderRequest("bad-customer", BigDecimal.TEN);
        when(paymentClient.validate("bad-customer")).thenReturn(false);

        assertThatThrownBy(() -> svc.createOrder(req))
            .isInstanceOf(BusinessException.class)
            .hasMessage("customer not valid");

        verify(repo, never()).save(any());
    }

    @ParameterizedTest
    @ValueSource(strings = {"", " ", "\t"})
    @DisplayName("createOrder с пустым customerId — throws")
    void createOrder_blankCustomerId_throws(String customerId) {
        OrderRequest req = new OrderRequest(customerId, BigDecimal.TEN);

        assertThatThrownBy(() -> svc.createOrder(req))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

Demonstrates. Multiple tests per class. Different scenarios explicitly named. ArgumentCaptor для detailed inspection. verify never для negative assertions. Parameterized test для multiple similar inputs.

## Итоги

Testing serves multiple purposes — regression prevention, documentation, refactoring confidence, design feedback, faster development. Not just obligation.

Pyramid pattern — many unit, some integration, few E2E. Balance between coverage, speed, maintenance cost.

JUnit 5 provides. Test annotations. Lifecycle methods. Assertions (though AssertJ preferred). Parameterized tests. Nested tests. Extensions model.

AssertJ preferred over JUnit assertions. Fluent API. Better error messages. Rich matchers.

Mockito для test doubles. Mock creation через mock() или @Mock. when.thenReturn programs behavior. verify checks invocations. ArgumentCaptor для detailed inspection. @InjectMocks automated wiring.

Test doubles terminology — dummy, stub, spy, mock, fake. Practical usage uses «mock» generically.

F.I.R.S.T principles — Fast, Isolated, Repeatable, Self-validating, Timely.

AAA pattern — Arrange, Act, Assert. Standard structure.

Один assert идеал но не dogma. Multiple assertions on same object OK. Different behaviors — separate tests.

Naming — methodName_condition_expectedResult или should_behavior_when_condition. Team consistency важна.

Coverage as guide не цель. 100% coverage ≠ 100% tested. Assertion quality более important than coverage percentage.

Тестировать бизнес-логику, edge cases, exception paths, interactions. Не тестировать getters, framework code, trivial code, implementation details.

Мокать external, slow, non-deterministic. Не мокать own logic и value objects.

Дальше — integration testing plus slice tests в Spring Boot с Testcontainers.
