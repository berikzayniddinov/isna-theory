# 53. Тестирование: пирамида + Unit тесты

Зачем тесты, виды, детально про unit-тесты с JUnit + Mockito + AssertJ.

---

## 1. Зачем писать тесты

- **Regression prevention** — правишь одно, ломаешь другое → тест ловит.
- **Documentation** — тест показывает как использовать код.
- **Confidence при рефакторинге** — переписал, тесты зелёные → работает.
- **Design feedback** — если код сложно тестировать → плохо спроектирован.
- **Faster development** — на самом деле, ловля багов раньше = дешевле.

Без тестов через 6 месяцев страшно менять код.

---

## 2. Пирамида тестов

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

**Правило**: много unit, меньше integration, ещё меньше E2E.

Обратная пирамида (много E2E, мало unit) — anti-pattern. Медленно, flaky, дорого поддерживать.

### 2.1 Ice cream cone (anti-pattern)

Наоборот: много manual + E2E, мало unit. Классический legacy без тестов.

### 2.2 Diamond

Middle-heavy: много integration, средне unit + E2E. Иногда OK для микросервисов (много interaction).

---

## 3. Уровни тестов

### 3.1 Unit tests

Тестируют **один класс / метод** в изоляции. Мокают всё внешнее.

- Быстро (мс).
- Много (тысячи).
- Легко.
- **Не** проверяют интеграцию.

### 3.2 Integration tests

Тестируют **связку компонентов** (класс + БД, класс + Rabbit).

- Медленнее (секунды).
- Меньше (сотни).
- Реалистичнее.
- Часто с **Testcontainers**.

### 3.3 Component tests

Один микросервис целиком, но external dependencies моканы (WireMock).

### 3.4 Contract tests

Тестируют **контракт между сервисами** (Spring Cloud Contract, Pact).

### 3.5 E2E tests

Полная цепочка через все сервисы + БД + broker.

- Медленно (минуты).
- Мало (десятки).
- Flaky.
- Ловят интеграционные баги.

### 3.6 Smoke tests

**Быстрая проверка**: система жива? Основные операции работают?

Обычно после deploy — прогон 5-20 критичных сценариев.

---

## 4. JUnit 5

Стандарт в Java.

### 4.1 Базовый тест

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

- `@Test` — метод-тест.
- Тестовый класс — по конвенции `<Class>Test`.
- Метод — glagol_состояние (`createOrder_savesToDb`).

### 4.2 Lifecycle

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

### 4.3 Assertions

**AssertJ** — намного лучше JUnit assertions:

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

Более читаемо + лучшие сообщения об ошибках. Стандарт де-факто.

### 4.4 Exception testing

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

### 4.5 Nested tests

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

Организовать группы связанных тестов.

### 4.6 Parameterized tests

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

Один тест — много cases. Уменьшает copy-paste.

### 4.7 Display name

```java
@Test
@DisplayName("Order с amount > 0 сохраняется в БД")
void createOrder_savesToDb() { ... }
```

Красиво в отчёте.

### 4.8 Disabled / conditional

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

### 4.9 Assumptions

```java
@Test
void test() {
    assumeTrue(dbAvailable(), "DB не доступна — skip");
    // если false → skipped, не failed
}
```

---

## 5. Mockito

Библиотека моков.

### 5.1 Простой mock

```java
OrderRepository repo = mock(OrderRepository.class);

when(repo.findById(1L)).thenReturn(Optional.of(new Order(...)));
when(repo.findById(999L)).thenReturn(Optional.empty());

Optional<Order> result = repo.findById(1L);
assertThat(result).isPresent();
```

### 5.2 Verify (проверка что вызван)

```java
verify(repo).save(any());              // ровно 1 раз
verify(repo, times(3)).save(any());    // ровно 3
verify(repo, atLeast(1)).save(any());
verify(repo, never()).delete(any());
verifyNoMoreInteractions(repo);
```

### 5.3 Argument matchers

