# 60. Главные аннотации Spring и Spring Boot: internals каждой

## Зачем понимать аннотации глубже уровня «работает»

Разработчик который начинает Spring обычно использует аннотации как magic. @Service marks bean, @Autowired injects dependency, @Transactional wraps transaction. Работает — moving on. Реальные production issues требуют understanding как эти аннотации реально processed. Почему @Transactional не срабатывает when called from same class? Почему @Autowired циклическая dependency ломает startup? Почему @Configuration класс должен быть CGLib proxied? Почему @Async return void не ловит exceptions?

Разница между разработчиком «использующим аннотации» и «понимающим их internals» проявляется в troubleshooting сложных scenarios. Первый видит «self-invocation issue» и searches Stack Overflow для fix. Второй знает что @Transactional работает через AOP-proxy (CGLib для class, JDK для interface). Знает что calling method through this bypasses proxy. Знает что @Configuration класс обёрнут в CGLib proxy чтобы @Bean method calls внутри same class returned same singleton instance. Знает что @Autowired обрабатывается AutowiredAnnotationBeanPostProcessor через reflection. Knowledge internals enables predicting behavior plus diagnosing issues.

В этом файле разберём все ключевые Spring аннотации с deep dive в implementation. Как Spring обрабатывает аннотации в общем — BeanPostProcessor, BeanFactoryPostProcessor. @SpringBootApplication composition. Stereotypes (@Component, @Service, @Repository, @Controller). @Configuration и CGLib proxy для @Bean methods. @Autowired через reflection. @Qualifier, @Primary, @Lazy. @Scope. @EnableAutoConfiguration магия. @Conditional* family. @Value plus @ConfigurationProperties. @Profile. @Enable* pattern. @Async, @Scheduled, @Transactional (AOP-proxy family). @Cacheable. @PostConstruct/@PreDestroy. @EventListener. Web annotations. @Valid, @ControllerAdvice. Utility annotations.

## Как Spring обрабатывает аннотации в целом

Механизм основывается на двух key extension points.

BeanFactoryPostProcessor — модифицирует определения бинов (BeanDefinition) до их создания. Пример — ConfigurationClassPostProcessor парсит @Configuration классы. Runs early в container lifecycle. Modifies bean registration information.

BeanPostProcessor — модифицирует бины после создания. Пример — AutowiredAnnotationBeanPostProcessor обрабатывает @Autowired поля. Runs during bean instantiation. Wraps beans или injects dependencies.

Плюс проксирование (CGLib / JDK Dynamic Proxy) для @Transactional, @Async, @Cacheable. AOP-driven behaviors implemented через proxy interception.

Together these mechanisms обеспечивают declarative programming model Spring. Annotations mark intentions, framework processes them at appropriate lifecycle stages.

## @SpringBootApplication

Композиция трёх аннотаций:
```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication { ... }
```

@SpringBootConfiguration = @Configuration (специальный маркер для Spring Boot Test — different testing frameworks recognize).

@EnableAutoConfiguration — включает автоконфигурацию (см. section 11).

@ComponentScan — сканирует пакет plus подпакеты. Default — package of annotated class.

Правило. Класс с этой аннотацией в корневом пакете чтобы ComponentScan охватил всё. Package structure aligned с scanning expectations.

Под капотом. Обработчики автоконфига читают META-INF/spring.factories (Boot 2) или META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports (Boot 3) из всех jar'ов classpath. Apply conditional logic для each auto-configuration.

## Stereotypes: @Component, @Service, @Repository, @Controller

Все — маркеры «этот класс — bean, регистрируй». Функционально почти одинаковы. Semantic differences в некоторых.

@Component — базовый. Generic bean marker.

@Service — семантика business logic. Non-technical marker для service layer classes.

@Repository — семантика data access. Плюс важное — Spring оборачивает исключения в DataAccessException через PersistenceExceptionTranslationPostProcessor. Database vendor exceptions translated к consistent Spring hierarchy.

@Controller — MVC-контроллер. Возвращает view names (не response body).

@RestController = @Controller plus @ResponseBody. JSON everywhere convenience. Standard для REST APIs.

Реализация. ClassPathBeanDefinitionScanner сканирует пакет. Ищет классы с этими аннотациями (через meta-annotation @Component). Регистрирует как BeanDefinition. Дальше — обычная фабрика бинов создаёт instances.

