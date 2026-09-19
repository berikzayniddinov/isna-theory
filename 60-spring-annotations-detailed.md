# 60. Главные аннотации Spring / Spring Boot

Каждая ключевая аннотация: что делает, как реализована под капотом.

---

## 1. Как Spring обрабатывает аннотации в целом

Механизм: **BeanPostProcessor + BeanFactoryPostProcessor**.

- **BeanFactoryPostProcessor** — модифицирует определения бинов (BeanDefinition) до их создания. Пример: `ConfigurationClassPostProcessor` парсит `@Configuration`.
- **BeanPostProcessor** — модифицирует бины после создания. Пример: `AutowiredAnnotationBeanPostProcessor` обрабатывает `@Autowired`.

Плюс **проксирование** (CGLib / JDK Dynamic Proxy) для `@Transactional`, `@Async`, `@Cacheable`.

---

## 2. @SpringBootApplication

Композиция трёх:
```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication { ... }
```

- **@SpringBootConfiguration** = `@Configuration` (специальный маркер для Spring Boot Test).
- **@EnableAutoConfiguration** — включает автоконфигурацию (см. §11).
- **@ComponentScan** — сканирует пакет + подпакеты.

**Правило**: класс с этой аннотацией — в **корневом пакете**, чтобы ComponentScan охватил всё.

Под капотом: обработчики автоконфига читают `spring.factories` (Boot 2) / `AutoConfiguration.imports` (Boot 3), applyят условно.

---

## 3. Стереотипы: @Component, @Service, @Repository, @Controller

Все — маркеры «этот класс — bean, регистрируй». Функционально почти одинаковы.

- **@Component** — базовый.
- **@Service** — семантика: business logic (bean-slice).
- **@Repository** — семантика: data access. **Плюс** — Spring оборачивает исключения в `DataAccessException` через `PersistenceExceptionTranslationPostProcessor`.
- **@Controller** — MVC-контроллер, возвращает view names.
- **@RestController** = `@Controller` + `@ResponseBody` (JSON everywhere).

### 3.1 Реализация

`ClassPathBeanDefinitionScanner` сканирует пакет, ищет классы с этими аннотациями (через meta-annotation `@Component`), регистрирует как **BeanDefinition**.

Дальше — обычная фабрика бинов.

---

## 4. @Configuration

Класс с `@Bean`-методами.

```java
@Configuration
class AppConfig {
    @Bean DataSource dataSource() { ... }
    @Bean JdbcTemplate jdbc(DataSource ds) { return new JdbcTemplate(ds); }
}
```

### 4.1 Реализация

`ConfigurationClassPostProcessor` (BeanFactoryPostProcessor):
1. Находит все `@Configuration` классы.
2. Читает их `@Bean` методы.
3. Регистрирует каждый метод как BeanDefinition (factory method).

### 4.2 CGLib proxy

`@Configuration` класс **оборачивается в CGLib proxy**. Зачем?

```java
@Configuration
class Cfg {
    @Bean DataSource ds() { return new HikariDataSource(...); }
    @Bean JdbcTemplate jdbc() {
        return new JdbcTemplate(ds());   // ← если бы не прокси, было бы второй DataSource
    }
}
```

Прокси перехватывает `ds()` внутри `jdbc()` → возвращает **тот же** singleton, не создаёт новый.

Отключить (быстрее старт, но без гарантии singleton внутри):
```java
@Configuration(proxyBeanMethods = false)
```

---

## 5. @Bean

Метод как factory для bean.

```java
@Bean
DataSource dataSource() {
    return new HikariDataSource(...);
}

@Bean(name = "mySpecialDs")
DataSource specialDs() { ... }

@Bean(initMethod = "init", destroyMethod = "close")
MyBean myBean() { ... }
```

Использование:
- Сторонние классы (не можешь навесить `@Component`).
- Условная логика создания.
- Auto-configuration.

Имя bean = имя метода (или явно через `@Bean(name = "...")`).

---

## 6. @Autowired

Внедрение зависимости.

```java
@Service
class OrderService {
    @Autowired FnoRepository repo;                    // field injection (плохо!)

    @Autowired
    public OrderService(FnoRepository repo) { ... }   // constructor (best)

    @Autowired
    public void setRepo(FnoRepository repo) { ... }   // setter
}
```

