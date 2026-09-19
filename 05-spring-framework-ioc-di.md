# 05. Spring Framework: IoC, DI, beans, контекст и AOP-прокси

## Почему Spring везде

Если ты работаешь Java-разработчиком в 2020-х, ты работаешь со Spring. Практически весь enterprise Java-мир построен на нём. Spring MVC для REST-контроллеров, Spring Data JPA для БД, Spring Security для аутентификации, Spring Cloud для микросервисной инфраструктуры, Spring Boot для быстрого запуска — экосистема покрывает всё. В КНП каждый микросервис — Spring Boot приложение.

Знание Spring на собеседовании senior-инженера обязательно. Не потому что "модно", а потому что вся архитектура enterprise Java-приложений сегодня построена вокруг его концепций. IoC контейнер, dependency injection, application context, AOP прокси, декларативное управление транзакциями — фундамент, без понимания которого невозможно писать хороший код или диагностировать проблемы.

Многие разработчики учатся Spring на уровне "куда какую аннотацию поставить". `@Service`, `@Autowired`, `@Transactional` — работает, ладно. Но когда возникают нестандартные ситуации — приложение не стартует с BeanCreationException, `@Transactional` не работает несмотря на аннотацию, циклическая зависимость — знания на этом уровне не хватает. Нужно понимание что происходит под капотом.

В этом файле разберём Spring детально. Что такое IoC и почему он полезен. Как реализуется DI и три способа инъекции. Что такое bean и lifecycle. Как работает ApplicationContext и component scan. Scopes и как их использовать. AOP-прокси, `@Transactional` изнутри, self-invocation gotcha. `@Configuration` proxy и singleton гарантии. Profiles, `@Value`, `@ConfigurationProperties`. Всё это на уровне достаточном чтобы понимать любую Spring-специфичную проблему в production.

## Проблема, которую решает Spring

Начнём с проблемы. Пусть у нас класс, которому нужен репозиторий, а тому — база данных, а той — конфигурация. Наивная реализация:

```java
class KnpService {
    private final FnoRepository repo;

    public KnpService() {
        Properties config = readConfig("jdbc.properties");
        DataSource ds = new HikariDataSource(configFromProperties(config));
        Database db = new Database(ds);
        this.repo = new FnoRepository(db);
    }

    public void process(Fno fno) { 
        repo.save(fno); 
    }
}
```

Проблем много.

**Жёсткая связь**. KnpService знает про Hikari, знает про файл конфигурации, знает как это всё собрать. Если завтра решат сменить HikariDataSource на TomcatJdbcDataSource — нужно менять код в KnpService. Плюс во всех остальных сервисах, которые тоже сами создают DataSource. Один архитектурный change требует правок в десятках мест.

**Дублирование**. Каждый сервис самостоятельно создаёт DataSource. У нас 50 микросервисов, 50 разных мест где HikariDataSource инстанцируется. Никакой централизованной точки конфигурации.

**Тестируемость**. Юнит-тест KnpService конструктор попытается открыть реальную базу данных, читая конфиг. Тест перестаёт быть юнит-тестом. Для теста нужно как-то подсунуть мок репозитория — но конструктор не позволяет.

**Порядок создания**. Кто создаёт что и когда? В простом примере понятно, в реальной системе с сотнями компонентов — граф зависимостей становится непонятным. Легко нарушить порядок и получить NullPointerException.

Все эти проблемы решает Spring. Правильная версия того же кода:

```java
@Service
class KnpService {
    private final FnoRepository repo;

    public KnpService(FnoRepository repo) {
        this.repo = repo;
    }

    public void process(Fno fno) { 
        repo.save(fno); 
    }
}
```

Ноль `new`. Никакого знания про Hikari или конфигурацию. Компоненту нужен `FnoRepository` — он объявляет это в конструкторе, Spring подставит правильный экземпляр. Кто и как создаёт этот `FnoRepository` — забота Spring, не наша.

**Слабая связь** — Spring service не знает про реализацию репозитория, детали DataSource. Можно поменять внутреннюю реализацию без правок сервиса.