Semantic naming важен для readability. Reader знает role класса по annotation. Non-functional difference но important для code understanding.

## @Configuration и CGLib proxy

Класс с @Bean методами:
```java
@Configuration
class AppConfig {
    @Bean DataSource dataSource() { ... }
    @Bean JdbcTemplate jdbc(DataSource ds) { return new JdbcTemplate(ds); }
}
```

Реализация. ConfigurationClassPostProcessor (BeanFactoryPostProcessor). Находит все @Configuration классы. Читает их @Bean методы. Регистрирует каждый метод как BeanDefinition (factory method).

CGLib proxy. @Configuration класс оборачивается в CGLib proxy. Зачем?
```java
@Configuration
class Cfg {
    @Bean DataSource ds() { return new HikariDataSource(...); }
    @Bean JdbcTemplate jdbc() {
        return new JdbcTemplate(ds());   // если бы не прокси, было бы второй DataSource
    }
}
```

Прокси перехватывает ds() внутри jdbc() — возвращает тот же singleton, не создаёт новый. Ensures @Bean methods return same instance regardless of call location.

Без proxy — ds() вызывается directly, creates new HikariDataSource каждый раз. Bean registration semantics broken. Multiple DataSources в container вместо singleton.

Отключить (быстрее старт, но без гарантии singleton внутри):
```java
@Configuration(proxyBeanMethods = false)
```

Используется в auto-configuration classes где @Bean methods не call each other. Optimization.

## @Bean

Метод как factory для bean:
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

Использование. Сторонние классы (не можешь навесить @Component). Условная логика создания. Auto-configuration.

Имя bean = имя метода (или явно через @Bean(name = "...")). Multiple names via name = {"n1", "n2"}.

initMethod / destroyMethod для lifecycle callbacks. Called после construction / перед destruction. Alternative to InitializingBean/DisposableBean interfaces.

## @Autowired

Внедрение зависимости:
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

Правило constructor injection. Immutability. Explicit dependencies. Easier testing (no reflection required).

С Spring 4.3+ на единственном конструкторе @Autowired не нужен. Implicit annotation.

Реализация. AutowiredAnnotationBeanPostProcessor (BeanPostProcessor). При создании bean — сканирует поля/сеттеры/конструкторы с @Autowired. Для каждой зависимости — резолвит через BeanFactory.getBean(type). Injectит через reflection (или constructor).

@Autowired(required = false) не падать если нет bean:
```java
@Autowired(required = false) SomeOptional optional;
```

Или через Optional<T>:
```java
@Autowired Optional<SomeOptional> optional;
```

Cleaner API для optional dependencies. Explicit Optional wrapping preferred over required=false.

## @Qualifier

Уточняет какой bean, когда несколько кандидатов:
```java
@Bean("fast") FnoService fast() { ... }
@Bean("slow") FnoService slow() { ... }

@Autowired @Qualifier("fast") FnoService svc;
```

Handles multiple beans same type. Explicit choice.

## @Primary

Метка «этот по умолчанию»:
```java
@Bean @Primary FnoService fast() { ... }
@Bean FnoService slow() { ... }

@Autowired FnoService svc;   // fast (потому что @Primary)
```

Default choice when multiple beans same type. @Qualifier overrides @Primary когда explicitly specified.

## @Lazy

Bean создаётся не при старте, а при первом использовании:
```java
@Component @Lazy
class ExpensiveBean { ... }
```

Или на injection:
```java
@Autowired @Lazy ExpensiveBean bean;   // прокси, реальный при первом вызове
```

Использование. Тяжёлые бины (не создавать если не нужны). Разрешение циклических зависимостей.

Circular dependency resolution. @Lazy на injection creates proxy — actual bean fetched on first method call. Both sides can initialize через proxy references.

## @Scope

Область жизни bean:
```java
@Component @Scope("prototype") class FnoBuilder { }
@Component @Scope("request") class RequestContext { }   // web-only
```

Scopes. singleton (default) — one instance per container. prototype — new instance каждый раз. request — one instance per HTTP request. session — one per HTTP session. application — one per servlet context.

Реализация. BeanFactory.getBean() каждый раз для prototype (new instance created). Кэш для singleton (same instance returned). Web scopes tie к servlet lifecycle.

## @EnableAutoConfiguration

Магия Spring Boot. Автоматически конфигурирует бины по classpath.

