# 55. E2E, Smoke, Selenium, Performance тесты

Тесты вершины пирамиды. Реальные user flows.

---

## 1. E2E (End-to-end) тесты

**E2E** = полная цепочка через **все компоненты**: UI → backend → БД → external → response.

### 1.1 Пример

Пользователь на сайте:
1. Открывает `knp.kgd.gov.kz`.
2. Логинится через ЭЦП.
3. Открывает форму ФНО.
4. Заполняет.
5. Подписывает.
6. Отправляет.
7. Видит в списке отправленных.

E2E тест: делает всё это программно, проверяет что каждый шаг работает.

### 1.2 Зачем

- Ловит **integration bugs**, невидимые в unit/integration.
- Проверяет **critical user journeys**.
- Regression на **production**-like environment.

### 1.3 Кавет

- **Медленные** — минуты на тест.
- **Flaky** — сеть, timing, UI-элементы.
- **Дорого** — поддержка + инфраструктура.
- **Сложно отладить** — что упало где?

Правило: **мало E2E**, только критичные paths.

---

## 2. Smoke тесты

**Smoke test** = «дым идёт?». Быстрая проверка что базовые вещи работают.

### 2.1 Идея

После deploy — прогнать 5-20 критичных тестов за 1-5 минут. Если зелёные → всё OK, если красное → откатить.

### 2.2 Что проверять

- Приложение отвечает (`/actuator/health`).
- Логин работает.
- Основной GET endpoint возвращает данные.
- POST одного ресурса создаётся.

**Не проверять**: edge cases, полные бизнес-процессы. Smoke = «критичное работает», не «всё работает».

### 2.3 В ИСНА — knp-e2e

Из memory:
- `knp-e2e` — раннер для e2e-тестов против живого стенда.
- **Прод-смок** (`prod-smoke-*`) — набор критичных проверок.
- Разные наборы для разных сервисов: `prod-smoke-fno-ul`, `prod-smoke-fo-ul`.

Правила пользователя (memory `knp-e2e-prod-smoke-scope-rules`):
1. **НЕ смокать мутирующие/опасные API** (send/запись/запрос в ЛС-ИШ-ОС под ЭЦП владельца).
2. **Только gap-эндпоинты** — покрывать отсутствующие сейчас, потом только safe GET.

### 2.4 Синтетический мониторинг

Smoke прогоны каждые N минут (Grafana Synthetics, Pingdom, Datadog Synthetics).

Даёт alert если что-то упало между полными деплоями.

---

## 3. Виды тестов по функционалу

### 3.1 Functional tests

Проверяют что функция работает как ожидается. Unit/integration/E2E — все могут быть functional.

### 3.2 Non-functional

- **Performance** — скорость.
- **Load** — сколько req/sec.
- **Stress** — до какого предела.
- **Soak** — работает ли долго (утечки).
- **Spike** — как переживёт резкий всплеск.
- **Security** — уязвимости.
- **Usability** — удобство UI.
- **Accessibility** — a11y.

---

## 4. UI тесты — Selenium

**Selenium WebDriver** — стандарт для UI-автоматизации.

### 4.1 Как работает

```
Test code (Java) → Selenium WebDriver API → Browser (ChromeDriver / GeckoDriver) → реальный Chrome / Firefox
```

Программно управляешь браузером: клики, ввод, чтение элементов.

### 4.2 Зависимости

```gradle
testImplementation 'org.seleniumhq.selenium:selenium-java:4.16.0'
testImplementation 'io.github.bonigarcia:webdrivermanager:5.6.0'   // авто-скачивание driver
```

### 4.3 Базовый пример

```java
class LoginTest {

    WebDriver driver;

    @BeforeEach
    void setup() {
        WebDriverManager.chromedriver().setup();
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless");
        driver = new ChromeDriver(options);
    }

    @AfterEach
    void tearDown() {
        driver.quit();
    }

    @Test
    void loginSuccess() {
        driver.get("https://knp.kgd.gov.kz/login");

        driver.findElement(By.id("username")).sendKeys("berik");
        driver.findElement(By.id("password")).sendKeys("secret");
        driver.findElement(By.id("submit")).click();

        WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
        WebElement dashboard = wait.until(
            ExpectedConditions.visibilityOfElementLocated(By.id("dashboard")));

        assertThat(dashboard.isDisplayed()).isTrue();
    }
}
```