**Тестируемость** — в юнит-тесте создаёшь `new KnpService(mockRepo)`, никакой Spring не нужен, никакие БД не открываются.

**Единая точка конфигурации** — где-то (обычно в `@Configuration` классах или через auto-configuration) Spring один раз собирает граф зависимостей. Все сервисы получают правильные экземпляры автоматически.

**Явные зависимости** — параметры конструктора видны, легко понять что от чего зависит.

## IoC: Inversion of Control

**Inversion of Control (IoC)** — фундаментальный принцип, стоящий за Spring. Название описывает суть: инверсия контроля.

Раньше (без IoC) ты сам создавал объекты, знал как их конструировать, помнил порядок инициализации. Твой код контролировал жизненный цикл. С IoC — **контейнер** создаёт объекты, знает граф зависимостей, инициализирует всё в правильном порядке. Ты только описываешь: какие мне нужны компоненты и что от чего зависит.

Аналогия. В магазине без IoC ты сам ищешь товары по полкам, знаешь где что лежит, собираешь корзину. С IoC — приходишь с списком нужного (аннотации на классах и конструкторы), кто-то другой (контейнер) собирает всё что тебе нужно, доставляет.

IoC — общее понятие, применимое к многим ситуациям. Callback-style программирование — тоже IoC (не ты вызываешь код, а фреймворк вызывает твой callback). Event-driven архитектура — IoC (события управляют потоком, не последовательный код).

В Spring реализация IoC — **Dependency Injection (DI) контейнер**. Компоненты объявляют зависимости, контейнер их удовлетворяет.

## Три способа Dependency Injection

Есть три способа "впрыснуть" зависимость в объект. У каждого свои плюсы и минусы, и важно понимать когда что использовать.

**Constructor Injection** — предпочтительный подход.

```java
@Service
class KnpService {
    private final FnoRepository repo;
    private final PublishService publisher;

    public KnpService(FnoRepository repo, PublishService publisher) {
        this.repo = repo;
        this.publisher = publisher;
    }
}
```

Зависимости передаются через параметры конструктора. Плюсы:

- Поля могут быть `final` — immutable, thread-safe, невозможно случайно изменить.
- Все зависимости в сигнатуре конструктора — сразу видно от чего зависит класс.
- Компонент не может существовать без своих зависимостей — конструктор их требует.
- Легко тестировать — просто `new KnpService(mock1, mock2)`, без Spring.
- Циклические зависимости обнаруживаются при старте, а не в runtime.

С Spring 4.3+ **не нужна аннотация `@Autowired`** на конструкторе если он единственный — Spring сам определит.

Правило: для новых классов всегда constructor injection.

**Setter Injection** — устаревающий подход:

```java
@Service
class KnpService {
    private FnoRepository repo;

    @Autowired
    public void setRepo(FnoRepository repo) {
        this.repo = repo;
    }
}
```

Зависимости через отдельные setter методы. Использовать только:

- Для опциональных зависимостей (`@Autowired(required = false)`) — если компонент может работать без некоторой зависимости.
- Для разрешения циклических зависимостей (последнее средство).

Минусы: поле не `final`, объект может существовать без зависимости, скрывает зависимости.

**Field Injection** — плохой, но часто встречается:

```java
@Service
class KnpService {
    @Autowired
    private FnoRepository repo;
}
```

Минимум кода — прямо в поле. Но много минусов:

- Поле не `final`.
- Не работает без Spring (в тестах нужен `@InjectMocks` или reflection).
- Скрывает зависимости — они не в конструкторе, легко пропустить.
- Циклические зависимости обнаруживаются в runtime.
- Изменение зависимости требует reflection, что противоречит encapsulation.

Field injection в новом коде — anti-pattern. В legacy встречается, но при рефакторинге лучше конвертировать в constructor injection.

## Что такое Bean

Компонент, которым управляет Spring контейнер, называется **bean** (компонент, боб — устоявшийся термин из Java EE).