Реализация. При старте. Читаются файлы META-INF/spring.factories (Boot 2) или META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports (Boot 3) из всех jar'ов. Получается список auto-configuration классов. Каждый — @Configuration с @Conditional* условиями. Если условия выполнены — бины регистрируются.

Пример flow. Spring Boot Data JPA starter в classpath — includes JpaAutoConfiguration. Condition — @ConditionalOnClass(EntityManagerFactory.class). Если EntityManagerFactory на classpath — configuration applied — beans registered.

Отключение специфических autoconfigurations:
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

Sometimes needed когда default auto-configuration incompatible с specific requirements.

## @Conditional* family

Регистрация bean по условию:
```java
@Bean
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean
@ConditionalOnProperty(name = "spring.datasource.url")
DataSource dataSource() { ... }
```

Виды conditions.

@ConditionalOnClass — есть класс в classpath. Enable feature только если dependency available.

@ConditionalOnMissingClass — нет класса. Opposite.

@ConditionalOnBean — есть bean этого типа. Chain configurations.

@ConditionalOnMissingBean — нет bean. User did not provide own — use default. Enables user overrides.

@ConditionalOnProperty(name, havingValue, matchIfMissing) — по property value. Feature flags.

@ConditionalOnWebApplication — web context.

@ConditionalOnJava — Java version. Version-specific configurations.

@Conditional(MyCondition.class) — custom logic. Extend for specific requirements.

Основа Spring Boot auto-configuration. Combining conditions enables sophisticated intelligent defaults.

## @ComponentScan

Сканирование classpath на bean'ы:
```java
@ComponentScan(basePackages = {"kz.gov.kgd.isna.knp", "kz.gov.kgd.isna.commons"})
class Config { }
```

По умолчанию (внутри @SpringBootApplication) — сканирует пакет класса с аннотацией plus подпакеты.

Реализация. ClassPathScanningCandidateComponentProvider. Reads class files. Checks for stereotype annotations. Registers matching classes as BeanDefinitions.

Package structure critical. Application class в корне scanned package. Sub-packages для different concerns. Everything within reach.

## @Value

Injection value из configuration:
```java
@Value("${server.port}") int port;
@Value("${app.name:defaultName}") String appName;
@Value("#{systemProperties['user.name']}") String user;   // SpEL
@Value("${servers}") List<String> servers;                  // comma-separated
```

Реализация. Environment.getProperty(...) plus placeholder resolution. SpEL expressions evaluated at injection.

Detailed usage в файле 57. Here — annotation semantic reminder.

## @ConfigurationProperties

Группа properties как typed bean:
```java
@ConfigurationProperties(prefix = "knp")
@Component
public class KnpProperties {
    private int maxBatchSize = 100;
    private String outerSystemUrl;
    // getters/setters
}
```

Уже разбирали в файле 57. Реализация — ConfigurationPropertiesBinder через reflection plus type conversion.

Preferred over @Value для multiple related properties. Type safety, validation, IDE support.

## @Profile

Bean/config активен только при активном profile:
```java
@Configuration @Profile("prod") class ProdConfig { }
@Component @Profile({"dev", "test"}) class DevOnlyBean { }
@Component @Profile("!prod") class NonProdBean { }
```

Реализация. ProfileCondition (частный случай @Conditional). Checks active profiles from Environment.

Enables completely different configurations per environment. Discussed in file 57.

## @EnableXxx pattern

Spring/Boot использует много enabler'ов.

@EnableTransactionManagement — включает @Transactional через AOP. @EnableAsync — включает @Async. @EnableScheduling — включает @Scheduled. @EnableCaching — включает @Cacheable. @EnableAspectJAutoProxy — включает AOP infrastructure. @EnableConfigurationProperties(Xxx.class) — регистрирует @ConfigurationProperties bean. @EnableWebSecurity — Security config. @EnableJpaRepositories — Spring Data JPA. @EnableFeignClients — Feign.

Механика. @Import загружает specific configuration classes. Bootstrap beans plus infrastructure для feature.

В Spring Boot большинство @Enable автоматически (не нужны явно) — auto-configuration включает when appropriate dependencies present. Manual usage для specific control или для non-Boot Spring applications.

## @Async

Метод выполняется в другом потоке:
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

Требует @EnableAsync.

Реализация. AOP-прокси перехватывает вызов. Сабмитит в TaskExecutor. Возвращает Future (или void).

