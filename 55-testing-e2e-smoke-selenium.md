# 55. E2E, Smoke, Selenium, Performance testing

## Зачем нужны вершинные уровни pyramid

Разработчик который acquiring understanding testing hierarchy обычно останавливается на unit plus integration tests. Considers whole story. Production reality приносит другие категории проблемы. UI работает functionally но regression в CSS ломает business flow. Api tests зелёные но end-to-end user journey fails из-за timing issues между sync services. Load performance passes для one instance но degrades при deployment scale. All these выходят за scope unit и integration tests.

Разница между разработчиком «знающим unit/integration» и «понимающим full test spectrum» проявляется в production incident response. Первый видит user-reported bug и says «unit tests зелёные, must be user error». Второй знает что E2E tests catch integration bugs between services и UI, smoke tests catch deployment issues быстро, performance tests catch degradation при scale, chaos engineering catches resilience gaps. Знает что каждый level имеет своё место и свою cost — pyramid balances all of them.

В этом файле разберём вершину pyramid глубоко. E2E tests — real user journeys through полную систему. Smoke tests — быстрая post-deploy verification. Виды тестов по функционалу (functional vs non-functional). Selenium как industry standard для UI automation, mechanics plus caveats. Alternative UI tools — Playwright, Cypress. API E2E через REST-assured, Karate. Performance testing — load, stress, soak, spike. SLA/SLO/SLI terminology. Инструменты — JMeter, Gatling, k6. Chaos engineering. Security testing (SAST, DAST, SCA). Реальные КНП кейсы. CI/CD integration тестов.

## E2E: end-to-end tests

Полная цепочка через все компоненты. UI → backend → БД → external → response. Testing complete user journey не just individual pieces.

Пример scenario. Пользователь на сайте. Открывает knp.kgd.gov.kz. Логинится через ЭЦП. Открывает форму ФНО. Заполняет. Подписывает. Отправляет. Видит в списке отправленных.

E2E test делает всё это программно. Verifies что каждый шаг работает. Catches integration bugs invisible в unit/integration tests.

Зачем E2E. Ловит integration bugs missed в other levels. Проверяет critical user journeys. Regression на production-like environment. Business confidence — тесты show system works end-to-end.

Caveats E2E. Медленные — минуты на тест vs milliseconds для unit. Flaky — сеть, timing, UI-элементы могут vary. Дорого — поддержка plus infrastructure. Сложно отладить — что упало где в chain?

Правило pyramid. Мало E2E, только критичные paths. Не comprehensive coverage через E2E — too expensive и слишком slow.

## Smoke tests

Дым идёт? Быстрая проверка что базовые вещи работают.

Идея. После deploy — прогнать 5-20 критичных тестов за 1-5 минут. Если зелёные — всё OK, если красное — откатить.

Что проверять. Приложение отвечает (/actuator/health). Логин работает. Основной GET endpoint возвращает данные. POST одного ресурса создаётся.

Не проверять. Edge cases. Полные бизнес-процессы. Smoke = «критичное работает», не «всё работает».

В КНП — knp-e2e. Из memory. knp-e2e — раннер для e2e-тестов против живого стенда. Прод-смок (prod-smoke-*) — набор критичных проверок. Разные наборы для разных сервисов — prod-smoke-fno-ul, prod-smoke-fo-ul.

Правила пользователя (memory knp-e2e-prod-smoke-scope-rules). НЕ смокать мутирующие/опасные API (send/запись/запрос в ЛС-ИШ-ОС под ЭЦП владельца). Только gap-эндпоинты — покрывать отсутствующие сейчас, потом только safe GET.

Синтетический мониторинг. Smoke прогоны каждые N минут (Grafana Synthetics, Pingdom, Datadog Synthetics). Даёт alert если что-то упало между полными деплоями.

Continuous synthetic monitoring разительно улучшает detection outages. Traditional monitoring reports когда metrics deviate. Synthetic monitoring proactively tests actual user paths.

## Виды тестов по функционалу

Functional tests. Проверяют что функция работает как ожидается. Unit/integration/E2E — все могут быть functional. Behavior verification.