Не любой Java-объект — bean. Ключевые свойства bean:

- Создан контейнером (Spring вызывает конструктор или `@Bean`-метод).
- Живёт в контейнере (обычно всё время работы приложения).
- Может быть инжектирован в другие компоненты через DI.
- Проходит через lifecycle hooks (init, destroy).
- Управляется контейнером до shutdown.

Обычный DTO вроде `new UserDto()` — не bean. Ты его создал руками, Spring про него не знает.

## Как объявить bean

Есть два основных способа объявить класс как bean.

**Стереотипные аннотации + component scan**. Ставишь на класс одну из аннотаций-стереотипов:

```java
@Service class KnpService { ... }
@Repository class FnoRepository { ... }
@Controller class FnoController { ... }
@Component class SomeUtil { ... }
```

Где-то (обычно на `@SpringBootApplication`, который включает `@ComponentScan`) Spring сканирует указанный пакет и вложенные, находит все аннотированные классы, регистрирует их как beans.

Component scan рекурсивно проходит по всем classpath resources в указанных пакетах, ищет `.class` файлы, читает их метаданные, находит стереотипы, регистрирует beans.

**`@Configuration` + `@Bean` методы**. Явное определение beans через конфигурационные классы:

```java
@Configuration
class AppConfig {

    @Bean
    public DataSource dataSource() {
        HikariConfig cfg = new HikariConfig();
        cfg.setJdbcUrl("jdbc:postgresql://...");
        return new HikariDataSource(cfg);
    }

    @Bean
    public FnoRepository fnoRepository(DataSource ds) {
        return new FnoRepository(ds);
    }
}
```

Каждый `@Bean` метод — фабрика для bean того типа, что возвращает. Параметры метода — зависимости, Spring их подставляет из других beans.

Использование `@Configuration`:

- Класс сторонний, не можешь навесить `@Service` (`HikariDataSource` — не твой класс).
- Нужна условная логика создания (`@ConditionalOnProperty`).
- Комплексная инициализация с настройками.
- Auto-configuration в Spring Boot полностью построена на этом подходе.

## Стереотипные аннотации и их семантика