```java
verify(repo).save(any(Order.class));
verify(repo).save(argThat(o -> o.getStatus() == NEW));
verify(repo).findById(eq(1L));
verify(repo).findByStatus(any(OrderStatus.class));
```

Правило: если один argument — matcher, то **все** через matchers.

### 5.4 ArgumentCaptor

Захватить аргумент для inspection:
```java
ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
verify(repo).save(captor.capture());

Order saved = captor.getValue();
assertThat(saved.getStatus()).isEqualTo(NEW);
assertThat(saved.getCreatedAt()).isCloseTo(now(), within(1, SECONDS));
```

### 5.5 Throw exception

```java
when(repo.findById(1L)).thenThrow(new RuntimeException("DB error"));
```

### 5.6 Void methods

```java
doNothing().when(repo).delete(any());
doThrow(new RuntimeException()).when(repo).delete(any());
```

### 5.7 Spy (частичный mock)

```java
List<String> list = new ArrayList<>();
List<String> spy = spy(list);

spy.add("a");
verify(spy).add("a");
assertThat(spy).hasSize(1);   // реальный метод выполнился
```

Использовать редко — обычно plain mock лучше.

### 5.8 @Mock / @InjectMocks аннотации

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

Mockito автоматически создаёт mocks + inject в `@InjectMocks` через конструктор.

### 5.9 mockStatic

Для static методов:
```java
try (MockedStatic<Instant> mocked = mockStatic(Instant.class)) {
    mocked.when(Instant::now).thenReturn(Instant.parse("2026-09-05T10:00:00Z"));
    Instant result = Instant.now();
    assertThat(result).isEqualTo(Instant.parse("2026-09-05T10:00:00Z"));
}
```

Использовать редко (static usage — плохой дизайн).

---

## 6. Test doubles — терминология

Meszaros в «xUnit Test Patterns»:
- **Dummy** — объект-заглушка, значение не важно (для параметра).
- **Stub** — возвращает заранее заданные ответы.
- **Spy** — записывает вызовы для проверки + может выполнять реальную логику.
- **Mock** — как stub, но verify что вызван.
- **Fake** — упрощённая реальная реализация (in-memory DB).

Mockito делает **все**, часто просто говорят «mock».

---

## 7. Хороший unit test

### 7.1 F.I.R.S.T принципы

- **F**ast — миллисекунды.
- **I**solated / **I**ndependent — тест не зависит от других.
- **R**epeatable — одинаковый результат каждый раз.
- **S**elf-validating — pass/fail автоматически (без ручного просмотра).
- **T**imely — писать одновременно с кодом (TDD) или сразу после.

### 7.2 AAA pattern

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

3 секции. Читается сверху вниз.

### 7.3 Один assert per test (не догма)

Идеал — один assert на тест. Но:
```java
assertThat(order)
    .satisfies(o -> {
        assertThat(o.getStatus()).isEqualTo(NEW);
        assertThat(o.getTotal()).isEqualTo(100);
        assertThat(o.getCustomerId()).isEqualTo("customer-1");
    });
```

Несколько assertions на **одном объекте** — норма.

Несколько **разных вещей** — обычно разные тесты.

### 7.4 Naming

Формат: `methodName_condition_expectedResult`
```
createOrder_withValidRequest_savesToDb
createOrder_withNullRequest_throwsException
findById_whenNotFound_returnsEmpty
```

Или BDD-style:
```
should_save_order_when_request_is_valid
```

### 7.5 Что мокать

- **External** (БД, HTTP, файлы) → mock.
- **Slow** (сеть, GC) → mock.
- **Non-deterministic** (time, random) → mock / inject.

- **Own logic** — НЕ мокать (иначе тестируешь mock, не код).

### 7.6 Правило не мокать value objects

```java
// плохо
Instant now = mock(Instant.class);
when(now.isBefore(...)).thenReturn(true);

// хорошо
Instant now = Instant.parse("2026-09-05T10:00:00Z");
```