### 4.4 Локаторы

- `By.id("username")`.
- `By.name("email")`.
- `By.className("btn-primary")`.
- `By.cssSelector("input[type='text']")`.
- `By.xpath("//button[contains(text(), 'Submit')]")`.
- `By.linkText("Log out")`.

Правило: **id > css > xpath**. Xpath ломкий.

### 4.5 Waits

**Fatal error**: `Thread.sleep(5000)` — flaky.

**Правильно**: explicit waits.

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// ждать элемент видимым
wait.until(ExpectedConditions.visibilityOfElementLocated(By.id("...")));

// ждать элемент кликабельным
wait.until(ExpectedConditions.elementToBeClickable(By.id("...")));

// ждать текст
wait.until(ExpectedConditions.textToBePresentInElement(el, "Success"));

// ждать URL
wait.until(ExpectedConditions.urlContains("/dashboard"));

// custom condition
wait.until(d -> d.findElements(By.className("row")).size() > 0);
```

**Implicit wait** — глобальный (не рекомендуется, скрывает проблемы):
```java
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
```

### 4.6 Page Object Model

Абстракция: page → отдельный класс с методами.

Плохо (везде By.id):
```java
driver.findElement(By.id("username")).sendKeys(...);
driver.findElement(By.id("submit")).click();
// разбросано по всем тестам
```

Хорошо (Page Object):
```java
public class LoginPage {
    private final WebDriver driver;

    @FindBy(id = "username") WebElement username;
    @FindBy(id = "password") WebElement password;
    @FindBy(id = "submit") WebElement submitBtn;

    public LoginPage(WebDriver driver) {
        this.driver = driver;
        PageFactory.initElements(driver, this);
    }

    public DashboardPage loginAs(String user, String pass) {
        username.sendKeys(user);
        password.sendKeys(pass);
        submitBtn.click();
        return new DashboardPage(driver);
    }
}

// в тесте:
LoginPage login = new LoginPage(driver);
DashboardPage dashboard = login.loginAs("berik", "secret");
assertThat(dashboard.isOpen()).isTrue();
```

Плюсы:
- Логика page — в одном месте.
- Тесты читаемы (business language).
- Изменение UI → правка одного Page Object.

### 4.7 Selenium Grid

Запускать тесты параллельно на много browser'ов / машин.

```
Hub → Node 1 (Chrome)
    → Node 2 (Firefox)
    → Node 3 (Safari)
```

Тесты параллельно ускоряют прогон.

---

## 5. Альтернативы Selenium

### 5.1 Playwright

Модерн от Microsoft. Быстрее, стабильнее.

```java
try (Playwright pw = Playwright.create()) {
    Browser browser = pw.chromium().launch();
    Page page = browser.newPage();
    page.navigate("https://example.com");
    page.locator("#username").fill("berik");
    page.locator("#submit").click();
    page.waitForSelector("#dashboard");
}
```

Плюсы:
- Auto-wait (не нужен explicit).
- Multi-language (Java, TS, Python, .NET).
- Быстрее.
- Network interception.

### 5.2 Cypress

JS-only, для frontend команд. Живёт в браузере, не через WebDriver.

Плюсы: быстрый, DX хороший.
Минусы: JS only, single tab.

### 5.3 Puppeteer

JS-only, для Chrome. Google.

---

## 6. API E2E тесты (без UI)

Часто проще тестировать через API, не UI.

### 6.1 REST-assured

```java
import static io.restassured.RestAssured.*;

@Test
void createOrder_returnsCreated() {
    given()
        .baseUri("https://api.example.com")
        .header("Authorization", "Bearer " + token)
        .contentType("application/json")
        .body("""
            {
              "customerId": "c1",
              "amount": 100
            }
            """)
    .when()
        .post("/api/orders")
    .then()
        .statusCode(201)
        .body("status", equalTo("NEW"))
        .body("total", equalTo(100.0))
        .header("Location", matchesRegex("/api/orders/\\d+"));
}
```

Читаемо. Стандарт для API E2E.

### 6.2 Karate

DSL для API tests. Один файл описывает scenario:

```gherkin
Feature: Order API

Scenario: Create order
    Given url baseUrl + '/api/orders'
    And request { customerId: 'c1', amount: 100 }
    When method POST
    Then status 201
    And match response.status == 'NEW'