Разные стереотипы функционально почти одинаковы (все делают класс bean'ом), но с семантическими различиями.

**`@Component`** — базовый стереотип. Просто "я bean". Используется когда никакая другая семантика не подходит.

**`@Service`** — семантика бизнес-логики, application services. Функционально идентичен `@Component`, но говорит "это сервисный слой". Помогает читаемости кода — сразу видно роль класса.

**`@Repository`** — data access layer. Здесь есть функциональное отличие: Spring подключает `PersistenceExceptionTranslationPostProcessor`, оборачивающий низкоуровневые исключения (SQLException, HibernateException) в универсальные Spring `DataAccessException`. Это позволяет писать catch-блоки независимо от нижней технологии (JDBC, JPA, MongoDB).

**`@Controller`** — MVC контроллер. Методы возвращают view names для рендеринга template engine. В микросервисах редко.

**`@RestController`** — `@Controller` + `@ResponseBody` на всех методах. Автоматически сериализует возвращаемые объекты в JSON/XML. Стандарт для REST API.

**`@Configuration`** — класс с фабричными методами. Особая обработка: Spring оборачивает такой класс в CGLIB proxy для гарантии singleton (см. ниже).

В реальном коде видишь mix всех. Тонкости различий редко имеют значение, кроме `@Repository` (exception translation) и `@Configuration` (proxy behavior).

## ApplicationContext и BeanFactory

Контейнер, содержащий все beans приложения, — **BeanFactory**. Базовый интерфейс, минимальный API для управления beans.

Практически всегда используется расширение **ApplicationContext** — добавляет много полезного поверх BeanFactory:

- Event publishing (`ApplicationEventPublisher`) — publish-subscribe между компонентами.
- Resource loading — доступ к файлам, classpath ресурсам через единый API.
- Message resolution — интернационализация (i18n).
- Environment — доступ к properties, profiles, system variables.
- Lifecycle management — hooks для init и shutdown.

Разные реализации ApplicationContext под разные сценарии.

**`AnnotationConfigApplicationContext`** — для конфигурации через аннотации (`@Configuration`).

**`ClassPathXmlApplicationContext`** — legacy XML конфигурация.

**`WebApplicationContext`** — для web-приложений.

**`ServletWebServerApplicationContext`** — Spring Boot с embedded server (Tomcat, Jetty, Undertow).

**`ReactiveWebServerApplicationContext`** — Spring Boot с WebFlux и Netty.

Для типичного Spring Boot приложения ты не создаёшь контекст руками — `SpringApplication.run()` делает это автоматически.

Получить bean программно можно через инъекцию ApplicationContext:

```java
@Autowired
private ApplicationContext ctx;

FnoService svc = ctx.getBean(FnoService.class);
FnoService named = ctx.getBean("specificName", FnoService.class);
```

Обычно не нужно — Spring сам инжектирует зависимости. Программный getBean используется для plugin-style архитектур, динамического lookup, специальных случаев.

## Bean lifecycle

Каждый bean проходит через определённые фазы от создания до уничтожения.

**Фаза 1: Instantiation**. Spring вызывает конструктор или `@Bean`-метод, создавая экземпляр объекта.

**Фаза 2: Populate properties**. Если есть `@Autowired` поля или setter'ы, Spring внедряет зависимости. Для constructor injection — эта фаза уже сделана в конструкторе.

**Фаза 3: Aware interfaces**. Если bean реализует специальные интерфейсы (`BeanNameAware`, `BeanFactoryAware`, `ApplicationContextAware`), Spring вызывает их setter методы, подсовывая нужные объекты.

**Фаза 4: `BeanPostProcessor.postProcessBeforeInitialization`**. Ключевая расширяемая точка. Все зарегистрированные BeanPostProcessor'ы обрабатывают bean. Здесь применяется @Autowired через AutowiredAnnotationBeanPostProcessor, обрабатываются другие аннотации.

**Фаза 5: `@PostConstruct` метод**. Если у bean есть метод помеченный `@PostConstruct`, он вызывается. Bean уже полностью настроен (зависимости внедрены), готов к финальной инициализации.

**Фаза 6: `InitializingBean.afterPropertiesSet()`**. Если bean реализует этот интерфейс, вызывается метод. Альтернатива @PostConstruct.

**Фаза 7: Custom init-method**. Если объявлен `@Bean(initMethod = "myInit")`, вызывается указанный метод.

**Фаза 8: `BeanPostProcessor.postProcessAfterInitialization`**. Здесь Spring оборачивает bean в proxy если нужно (для `@Transactional`, `@Async`, `@Cacheable`). Возвращённый proxy идёт в контекст вместо оригинала.

После этого — **bean готов, живёт в контейнере**, обслуживает запросы.

**При shutdown**:

**Фаза 9: `@PreDestroy`** — метод вызывается для cleanup.

**Фаза 10: `DisposableBean.destroy()`** — если реализован интерфейс.

**Фаза 11: Custom destroy-method**.

Практические применения. `@PostConstruct` — прогрев кэшей, валидация конфигурации, регистрация в внешних системах. `@PreDestroy` — закрыть connection pool, deregister из discovery, отправить последнюю партию сообщений.

Пример:

```java
@Service
class CacheWarmer {
    private final FnoRepository repo;
    private Cache cache;

    public CacheWarmer(FnoRepository repo) {
        this.repo = repo;
    }

    @PostConstruct
    void init() {
        cache = repo.loadAll().stream()
            .collect(Collectors.toMap(Fno::getId, f -> f));
        log.info("Cache warmed with {} entries", cache.size());
    }

    @PreDestroy
    void shutdown() {
        cache.clear();
    }
}
```

## Scopes

**Scope** определяет сколько живут экземпляры bean.

**Singleton** (по умолчанию) — один экземпляр на весь ApplicationContext. Все компоненты, зависящие от этого bean, получают тот же экземпляр. Правильно для сервисов, репозиториев, конфигураций — не нужно много копий.

**Prototype** — новый экземпляр на каждый запрос из контейнера. Полезно для stateful объектов, которые не должны шариться.

**Request** — один экземпляр на HTTP request. Web-only. Живёт от начала обработки запроса до ответа.

**Session** — один на HTTP session.

**Application** — один на ServletContext.

**Websocket** — один на WebSocket session.

Задаётся аннотацией:

```java
@Service
@Scope("prototype")
class FnoBuilder { ... }
```

Или через `@Scope(scopeName = ConfigurableBeanFactory.SCOPE_PROTOTYPE)`.

**Классический gotcha**: инжектируя prototype в singleton, ты получаешь один и тот же экземпляр — первый резолв. Singleton создан один раз, при создании DI вставила prototype, дальше singleton держит эту ссылку.

Если действительно нужно новый экземпляр каждый раз при использовании — использовать `ObjectProvider`:

```java
@Service
class Consumer {
    private final ObjectProvider<PrototypeService> providerss;

    public Consumer(ObjectProvider<PrototypeService> providers) {
        this.providers = providers;
    }

    public void doWork() {
        PrototypeService instance = providers.getObject();
        // используем instance
    }
}
```

Или method injection через `@Lookup`.

## @Autowired, @Qualifier, @Primary

`@Autowired` — резолвит зависимость **по типу**. Если найден один bean этого типа — внедряется. Если несколько — конфликт, ошибка `NoUniqueBeanDefinitionException`.

Пример конфликта:

```java
@Bean("fastFno")
FnoService fastImpl() { ... }

@Bean("slowFno")
FnoService slowImpl() { ... }

@Service
class SomeService {
    @Autowired
    FnoService fno;  // ошибка — два bean типа FnoService
}
```

**`@Qualifier`** — уточняет какой именно bean нужен:

```java
@Autowired
@Qualifier("fastFno")
FnoService fno;
```

**`@Primary`** — помечает bean как "по умолчанию":

```java
@Bean
@Primary
FnoService fastImpl() { ... }

@Bean
FnoService slowImpl() { ... }

// В другом месте:
@Autowired
FnoService fno;  // получает fastImpl из-за @Primary
```

**Инъекция коллекций** — Spring может инжектировать все beans одного типа:

```java
@Autowired
List<FnoValidator> validators;  // все bean'ы типа FnoValidator

@Autowired
Map<String, FnoValidator> validators;  // ключ = имя bean'а
```

Полезно для strategy-паттерна: разные валидаторы регистрируются как beans, сервис получает всех, обходит списком.

## Циклические зависимости

Классическая проблема: A зависит от B, B зависит от A.

```java
@Service class A {
    public A(B b) { ... }
}

@Service class B {
    public B(A a) { ... }
}
```

Spring не может создать ни один экземпляр: чтобы создать A нужен B, чтобы создать B нужен A. Deadlock.

При старте Spring обнаруживает: `BeanCurrentlyInCreationException`.

Способы решения (от лучшего к худшему):

1. **Refactor**. Цикл обычно указывает на плохой дизайн. Часто общая функциональность может быть извлечена в третий класс C, от которого зависят оба A и B.

2. **Setter injection**. Spring может создать оба объекта, потом через setter'ы взаимно связать. Работает для setter, не для constructor injection.

3. **`@Lazy`**. Один из bean'ов помечается `@Lazy`. Spring создаёт proxy без реального создания, при первом обращении инициализирует.

С Spring Boot 2.6+ **циклические зависимости запрещены по умолчанию** (`spring.main.allow-circular-references=false`). Правильный подход — рефакторить.

## AOP-прокси

Одна из самых мощных фич Spring — **AOP (Aspect-Oriented Programming)** через прокси. Позволяет добавлять cross-cutting concerns (транзакции, логирование, security, caching) декларативно, без правки бизнес-кода.

Как это работает. Когда ты пишешь:

```java
@Service
class KnpService {
    @Transactional
    public void doWork() { ... }
}
```

Spring не отдаёт тебе экземпляр `KnpService` напрямую. Он создаёт **proxy** — динамически сгенерированный подкласс:

```java
class KnpService$$EnhancerBySpringCGLIB extends KnpService {
    public void doWork() {
        openTransaction();
        try {
            super.doWork();  // вызов оригинального метода
            commit();
        } catch (Exception e) {
            rollback();
            throw e;
        }
    }
}
```

Везде где инжектируется KnpService — на самом деле инжектируется этот proxy. Все вызовы методов идут через proxy, вокруг них накладывается транзакционная логика.

## Два вида прокси

**JDK Dynamic Proxy** — стандартный механизм Java, работает только с интерфейсами. Прокси реализует те же интерфейсы, что и target. Использует reflection для делегирования вызовов.

**CGLIB Proxy** — генерирует **подкласс** целевого класса через byte code manipulation. Работает для классов без интерфейсов. Ограничения: не может проксировать `final` классы (нельзя extend), не может проксировать `final` методы (нельзя override).

Spring Boot по умолчанию использует CGLIB (`spring.aop.proxy-target-class=true`). До Spring Boot 2 по умолчанию было наоборот — JDK Dynamic Proxy когда есть интерфейсы. Переключение на CGLIB упростило многие ситуации.

Разница на практике редко видна пользователю. Оба работают, оба поддерживают AOP аспекты. CGLIB немного быстрее (нет reflection overhead), но не может проксировать final.

## Self-invocation: критический gotcha

Самая частая проблема с AOP-прокси — self-invocation. Прокси перехватывает вызовы **снаружи**. Если ты вызываешь метод изнутри того же класса через `this` — вызов идёт напрямую, минуя proxy.

```java
@Service
class NotificationService {

    @Transactional
    public void processAll() {
        for (var n : list) {
            this.processOne(n);  // вызов через this — НЕ через прокси!
        }
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void processOne(Notification n) {
        // хотели новую транзакцию на каждую итерацию
        // но получаем одну общую от processAll()
    }
}
```

Аннотация `@Transactional(REQUIRES_NEW)` на `processOne` НЕ срабатывает. Proxy оборачивает `processAll`, но внутренний вызов `this.processOne` идёт напрямую на объект, не через proxy. Никакая транзакционная логика вокруг processOne не запускается.

Это одна из самых частых ошибок Spring разработчиков. Real кейс из КНП: `knp-fo-sync-notification-bugs` — `@Transactional(REQUIRES_NEW)` был мёртв из-за self-invocation. Следствие: одна большая транзакция вместо N маленьких, при обработке большого количества сообщений возникали LazyInitializationException при работе с Hibernate.

**Как чинить**:

1. **Вытащить метод в другой bean**. Простое и правильное решение. `NotificationProcessor` в отдельном классе, содержит `processOne`. `NotificationService` инжектирует и вызывает — вызов идёт через proxy.

2. **Инжектировать самого себя**:

```java
@Service
class NotificationService {
    @Autowired
    private NotificationService self;  // инжектируется proxy
    
    public void processAll() {
        for (var n : list) {
            self.processOne(n);  // через proxy!
        }
    }
}
```

Работает, но выглядит странно. Требует Spring 4.3+.

3. **`AopContext.currentProxy()`** — получить прокси из thread-local контекста:

```java
((NotificationService) AopContext.currentProxy()).processOne(n);
```

Требует `@EnableAspectJAutoProxy(exposeProxy = true)`. Работает, но не рекомендуется — больше сложности.

## Аннотации, работающие через прокси

Все AOP-based аннотации подвержены self-invocation проблеме:

- `@Transactional`
- `@Async`
- `@Cacheable`, `@CacheEvict`, `@CachePut`
- `@Retryable` (Spring Retry)
- `@Scheduled` (в некоторой мере)
- `@PreAuthorize`, `@PostAuthorize`, `@Secured` (Spring Security)
- Custom AOP аспекты через `@Aspect`

Правило: если метод помечен любой из этих аннотаций, вызывайте его только извне (через инжектированный bean), никогда через `this`.

## @Configuration и singleton гарантии

Классы `@Configuration` тоже оборачиваются CGLIB proxy — зачем?

Пример:

```java
@Configuration
class AppConfig {
    @Bean
    public DataSource ds() { 
        return new HikariDataSource(...); 
    }

    @Bean
    public FnoRepo repo() {
        return new FnoRepo(ds());  // вызов ds() — через прокси!
    }
}
```

`repo()` вызывает `ds()`. Если бы AppConfig был обычным классом, каждый вызов `ds()` создавал бы новый DataSource. Два разных экземпляра — плохо, конфликт.

Проксирование @Configuration решает это: proxy перехватывает вызов `ds()`, возвращает singleton уже зарегистрированный в контексте. Гарантирует что все `@Bean`-методы возвращают singleton, независимо от того как вызваны.

Есть `@Configuration(proxyBeanMethods = false)` — режим "lite", без проксирования. Быстрее (нет CGLIB overhead), но `ds()` внутри `repo()` создаст новый экземпляр. Использовать только если знаешь что делаешь, обычно в специальных случаях auto-configuration.

## Environment, Profiles, Properties

Spring предоставляет унифицированный доступ к конфигурации через **Environment**.

**Environment injection**:

```java
@Autowired
Environment env;

String url = env.getProperty("spring.datasource.url");
Integer maxSize = env.getProperty("app.max-size", Integer.class);
```

**`@Value` для отдельных значений**:

```java
@Value("${knp.max-batch-size:100}")
private int batchSize;   // дефолт 100 если свойство отсутствует
```

Синтаксис placeholder `${...}`. Двоеточие для дефолта.

**`@ConfigurationProperties` для группировки** — предпочтительный подход для сложной конфигурации:

```java
@ConfigurationProperties(prefix = "knp")
@Component
class KnpProperties {
    private int maxBatchSize = 100;
    private String outerSystemUrl;
    // getters/setters или использовать @Data от Lombok
}
```

Все свойства с префиксом `knp.` мапятся на поля класса. Type-safe (проверка типов при старте), IDE-подсказки, все свойства в одном классе, легко тестировать.

Использование:

```java
@Autowired
KnpProperties props;

// props.getMaxBatchSize(), props.getOuterSystemUrl()
```

**Profiles** — набор конфигураций для разных окружений:

```java
@Configuration
@Profile("prod")
class ProdDbConfig { ... }

@Configuration
@Profile({"dev", "test"})
class DevDbConfig { ... }
```

Активация:

```bash
java -jar app.jar --spring.profiles.active=prod
# или через env variable
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

`application-prod.yml`, `application-dev.yml` подхватываются автоматически при активном профиле.

В КНП: dev, test, preprod, prod, local. Стандартный набор для enterprise.

## Events

Spring имеет встроенный event mechanism для decoupling компонентов.

**Публикация**:

```java
@Autowired
ApplicationEventPublisher publisher;

// где-то в коде:
publisher.publishEvent(new FnoSubmittedEvent(fnoId));
```

**Подписка**:

```java
@Component
class FnoEventHandler {
    @EventListener
    public void onFnoSubmitted(FnoSubmittedEvent event) {
        log.info("Got fno {}", event.getFnoId());
        // обработка
    }
}
```

По умолчанию синхронно — publisher ждёт пока все listener'ы отработают.

Асинхронно через `@Async` + `@EnableAsync`:

```java
@EventListener
@Async
public void onFnoSubmitted(FnoSubmittedEvent event) {
    // выполняется в отдельном потоке
}
```

Полезно для decoupling. Сервис A публикует событие, не зная кто на него реагирует. Сервис B подписывается, реагирует. A и B не связаны напрямую.

## Полный пример: минимальный Spring Boot проект

Соберём всё вместе на маленьком примере.

```java
// Главный класс
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}