Кавет. Self-invocation — та же проблема что @Transactional. Method call через this bypasses proxy. Не async.

Void методы — не узнаешь про exception. Использовать CompletableFuture для error handling.

По default пул — SimpleAsyncTaskExecutor (создаёт thread на вызов). Плохо для нагрузки. Настрой свой:
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

Bounded pool с queue. Prevents unlimited thread creation. Production-ready configuration.

## @Scheduled

Cron / periodic tasks:
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

Требует @EnableScheduling.

Реализация. ScheduledAnnotationBeanPostProcessor находит @Scheduled методы. Регистрирует в TaskScheduler. TaskScheduler дергает according к schedule.

Кавет multi-instance. Если приложение в K8s с 3 репликами — все 3 запустят cron параллельно. Duplicate execution problem.

Fix ShedLock — distributed lock:
```java
@Scheduled(cron = "...")
@SchedulerLock(name = "myTask", lockAtMostFor = "PT30S")
public void task() { ... }
```

Only one instance executes at a time. Lock stored в shared database.

Реальный ИСНА-кейс knp-fno21-shedlock-stale-image-dup-regnum — без ShedLock дубли. Fix — deploy с ShedLock plus proper lock configuration.

## @Transactional

Обёртка транзакций через AOP-proxy. Разбирали детально в файлах 32-35:
```java
@Transactional
public void save(Order o) { ... }

@Transactional(readOnly = true, timeout = 10, propagation = REQUIRES_NEW,
    rollbackFor = Exception.class)
public Order read(Long id) { ... }
```

Полный deep dive в earlier files. Here — brief reminder что это AOP-proxy based mechanism through TransactionInterceptor.

## @Cacheable

Кэширование результата метода:
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

Требует @EnableCaching plus cache-manager (Hazelcast, Caffeine, Redis).

Реализация через AOP-proxy. First call — actual method invoked, result cached. Subsequent calls с same key — cached value returned без method execution.

@CacheEvict removes entries. @CachePut updates cache plus returns method result. @Caching для combining multiple operations.

Self-invocation — та же проблема. this-based calls bypass proxy.

Cache keys default from method arguments. Custom keys через SpEL (key = "#id").

## @PostConstruct / @PreDestroy

Lifecycle-хуки bean:
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

С Spring 6 / Java 9+ — из пакета jakarta.annotation.* (не javax.annotation.*). Java module system change.

Реализация. CommonAnnotationBeanPostProcessor. Invoked after dependency injection (PostConstruct) plus before bean destruction (PreDestroy).

Кавет. @Transactional НЕ работает в @PostConstruct. Bean ещё не проксирован. Fix через ApplicationRunner или @EventListener(ContextRefreshedEvent.class) для post-startup logic.

## @EventListener

Подписка на events:
```java
@Component
class OrderListener {
    @EventListener
    public void handle(OrderCreated event) { ... }

    @EventListener(condition = "#event.status == 'NEW'")
    public void handleNew(OrderCreated event) { ... }
}
```

Синхронно (в том же потоке что publishEvent). Event handled inline с publishing thread.

С @Async — async execution. Non-blocking publishing.

@TransactionalEventListener для transaction-aware events:
```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void afterCommit(OrderCreated event) { ... }
```

Только после успешного commit tx. Prevents events firing на rolled back transactions. Разбирали в файле 34.

## @RestController vs @Controller

@Controller — MVC, возвращает view name (Thymeleaf/JSP):
```java
@Controller
class OrderController {
    @GetMapping("/orders")
    public String list(Model model) {
        model.addAttribute("orders", ...);
        return "orders";   // orders.html
    }
}
```

@RestController = @Controller plus @ResponseBody на всех методах:
```java
@RestController
class OrderRestController {
    @GetMapping("/api/orders")
    public List<Order> list() {
        return svc.findAll();   // JSON
    }
}
```

Обычно микросервисы — @RestController. Traditional web apps — @Controller.

Difference. @Controller returns view names (rendered by template engine). @RestController returns objects (serialized via Jackson к JSON).

## @RequestMapping и variants

Full request mapping:
```java
@RequestMapping(value = "/api/orders", method = RequestMethod.GET)
public List<Order> list() { }
```