Non-functional. Different aspects beyond «works correctly».

Performance — скорость. How fast operations complete.

Load — сколько req/sec система обрабатывает. Normal expected traffic testing.

Stress — до какого предела. Increasing load until breaking point.

Soak — работает ли долго (утечки). Sustained load для hours detecting memory leaks, resource exhaustion.

Spike — как переживёт резкий всплеск. Sudden traffic surge handling.

Security — уязвимости. Attack vectors testing.

Usability — удобство UI. Human-centered evaluation.

Accessibility — a11y (accessibility for disabled users). WCAG compliance testing.

## Selenium для UI автоматизации

Selenium WebDriver — стандарт для UI automation. Long-established, widely used.

Как работает:
```
Test code (Java) → Selenium WebDriver API → Browser (ChromeDriver / GeckoDriver) → реальный Chrome / Firefox
```

Программно управляешь браузером. Клики, ввод, чтение элементов. Real browser behavior.

Зависимости:
```gradle
testImplementation 'org.seleniumhq.selenium:selenium-java:4.16.0'
testImplementation 'io.github.bonigarcia:webdrivermanager:5.6.0'
```

WebDriverManager автоматически downloads correct browser driver.

Базовый пример:
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

Headless mode (--headless) — без visual browser. Faster, works в CI environments без displays.

Локаторы для finding elements. By.id("username") — most preferred, stable. By.name("email"). By.className("btn-primary"). By.cssSelector("input[type='text']"). By.xpath("//button[contains(text(), 'Submit')]"). By.linkText("Log out").

Правило локатор priority. id > css > xpath. Xpath ломкий — DOM changes break selectors easily.

Waits критически важны для reliable tests.

Fatal error. Thread.sleep(5000) — flaky. Sometimes слишком мало, sometimes слишком много. Timing depends на environment.

Правильно — explicit waits:
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

Wait для actual condition вместо fixed time. Tests complete when ready instead of waiting fixed duration. Faster и more reliable.

Implicit wait — глобальный:
```java
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
```

Не рекомендуется. Скрывает проблемы (unclear waiting behavior). Prefer explicit waits.

## Page Object Model

Абстракция page → отдельный класс с методами. Design pattern для maintainable UI tests.

Плохо (везде By.id):
```java
driver.findElement(By.id("username")).sendKeys(...);
driver.findElement(By.id("submit")).click();
// selectors разбросаны по всем тестам
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

Плюсы. Логика page — в одном месте. Тесты читаемы (business language). Изменение UI → правка одного Page Object. Multiple tests reuse same Page Object.

Standard practice для serious UI testing. Reduces maintenance burden significantly.

## Selenium Grid

Запускать тесты параллельно на много browser'ов и машин:
```
Hub → Node 1 (Chrome)
    → Node 2 (Firefox)
    → Node 3 (Safari)
```

Тесты параллельно ускоряют прогон. Cross-browser testing enabled. Different browsers detect different bugs sometimes.

Enterprise Selenium setups use Grid либо cloud services (Sauce Labs, BrowserStack) для same effect.

## Альтернативы Selenium

Playwright — modern от Microsoft. Быстрее, стабильнее:
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

Плюсы. Auto-wait (не нужен explicit). Multi-language (Java, TS, Python, .NET). Быстрее. Network interception. Better reliability чем Selenium.

Gaining popularity. Consider для new projects.

Cypress. JS-only, для frontend команд. Живёт в браузере, не через WebDriver.

Плюсы. Быстрый. DX хороший. Time-travel debugging. Automatic waiting.

Минусы. JS only (frontend teams). Single tab (limitation). Не cross-browser все.

Puppeteer. JS-only, для Chrome. Google product. Precursor Playwright.

## API E2E tests

Часто проще тестировать через API, не UI. Faster, more reliable, easier to maintain.

REST-assured:
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

Читаемо. Стандарт для API E2E тестов. Fluent DSL familiar most Java developers.

Karate. DSL для API tests. Один файл описывает scenario:
```gherkin
Feature: Order API

Scenario: Create order
    Given url baseUrl + '/api/orders'
    And request { customerId: 'c1', amount: 100 }
    When method POST
    Then status 201
    And match response.status == 'NEW'