`Instant`, `String`, `LocalDate` — value objects. Просто создавай реальные.

---

## 8. Code coverage

Мера сколько % кода покрыто тестами.

### 8.1 Инструменты

- **JaCoCo** — стандарт в Java.
- **Cobertura** (старый).

### 8.2 Уровни

- **Line coverage** — % строк выполнено.
- **Branch coverage** — % условных ветвей (`if`).
- **Method coverage**.
- **Class coverage**.

Обычно смотрят line + branch.

### 8.3 Как настроить

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

`./gradlew test jacocoTestReport` → отчёт в `build/jacoco/`.

### 8.4 Coverage ВРЁТ

**100% coverage ≠ 100% tested**.

```java
@Test
void test() {
    calculator.add(1, 2);
    // никаких assertions! но 100% строк выполнено
}
```

Coverage говорит **какой код выполнен**, не **правильно ли работает**.

Используй как guide (что НЕ покрыто), не как цель.

### 8.5 Разумные цели

- 60-80% overall — норма.
- 90%+ для критичной бизнес-логики.
- 0-40% для boilerplate (DTO, config).

Погоня за 95%+ на всём → пишешь тесты ради тестов.

---

## 9. Что тестировать в unit

### 9.1 Тестируй

- **Бизнес-логика** — расчёты, валидация, state transitions.
- **Edge cases** — null, empty, boundary (0, -1, MAX_VALUE).
- **Exception paths** — что выбрасывается на invalid input.
- **Interaction** — что нужный метод вызывается (verify).

### 9.2 Не тестируй

- **Getters/setters** — не полезно.
- **Framework code** — Spring, Hibernate уже тестировали.
- **Trivial code** — `return this.field`.
- **Реализацию** — тестируй **поведение**, не implementation details (иначе рефакторинг = ломает тесты).

---

## 10. Пример полного unit test

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

---

## 11. Собесные вопросы

1. **Что такое пирамида тестов?** — Много unit (быстро), меньше integration, ещё меньше E2E (медленно).
2. **JUnit 5 vs JUnit 4?** — Другой пакет (jupiter), @BeforeEach вместо @Before, Extension model, native parameterized, nested tests.
3. **Что такое AssertJ?** — Fluent assertion library; лучше JUnit assertions по читаемости и сообщениям.
4. **Что такое Mockito?** — Библиотека моков; `mock()`, `when().thenReturn()`, `verify()`.
5. **@Mock vs mock()?** — Аннотация для JUnit; работает через MockitoExtension.
6. **Что такое ArgumentCaptor?** — Захват аргумента для inspection после вызова.
7. **spy vs mock?** — spy на реальном объекте (частично реальный); mock — полностью fake.
8. **AAA pattern?** — Arrange / Act / Assert.
9. **F.I.R.S.T?** — Fast, Isolated, Repeatable, Self-validating, Timely.
10. **Что такое code coverage?** — % кода выполненного тестами; JaCoCo стандарт.
11. **100% coverage — хорошо?** — Coverage не гарантирует качество; хорошо для критичной логики, вредно для boilerplate.
12. **Test doubles?** — Dummy/Stub/Spy/Mock/Fake (терминология xUnit).
13. **Что мокать?** — External (БД, HTTP), slow, non-deterministic; НЕ own logic.
14. **Что не тестировать unit?** — Getters/setters, framework, trivial code, implementation details.
15. **Parameterized tests — когда?** — Один тест на много cases с разными inputs.

---

## Итог

- **Пирамида**: много unit, меньше integration, чуть E2E.
- **JUnit 5** + **AssertJ** + **Mockito** = стандартный стек.
- **AAA pattern**, **F.I.R.S.T**.
- **Verify** взаимодействий + **ArgumentCaptor** для inspection.
- **Coverage** как guide, не цель.
- **Тестируй поведение**, не implementation.
- Мокать external, не own logic.

Следующий — `54-testing-integration-slice.md`.