Shortcuts более common:
```java
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

## Параметры контроллера

Rich API для extracting request data:
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

@PathVariable — URL path segments. @RequestParam — query parameters. @RequestBody — HTTP body (deserialized via Jackson). @RequestHeader — HTTP headers. @AuthenticationPrincipal — current authenticated user.

## @Valid / @Validated

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

При invalid — MethodArgumentNotValidException — 400 response.

Standard validation annotations. @NotNull, @NotBlank, @Size, @Min, @Max, @Email, @Pattern. Composable для complex validations.

Custom validators possible через extending framework.

## @ExceptionHandler / @ControllerAdvice

Централизованная обработка exceptions:
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

@ControllerAdvice — global (все controllers) или scoped:
```java
@ControllerAdvice(basePackages = "kz.gov.kgd.isna.knp.api")
```

Applies to controllers в specified package. Centralized error handling instead of repeated try/catch в каждом controller method.

Consistent error responses. Business exceptions mapped к appropriate HTTP status codes. Client sees uniform error format.

## Другие полезные аннотации

@Order приоритет beans при инжекции коллекций / автопрокси:
```java
@Component @Order(1) class Handler1 { }
@Component @Order(2) class Handler2 { }

@Autowired List<Handler> handlers;   // [Handler1, Handler2]
```

Deterministic ordering. Useful для chain-of-responsibility patterns.

@Import импорт additional Configuration classes:
```java
@Configuration
@Import({DbConfig.class, SecurityConfig.class})
class AppConfig { }
```

Composition configurations. Основа @Enable* аннотаций. @EnableXxx imports specific configuration classes internally.

@PropertySource дополнительный source properties:
```java
@Configuration
@PropertySource("classpath:custom.properties")
class Config { }
```

Устарело в пользу yml plus profile-specific. Rarely used в modern applications.

@Retryable (Spring Retry) retry метода при exception:
```java
@Retryable(retryFor = IOException.class, maxAttempts = 3, backoff = @Backoff(delay = 1000))
public String call() { ... }
```

Discussed в файле 52 detail. AOP-proxy based similar к @Transactional.

## Итоги

@SpringBootApplication composition из three annotations. @SpringBootConfiguration plus @EnableAutoConfiguration plus @ComponentScan. Central entry point.

Stereotypes (@Component/@Service/@Repository/@Controller) семантика plus @Repository exception translation. Regular beans registered via ClassPathBeanDefinitionScanner.

@Configuration proxied через CGLib для singleton гарантии в @Bean method calls. proxyBeanMethods=false для optimization в auto-configuration.

@Bean methods as factories. Third-party classes без @Component. Conditional bean creation.

@Autowired через AutowiredAnnotationBeanPostProcessor plus reflection. Constructor injection preferred. Optional<T> для optional dependencies.

@Qualifier resolves ambiguity. @Primary sets default. @Lazy defers creation.

@Scope controls lifecycle. Singleton default. Prototype, request, session, application альтернативы.

@EnableAutoConfiguration reads spring.factories/AutoConfiguration.imports. Applies conditional configurations based на classpath.

@Conditional* family enables intelligent defaults. @ConditionalOnClass, @ConditionalOnBean, @ConditionalOnMissingBean, @ConditionalOnProperty стандартные.

@ComponentScan discovers beans. Package structure aligned с scan requirements.

@Value simple property injection. @ConfigurationProperties typed groups preferred для multiple related properties.

@Profile conditional based на active profiles. @Enable* pattern для feature toggles.

@Async, @Scheduled, @Transactional, @Cacheable через AOP-proxy. Все страдают self-invocation limitation.

@PostConstruct/@PreDestroy lifecycle callbacks. jakarta.annotation.* с Spring 6/Java 9+. @Transactional не работает в @PostConstruct.

@EventListener для synchronous events. @Async plus @EventListener для async. @TransactionalEventListener для transaction-aware.

@RestController = @Controller plus @ResponseBody. REST APIs standard.

@RequestMapping и HTTP method shortcuts для routing.

Parameter annotations — @PathVariable, @RequestParam, @RequestBody, @RequestHeader, @AuthenticationPrincipal.

@Valid triggers Bean Validation. @ControllerAdvice centralized exception handling.

Utility annotations — @Order, @Import, @PropertySource, @Retryable.

Understanding annotation internals enables predicting behavior plus diagnosing issues. Not magic — well-defined framework mechanisms.

Дальше — Spring MVC controllers internals с deep dive в DispatcherServlet, HandlerMapping, HandlerAdapter, argument resolution, response processing.