```

Плюсы. BDD-style, не Java-код. Readable business stakeholders.

Минусы. Свой DSL, не Java. Learning curve. Less flexibility.

WireMock. Мок для внешних сервисов в тестах:
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

Simulates HTTP downstream в tests. Verifies что expected calls made.

## Performance testing

Виды tests.

Load — нормальная нагрузка. Can we handle N req/sec? Baseline verification.

Stress — увеличиваем до отказа. Сколько выдержит? Finding limits.

Soak / Endurance — постоянная нагрузка часами. Утечки, деградация? Memory leaks, resource exhaustion.

Spike — резкий всплеск. Graceful? Handling sudden traffic increases.

Volume — большие данные (терабайты). Testing с data at scale.

## SLA / SLO / SLI

SLA (Service Level Agreement) — контракт с клиентом. 99.9% uptime. Formal commitment. Consequences для violations (financial penalties, credits).

SLO (Service Level Objective) — цель. p99 latency less than 500 ms. Internal target. Guides operations decisions.

SLI (Service Level Indicator) — измерение. Реальные p99 = 320 ms. Actual observed metric.

Relationship. SLIs measure. SLOs targets. SLAs contractual commitments. SLIs continuously monitored, SLO thresholds trigger alerting, SLA violations trigger business consequences.

Performance testing доказывает что SLO выполним. Не replaces monitoring но validates система can meet targets under expected load.

## Метрики performance

Throughput — req/sec. How much traffic system serves.

Latency percentiles — p50 (median), p95, p99, p99.9. Distribution of response times. Not average — average hides outliers.

Error rate — процент failed responses. Higher под load обычно.

Resource utilization — CPU, memory, DB pool. Where bottleneck resides.

Combined metrics tell full story. Throughput high, latency low, errors near zero, utilization sustainable = healthy. Any degradation reveals problem area.

## Инструменты performance

JMeter. Старейший, GUI plus XML config. Много features, plugins, готовые sample scripts.

Минусы. XML config неудобен. Memory-hungry. Старый UI.

Gatling. Scala DSL, современный:
```scala
val scn = scenario("Order flow")
  .exec(http("Login").post("/login").body(...))
  .exec(http("Create order").post("/api/orders").body(...))
  .exec(http("Get order").get("/api/orders/${id}"))