Правило: **constructor injection**. С Spring 4.3+ на **единственном** конструкторе `@Autowired` не нужен.

### 6.1 Реализация

**`AutowiredAnnotationBeanPostProcessor`** (BeanPostProcessor):
1. При создании bean — сканирует поля/сеттеры/конструкторы с `@Autowired`.
2. Для каждой зависимости — резолвит через `BeanFactory.getBean(type)`.
3. Injectит.

### 6.2 @Autowired(required = false)

Не падать если нет bean:
```java
@Autowired(required = false) SomeOptional optional;
```

Или через `Optional<T>`:
```java
@Autowired Optional<SomeOptional> optional;
```

---

## 7. @Qualifier

Уточняет какой bean, когда несколько кандидатов:

```java
@Bean("fast") FnoService fast() { ... }
@Bean("slow") FnoService slow() { ... }

@Autowired @Qualifier("fast") FnoService svc;
```

---

## 8. @Primary

Метка «этот по умолчанию»:

```java
@Bean @Primary FnoService fast() { ... }
@Bean FnoService slow() { ... }

@Autowired FnoService svc;   // → fast (потому что @Primary)
```

---

## 9. @Lazy

Bean создаётся не при старте, а при первом использовании.

```java
@Component @Lazy
class ExpensiveBean { ... }
```

Или на injection:
```java
@Autowired @Lazy ExpensiveBean bean;   // прокси, реальный при первом вызове
```

Использование:
- Тяжёлые бины (не создавать если не нужны).
- Разрешение циклических зависимостей.

---

## 10. @Scope

Область жизни bean:
```java
@Component @Scope("prototype") class FnoBuilder { }
@Component @Scope("request") class RequestContext { }   // web-only
```

Scopes: `singleton` (default), `prototype`, `request`, `session`, `application`.

Реализация: `BeanFactory.getBean()` каждый раз для prototype; кэш для singleton.

---

## 11. @EnableAutoConfiguration

Магия Spring Boot. Автоматически конфигурирует бины по classpath.

### 11.1 Реализация

При старте:
1. Читаются файлы `META-INF/spring.factories` (Boot 2) или `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 3) из всех jar'ов.
2. Получается список auto-configuration классов.
3. Каждый — `@Configuration` с `@Conditional*` условиями.
4. Если условия выполнены — бины регистрируются.

### 11.2 Отключение

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```

Или через properties:
```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## 12. @Conditional*

Регистрация bean по условию.

```java
@Bean
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean
@ConditionalOnProperty(name = "spring.datasource.url")
DataSource dataSource() { ... }
```

Виды:
- **@ConditionalOnClass** — есть класс в classpath.
- **@ConditionalOnMissingClass** — нет.
- **@ConditionalOnBean** — есть bean этого типа.
- **@ConditionalOnMissingBean** — нет (юзер не определил свой).
- **@ConditionalOnProperty(name, havingValue, matchIfMissing)** — по property.
- **@ConditionalOnWebApplication** — web-context.
- **@ConditionalOnJava** — Java version.
- **@Conditional(MyCondition.class)** — custom.

Основа Spring Boot auto-configuration.

---

## 13. @ComponentScan

Сканирование classpath на bean'ы.

```java
@ComponentScan(basePackages = {"kz.gov.kgd.isna.knp", "kz.gov.kgd.isna.commons"})
class Config { }
```

По умолчанию (внутри `@SpringBootApplication`) — сканирует **пакет класса с аннотацией + подпакеты**.

Реализация: `ClassPathScanningCandidateComponentProvider`.

---

## 14. @Value

Injection value из configuration.

```java
@Value("${server.port}") int port;
@Value("${app.name:defaultName}") String appName;
@Value("#{systemProperties['user.name']}") String user;   // SpEL
@Value("${servers}") List<String> servers;                  // comma-separated
```

Реализация: `Environment.getProperty(...)` + placeholder resolution.

---

## 15. @ConfigurationProperties

Группа properties как typed bean.

```java
@ConfigurationProperties(prefix = "knp")
@Component
public class KnpProperties {
    private int maxBatchSize = 100;
    private String outerSystemUrl;
    // getters/setters
}
```

Уже разбирали в `57-spring-boot-config-detailed.md`.

Реализация: `ConfigurationPropertiesBinder` — reflection + type conversion.

---

## 16. @Profile

Bean/config активен только при активном profile:

```java
@Configuration @Profile("prod") class ProdConfig { }
@Component @Profile({"dev", "test"}) class DevOnlyBean { }
@Component @Profile("!prod") class NonProdBean { }
```

Реализация: `ProfileCondition` (частный случай `@Conditional`).

---

## 17. @EnableXxx аннотации

Spring/Boot использует много enabler'ов:

- **@EnableTransactionManagement** — включает `@Transactional` через AOP.
- **@EnableAsync** — включает `@Async`.
- **@EnableScheduling** — включает `@Scheduled`.
- **@EnableCaching** — включает `@Cacheable`.
- **@EnableAspectJAutoProxy** — включает AOP.
- **@EnableConfigurationProperties(Xxx.class)** — регистрирует @ConfigurationProperties bean.
- **@EnableWebSecurity** — Security config.
- **@EnableJpaRepositories** — Spring Data JPA.
- **@EnableFeignClients** — Feign.

Механика: `@Import` → загружает specific configuration classes.

В Spring Boot большинство `@Enable` автоматически (не нужны явно) — auto-configuration включает.

---

## 18. @Async

Метод выполняется в **другом потоке**.

```java
@Service
class NotificationService {
    @Async
    public void sendEmail(String to) {
        // выполнится на TaskExecutor
    }