// Модель
class Fno {
    Long id;
    String regNum;
    // ...
}

// Интерфейс репозитория
interface FnoRepository {
    void save(Fno f);
    Optional<Fno> findByRegNum(String num);
}

// Реализация
@Repository
class JdbcFnoRepository implements FnoRepository {
    private final JdbcTemplate jdbc;

    public JdbcFnoRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public void save(Fno f) {
        jdbc.update("INSERT INTO fno(reg_num) VALUES (?)", f.regNum);
    }

    public Optional<Fno> findByRegNum(String num) {
        return jdbc.query("SELECT * FROM fno WHERE reg_num=?",
            (rs, i) -> {
                Fno f = new Fno();
                f.id = rs.getLong("id");
                f.regNum = rs.getString("reg_num");
                return f;
            }, num).stream().findFirst();
    }
}

// Сервис
@Service
class FnoService {
    private final FnoRepository repo;

    public FnoService(FnoRepository repo) {
        this.repo = repo;
    }

    @Transactional
    public void submit(Fno f) {
        if (repo.findByRegNum(f.regNum).isPresent()) {
            throw new IllegalStateException("duplicate");
        }
        repo.save(f);
    }
}

// REST контроллер
@RestController
@RequestMapping("/api/fno")
class FnoController {
    private final FnoService svc;