```

Плюсы: BDD-style, не Java-код.
Минусы: свой DSL, не Java.

### 6.3 WireMock

Мок для внешних сервисов в тестах.

```java
@RegisterExtension
WireMockExtension wm = WireMockExtension.newInstance()
    .options(wireMockConfig().port(8089))
    .build();

@Test
void test() {
    wm.stubFor(get("/api/users/1")
        .willReturn(okJson("{\"id\":1,\"name\":\"berik\"}")));

    UserDto user = userClient.getUser(1L);

    assertThat(user.getName()).isEqualTo("berik");
    wm.verify(getRequestedFor(urlEqualTo("/api/users/1")));
}
```

---

## 7. Performance тесты

### 7.1 Виды

- **Load** — нормальная нагрузка; можем ли обслужить N req/sec?
- **Stress** — увеличиваем до отказа; сколько выдержит?
- **Soak / Endurance** — постоянная нагрузка часами; утечки, деградация?
- **Spike** — резкий всплеск; graceful?
- **Volume** — большие данные (терабайты).

### 7.2 SLA / SLO / SLI

- **SLA (Service Level Agreement)** — контракт с клиентом («99.9% uptime»).
- **SLO (Service Level Objective)** — цель («p99 latency < 500 ms»).
- **SLI (Service Level Indicator)** — измерение (реальные p99 = 320 ms).

Performance testing доказывает что SLO выполним.

### 7.3 Метрики

- **Throughput** — req/sec.
- **Latency percentiles** — p50, p95, p99, p99.9.
- **Error rate**.
- **Resource utilization** — CPU, memory, DB pool.

---

## 8. Инструменты performance

### 8.1 JMeter

Старейший, GUI + XML config.

Плюсы: много фич, plugins, готовые готовые sample.

Минусы: XML config неудобен, memory-hungry, старый UI.

### 8.2 Gatling

Scala DSL, современный.

```scala
val scn = scenario("Order flow")
  .exec(http("Login").post("/login").body(...))
  .exec(http("Create order").post("/api/orders").body(...))
  .exec(http("Get order").get("/api/orders/${id}"))

setUp(
  scn.inject(rampUsers(1000) during 60.seconds)
)
```

Плюсы: код в Scala, HTML отчёты, async engine (много concurrent).

### 8.3 k6

Modern, JS-scripting.

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
    vus: 100,
    duration: '30s',
};

export default function () {
    const res = http.get('https://api.example.com/orders');
    check(res, { 'status is 200': (r) => r.status === 200 });
}
```

Плюсы: простой JS, cloud (k6 Cloud), CI-friendly.

**В ИСНА** — k6 используется (из memory).

### 8.4 wrk / wrk2

CLI, простой. Быстрая проверка throughput.

```bash
wrk -t8 -c100 -d30s https://api.example.com/health
```

---

## 9. Chaos testing

Уже упоминал в `52-microservices-resilience.md`.

Ломаем намеренно (kill pods, slow network) → проверяем что система пережила.

Инструменты:
- **Chaos Monkey** (Netflix).
- **Chaos Mesh** (K8s).
- **Gremlin**.
- **Litmus**.

---

## 10. Security тесты

### 10.1 SAST (Static Application Security Testing)

Анализ **исходного кода** на уязвимости.

- **SonarQube**.
- **Checkmarx**.
- **Semgrep**.
- **Snyk Code**.

### 10.2 DAST (Dynamic)

Тестирование **работающего приложения** — SQL injection, XSS.

- **OWASP ZAP**.
- **Burp Suite**.

### 10.3 SCA (Software Composition Analysis)

Проверка **зависимостей** на известные CVE.

- **Snyk**.
- **OWASP Dependency-Check**.
- **Dependabot** (GitHub).
- **Trivy** (для Docker images).

### 10.4 Penetration testing

Ручное тестирование "белыми хакерами".

---

## 11. Real ИСНА cases

### 11.1 knp-e2e runner

Custom Java-based runner против живого стенда. Из memory:

- **`knp-e2e-runner-ops`** — как поднять и гонять flow-runner.
- **`knp-e2e-feign-smoke`** — синтетические Feign-смоки (проверка всех Feign-клиентов).
- **`knp-e2e-trigger-topology`** — как модули триггерят раннер (inline / E2E_LIST / E2E_SCOPE).
- **`knp-fo-frequency-tiering`** — правило 3/2/1 happy-кейсов для ФО.
- **`knp-e2e-liquidation-reversal-repro`** — flow воспроизводит баг переразноски.
- **`knp-e2e-prod-taxrep21-smoke-local-run`** — как локально гонять prod-smoke.