    @Async
    public CompletableFuture<String> asyncCompute() {
        return CompletableFuture.completedFuture("result");
    }
}
```

Требует `@EnableAsync`.

### 18.1 Реализация

AOP-прокси перехватывает вызов → сабмитит в `TaskExecutor` → возвращает `Future` (или void).

### 18.2 Кавет

- **Self-invocation** — та же проблема, что @Transactional.
- Void методы — не узнаешь про exception; используй `CompletableFuture`.
- По default пул — `SimpleAsyncTaskExecutor` (создаёт thread на вызов!) — плохо для нагрузки; настрой свой:
  ```java
  @Bean(name = "taskExecutor")
  TaskExecutor executor() {
      var e = new ThreadPoolTaskExecutor();
      e.setCorePoolSize(10);
      e.setMaxPoolSize(50);
      e.setQueueCapacity(200);
      e.setThreadNamePrefix("async-");
      e.initialize();
      return e;
  }
  ```

---

## 19. @Scheduled

Cron / periodic tasks.

```java
@Scheduled(fixedRate = 5000)              // каждые 5 сек
public void every5Sec() { ... }

@Scheduled(fixedDelay = 3000)             // 3 сек между окончанием и стартом
public void everyOther() { ... }

@Scheduled(cron = "0 0 3 * * *")          // каждый день в 3:00
public void daily() { ... }

@Scheduled(cron = "${schedule.cron}")     // из yml
public void configurable() { ... }
```

Требует `@EnableScheduling`.

### 19.1 Реализация

`ScheduledAnnotationBeanPostProcessor` находит `@Scheduled` методы, регистрирует в `TaskScheduler`.

### 19.2 Кавет multi-instance

Если приложение в K8s с 3 репликами — все 3 запустят cron параллельно.

Fix: **ShedLock** — distributed lock.

```java
@Scheduled(cron = "...")
@SchedulerLock(name = "myTask", lockAtMostFor = "PT30S")
public void task() { ... }
```

Реальный ИСНА-кейс `knp-fno21-shedlock-stale-image-dup-regnum` — без ShedLock дубли.

---

## 20. @Transactional

Обёртка транзакций через AOP-proxy.

Разбирали детально в файлах 32-35.

```java
@Transactional
public void save(Order o) { ... }

@Transactional(readOnly = true, timeout = 10, propagation = REQUIRES_NEW,
    rollbackFor = Exception.class)
public Order read(Long id) { ... }
```

---

## 21. @Cacheable

Кэширование результата метода.

```java
@Cacheable("orders")
public Order getOrder(Long id) { ... }