setUp(
  scn.inject(rampUsers(1000) during 60.seconds)
)
```

Плюсы. Код в Scala. HTML отчёты beautiful. Async engine (много concurrent).

k6. Modern, JS-scripting:
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

Плюсы. Простой JS. Cloud (k6 Cloud) для distributed load. CI-friendly. Modern developer experience.

В КНП — k6 используется (из memory).

wrk / wrk2. CLI, простой. Быстрая проверка throughput:
```bash
wrk -t8 -c100 -d30s https://api.example.com/health
```

Simple но powerful для quick benchmarks.

## Chaos engineering

Ломаем намеренно. Kill pods, slow network → проверяем что система пережила. Discussed в файле 52.

Инструменты. Chaos Monkey (Netflix). Chaos Mesh (K8s). Gremlin. Litmus.

Chaos testing verifies resilience patterns work в practice. Не just theoretical — actually recovers from failures.

## Security testing

SAST (Static Application Security Testing). Анализ исходного кода на уязвимости. Not running application — static analysis.

Инструменты. SonarQube. Checkmarx. Semgrep. Snyk Code. Integrated в CI pipeline typically.

DAST (Dynamic Application Security Testing). Тестирование работающего приложения. SQL injection, XSS attempts.

Инструменты. OWASP ZAP. Burp Suite. Automated crawling plus vulnerability probing.

SCA (Software Composition Analysis). Проверка зависимостей на известные CVE.

Инструменты. Snyk. OWASP Dependency-Check. Dependabot (GitHub). Trivy (для Docker images). Automated scanning against vulnerability databases.

Penetration testing. Ручное тестирование "белыми хакерами". Human expertise finding vulnerabilities automated tools miss. Periodic engagements.

## Real ИСНА cases

knp-e2e runner — custom Java-based runner против живого стенда. Из memory. knp-e2e-runner-ops — как поднять и гонять flow-runner. knp-e2e-feign-smoke — синтетические Feign-смоки (проверка всех Feign-клиентов). knp-e2e-trigger-topology — как модули триггерят раннер (inline / E2E_LIST / E2E_SCOPE). knp-fo-frequency-tiering — правило 3/2/1 happy-кейсов для ФО. knp-e2e-liquidation-reversal-repro — flow воспроизводит баг переразноски. knp-e2e-prod-taxrep21-smoke-local-run — как локально гонять prod-smoke.

Правила scope. knp-e2e-prod-smoke-scope-rules — не смокать мутирующие; только gap-endpoints.

gate-knp. Gate — проверка что тесты прошли перед merge. Реальный кейс knp-e2e-runner-hikari-isolation-poisoning — HikariCP отравлялся, gate краснел 18 минут. Fix — явный isolation setting.

e2e-gate-check-master race. Memory knp-e2e-gate-check-master-race — проверил гейт раньше чем release e2e дозавершился → Retry джобы.

Все эти cases показывают что testing infrastructure сложна и evolves с системой. Custom tooling часто необходимо для specific needs.

## CI/CD и тесты

Pipeline stages:
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

Каждый stage gates progression. Fail — stop pipeline. Investigate.

Test parallelization для speed:
```gradle
test {
    maxParallelForks = 4
    forkEvery = 100
}
```

Каждый fork — отдельная JVM. Ускоряет прогон. Trade-off — more resource intensive.

## Best practices E2E

Мало E2E — только критичные paths. Comprehensive coverage через unit / integration.

Test data setup — известное, не полагайся на existing. Reproducible test data important.

Cleanup после тестов (или используй non-critical accounts). Don't pollute test environment.

Явные waits, никаких Thread.sleep. Reliability requires explicit conditions.

Page Object для UI. Maintainability critical для UI tests.

Screenshots on failure — что видел браузер. Debugging visual issues requires seeing state.

Video recording для debug. Some frameworks support (Playwright). Priceless for flaky test investigation.

Retry flaky ограниченно (maxAttempts = 3) — но чинить причину. Retry hides symptoms — fix underlying cause.

Timeout всё — тест не должен висеть. Ensure tests complete в bounded time.

Environment isolation — не тестируй в prod прямыми записями. Sandbox environments preferred.

Мониторинг прогонов (JUnit XML plus Grafana Test Analytics). Test flakiness metrics tracked over time.

## Итоги

E2E тесты critical для critical user journeys. Мало но important. Full-chain verification.

Smoke тесты для post-deploy verification. Быстро (1-5 min). Прогон критичного.

В КНП — knp-e2e custom runner. prod-smoke по строгим правилам scope (не мутирующие, gap-endpoints).

Виды тестов по функционалу. Functional (behavior). Non-functional (performance, load, stress, soak, spike, security, usability, accessibility).

Selenium — industry standard для UI. Explicit waits обязательно. Page Object Model для maintainability. Grid для parallel execution.

Alternatives Selenium. Playwright более modern. Cypress для frontend teams (JS only).

API E2E — REST-assured (fluent DSL), Karate (BDD-style), WireMock (mocking external HTTP).

Performance testing. JMeter, Gatling, k6. Load, stress, soak, spike, volume different concerns.

SLA / SLO / SLI terminology. Contract, target, measurement respectively.

Метрики. Throughput, latency percentiles, error rate, resource utilization. Combined picture.

Chaos engineering для verification resilience. Ломаем намеренно, наблюдаем recovery.

Security testing. SAST (static code), DAST (running app), SCA (dependencies), penetration testing (human expertise).

CI/CD pipeline stages — unit, integration, smoke, production smoke. Each gates progression.

Real КНП cases показывают что testing infrastructure evolves с системой. Custom tooling часто необходимо.

Best practices E2E. Мало tests. Isolated data. Explicit waits. Page Object. Screenshots. Timeouts. Fix flaky.

Testing spectrum от unit to E2E to chaos to security обеспечивает production confidence. Pyramid balance keeps costs sustainable.