### 11.2 Правила scope

- `knp-e2e-prod-smoke-scope-rules` — не смокать мутирующие; только gap-endpoints.

### 11.3 gate-knp

Gate — проверка что тесты прошли перед merge. Реальный кейс `knp-e2e-runner-hikari-isolation-poisoning` — HikariCP отравлялся → gate краснел 18 мин; фикс — явный isolation.

### 11.4 e2e-gate-check-master race

Memory `knp-e2e-gate-check-master-race`: проверил гейт раньше чем release e2e дозавершился → Retry джобы.

---

## 12. CI/CD и тесты

### 12.1 Pipeline

```
commit
   │
   ▼
[unit + slice tests]     ← быстро, каждый commit
   │  ~5 min
   ▼
[integration tests]      ← @SpringBootTest + Testcontainers
   │  ~15 min
   ▼
[deploy to test env]
   │
   ▼
[smoke tests]            ← критичные user flows
   │  ~2 min
   ▼
[deploy to prod]
   │
   ▼
[production smoke]       ← post-deploy verify
```

### 12.2 Test parallelization

```gradle
test {
    maxParallelForks = 4
    forkEvery = 100
}
```

Каждый fork — отдельная JVM. Ускоряет прогон.

---

## 13. Best practices E2E

1. **Мало E2E** — только критичные paths.
2. **Test data setup** — известное, не полагайся на существующие.
3. **Cleanup** после тестов (или используй non-critical accounts).
4. **Явные waits**, никаких `Thread.sleep`.
5. **Page Object** для UI.
6. **Screenshots on failure** — что видел браузер.
7. **Video recording** для debug.
8. **Retry flaky** ограниченно (`maxAttempts = 3`) — но чинить причину.
9. **Timeout всё** — тест не должен висеть.
10. **Environment isolation** — не тестируй в prod прямыми записями.
11. **Мониторинг** прогонов (JUnit XML + Grafana Test Analytics).

---

## 14. Собесные вопросы

1. **Что такое E2E?** — Полная цепочка через все компоненты (UI/API → backend → БД).
2. **Что такое smoke test?** — Быстрая проверка что критичное работает после deploy.
3. **Selenium — как работает?** — Test → WebDriver API → ChromeDriver → реальный Chrome.
4. **Selenium waits — какие?** — Explicit (WebDriverWait), implicit (не рекомендуется), никогда Thread.sleep.
5. **Что такое Page Object Model?** — Абстракция: page → класс с методами; логика UI в одном месте.
6. **Selenium vs Playwright?** — Playwright новее, быстрее, auto-wait, multi-language.
7. **REST-assured — зачем?** — Fluent DSL для API E2E тестов.
8. **Load vs Stress vs Soak?** — Load: нормальная; stress: до отказа; soak: длительная (утечки).
9. **SLA vs SLO vs SLI?** — Agreement (контракт) / Objective (цель) / Indicator (измерение).
10. **Инструменты performance?** — JMeter, Gatling, k6, wrk.
11. **k6 — что за?** — JS-scripting, современный, CI-friendly.
12. **Что такое chaos engineering?** — Намеренная поломка prod (kill pods) для проверки resilience.
13. **SAST vs DAST?** — Static (код) vs Dynamic (running app).
14. **Как избежать flaky E2E?** — Explicit waits, не полагаться на порядок, изолированные данные, идемпотентные setup.
15. **WireMock — зачем?** — Mock external HTTP-сервисов в тестах.

---

## Итог

- **E2E** — критичные full-chain flows; **мало**, но важно.
- **Smoke** — быстрая post-deploy проверка.
- **Selenium/Playwright** для UI; **REST-assured** для API.
- **Page Object** обязательно для Selenium.
- **Explicit waits**, никогда Thread.sleep.
- **Performance**: JMeter/Gatling/k6.
- **SLA/SLO/SLI** — терминология уровня сервиса.
- **CI**: unit → integration → smoke → prod.
- В **ИСНА**: `knp-e2e` custom runner; prod-smoke по строгим правилам scope.

Следующий — `56-testing-best-practices.md`.