@Cacheable(value = "orders", key = "#id", condition = "#id > 0", unless = "#result == null")
public Order getOrder(Long id) { ... }

@CacheEvict(value = "orders", key = "#o.id")
public void deleteOrder(Order o) { ... }

@CachePut(value = "orders", key = "#o.id")
public Order updateOrder(Order o) { ... }
```

Требует `@EnableCaching` + cache-manager (Hazelcast, Caffeine, Redis).

Через AOP-proxy.

**Self-invocation** — та же проблема.

---

## 22. @PostConstruct / @PreDestroy

Lifecycle-хуки bean.

```java
@Component
class MyBean {
    @PostConstruct
    void init() {
        // после injection
    }

    @PreDestroy
    void cleanup() {
        // перед destroy
    }
}
```

С Spring 6 / Java 9+ — из пакета `jakarta.annotation.*` (не `javax.annotation.*`).

Реализация: `CommonAnnotationBeanPostProcessor`.

**Кавет**: `@Transactional` НЕ работает в `@PostConstruct` (bean ещё не проксирован).

---

## 23. @EventListener

Подписка на events.

```java
@Component
class OrderListener {
    @EventListener
    public void handle(OrderCreated event) { ... }

    @EventListener(condition = "#event.status == 'NEW'")
    public void handleNew(OrderCreated event) { ... }
}
```

Синхронно (в том же потоке что publishEvent).

С `@Async` — async.

### 23.1 @TransactionalEventListener

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void afterCommit(OrderCreated event) { ... }
```

Только после успешного commit tx.

Разбирали в `34-transactional-advanced.md`.

---

## 24. @RestController vs @Controller

- **@Controller** — MVC, возвращает view name (Thymeleaf/JSP).
- **@RestController** = `@Controller` + `@ResponseBody` на всех методах.

```java
@Controller
class OrderController {
    @GetMapping("/orders")
    public String list(Model model) {
        model.addAttribute("orders", ...);
        return "orders";   // → orders.html
    }
}

@RestController
class OrderRestController {
    @GetMapping("/api/orders")
    public List<Order> list() {
        return svc.findAll();   // → JSON
    }
}
```

Обычно микросервисы — `@RestController`.

---

## 25. @RequestMapping и его варианты

```java
@RequestMapping(value = "/api/orders", method = RequestMethod.GET)
public List<Order> list() { }

// или короче
@GetMapping("/api/orders")
public List<Order> list() { }

@PostMapping("/api/orders")
public Order create(@RequestBody OrderDto dto) { }

@PutMapping("/api/orders/{id}")
public Order update(@PathVariable Long id, @RequestBody OrderDto dto) { }

@DeleteMapping("/api/orders/{id}")
public void delete(@PathVariable Long id) { }

@PatchMapping("/api/orders/{id}")
public Order patch(@PathVariable Long id, @RequestBody Map<String, Object> updates) { }
```

Аннотации на классе — префикс для всех методов:
```java
@RestController
@RequestMapping("/api/orders")
class OrderController {
    @GetMapping("/{id}") ...   // /api/orders/{id}
}
```

---

## 26. Параметры контроллера

```java
@GetMapping("/orders/{id}")
public Order get(@PathVariable Long id) { }               // URL path

@GetMapping("/orders")
public List<Order> search(@RequestParam String status,
                          @RequestParam(defaultValue = "0") int page,
                          @RequestParam(required = false) String customerId) { }

@PostMapping("/orders")
public Order create(@RequestBody @Valid OrderDto dto,
                    @RequestHeader("X-Request-Id") String reqId) { }

@GetMapping("/user")
public User currentUser(@AuthenticationPrincipal Jwt jwt) { }
```

---

## 27. @Valid / @Validated

Bean Validation (JSR-380):
```java
class OrderDto {
    @NotBlank String customerId;
    @Min(1) int amount;
    @Email String contactEmail;
}

@PostMapping("/orders")
public Order create(@Valid @RequestBody OrderDto dto) { }
```

При invalid → `MethodArgumentNotValidException` → 400.

---

## 28. @ExceptionHandler / @ControllerAdvice

Централизованная обработка exceptions.