    public FnoController(FnoService svc) {
        this.svc = svc;
    }

    @PostMapping
    public void submit(@RequestBody Fno f) {
        svc.submit(f);
    }
}
```

Что происходит при старте:

1. Spring сканирует пакет `App`, находит `JdbcFnoRepository`, `FnoService`, `FnoController`.

2. Auto-configuration Spring Boot: определяет что есть DataSource в classpath, создаёт `HikariDataSource`, `JdbcTemplate`.

3. Строится граф зависимостей: DataSource → JdbcTemplate → JdbcFnoRepository → FnoService → FnoController.

4. Создаются beans в правильном порядке. FnoService оборачивается в proxy из-за `@Transactional`.

5. FnoController регистрируется в MVC infrastructure.

6. Стартует embedded Tomcat на 8080.

Приложение работает. POST на `/api/fno` вызывает Controller.submit → Service.submit (через proxy, открывающий транзакцию) → Repository.save (реальный SQL) → COMMIT (после успешного выхода из метода).

## Заключение

Spring Framework — фундамент enterprise Java. IoC контейнер снимает с разработчика заботу о создании объектов, DI позволяет писать слабо связанный тестируемый код. Понимание внутренностей — beans, lifecycle, scopes, proxies, configuration — критично для работы с любым нетривиальным приложением.

Ключевые практические принципы. Constructor injection всегда, никакой field injection в новом коде. `implementation` scope для зависимостей. Understanding proxy mechanics — почему `@Transactional` не работает при self-invocation, как решить. Careful use of `@Bean` methods в `@Configuration` классах, понимание proxy-based singleton гарантий. Правильное использование profiles для окружений, `@ConfigurationProperties` для группированной конфигурации.

Основные gotcha, о которых нужно помнить. Self-invocation ломает AOP-based аннотации — самая частая production проблема. Prototype в singleton даёт всегда одинаковый экземпляр — использовать ObjectProvider. Циклические зависимости — refactor, не workaround через @Lazy. `@Configuration` proxying для singleton гарантий — понимание чтобы не сломать через `proxyBeanMethods = false`.

Для КНП контекста особо релевантно. Все микросервисы Spring Boot — знание Spring обязательно. `@Transactional` для управления транзакциями в JPA/Hibernate. Feign clients — генерируются как proxy interfaces. Consul integration через Spring Cloud аннотации. Profiles для разделения dev/test/preprod/prod окружений.

Дальше — практика. Возьми любой существующий сервис, посмотри как он структурирован (какие beans, какие зависимости, какие профили). Попробуй написать простую конфигурацию через `@Configuration`. Реализуй custom AOP-аспект (например, для логирования method execution time). Разбери реальный случай self-invocation в коде команды. Каждое такое упражнение поднимает понимание Spring на новый уровень.