```java
@ControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorDto> notFound(EntityNotFoundException e) {
        return ResponseEntity.status(404).body(new ErrorDto(e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorDto> validation(MethodArgumentNotValidException e) {
        return ResponseEntity.badRequest().body(new ErrorDto("validation failed"));
    }
}
```

`@ControllerAdvice` — global (все controllers) или scoped:
```java
@ControllerAdvice(basePackages = "kz.gov.kgd.isna.knp.api")
```

---

## 29. Другие полезные

### 29.1 @Order

Приоритет beans при инжекции коллекций / автопрокси:
```java
@Component @Order(1) class Handler1 { }
@Component @Order(2) class Handler2 { }

@Autowired List<Handler> handlers;   // → [Handler1, Handler2]
```

### 29.2 @Import

Импорт additional Configuration classes:
```java
@Configuration
@Import({DbConfig.class, SecurityConfig.class})
class AppConfig { }
```

Основа `@EnableXxx` аннотаций.

### 29.3 @PropertySource

Дополнительный source properties:
```java
@Configuration
@PropertySource("classpath:custom.properties")
class Config { }
```

Устарело в пользу yml + profile-specific.

### 29.4 @Retryable (Spring Retry)

Retry метода при exception:
```java
@Retryable(retryFor = IOException.class, maxAttempts = 3, backoff = @Backoff(delay = 1000))
public String call() { ... }
```

---

## 30. Собесные вопросы

1. **Что делает @SpringBootApplication?** — Три в одной: `@SpringBootConfiguration`, `@EnableAutoConfiguration`, `@ComponentScan`.
2. **Разница @Component/@Service/@Repository?** — Семантика + `@Repository` даёт exception translation.
3. **Что делает @Configuration?** — Класс с @Bean методами; проксируется CGLib для singleton гарантии.
4. **Что делает @Bean?** — Метод как factory для bean.
5. **Как работает @Autowired?** — `AutowiredAnnotationBeanPostProcessor` через reflection; резолвит по типу.
6. **Разница @Primary и @Qualifier?** — Primary: этот default; Qualifier: уточнить какой.
7. **Что делает @Lazy?** — Bean создаётся при первом использовании (не при старте).
8. **@Scope prototype — когда?** — Новый экземпляр каждый раз; для stateful/mutable объектов.
9. **Как работает @EnableAutoConfiguration?** — Читает `spring.factories` / `AutoConfiguration.imports`, применяет `@Conditional*` условия.
10. **Что такое @ConditionalOnXxx?** — Регистрация bean по условию (класс в classpath, bean отсутствует, property установлено).
11. **@Value vs @ConfigurationProperties?** — Value: одно свойство; ConfigurationProperties: типизированная группа.
12. **@Profile — как работает?** — `ProfileCondition` (частный @Conditional) на активном profile.
13. **@Async — реализация?** — AOP-proxy сабмитит в TaskExecutor; кавет self-invocation.
14. **@Scheduled + K8s — проблема?** — Все реплики запускают → ShedLock для distributed lock.
15. **@Transactional — как?** — AOP-proxy (CGLib) с TransactionInterceptor.
16. **@RestController vs @Controller?** — RestController = Controller + @ResponseBody на всех методах.
17. **@Valid — где работает?** — На параметрах методов (контроллеров) с типом DTO.
18. **@ControllerAdvice?** — Глобальный @ExceptionHandler для всех controllers.
19. **@PostConstruct — когда?** — После injection всех зависимостей, до готовности bean.
20. **@Transactional в @PostConstruct — работает?** — Нет; bean ещё не проксирован.

---

## Итог

- **@SpringBootApplication** = 3-в-1.
- **Stereotypes** = маркеры для регистрации.
- **@Configuration** + **@Bean** = ручные фабрики.
- **@Autowired** через `AutowiredAnnotationBeanPostProcessor`.
- **@EnableAutoConfiguration** + **@Conditional*** = магия Boot.
- **@Value** / **@ConfigurationProperties** для конфига.
- **@Async / @Scheduled / @Transactional / @Cacheable** через **AOP-proxy** (все страдают self-invocation).
- **@RestController** для REST APIs; **@ControllerAdvice** для error handling.

Следующий — `61-spring-mvc-controllers-internals.md`.
