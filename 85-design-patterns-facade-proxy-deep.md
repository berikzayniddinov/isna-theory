# 85. Facade и Proxy: два паттерна, на которых стоит Spring

## Зачем это знать

Facade и Proxy — два из 23 GoF паттернов, но именно они настолько распространены в enterprise Java, что понимание их — не «академическое знание паттернов», а прямая эксплуатационная необходимость. Каждая аннотация Spring (`@Transactional`, `@Async`, `@Cacheable`, `@PreAuthorize`, `@Retryable`) работает через Proxy. Каждый `JdbcTemplate`, `RestTemplate`, `RabbitTemplate` — это Facade. Понимание этих двух паттернов даёт понимание того как Spring реально работает под капотом — и почему аннотация иногда не срабатывает, почему возникает `LazyInitializationException`, почему `@Transactional` на private методе игнорируется.

Разница между «слышал про паттерны» и «понимаю их применение» — это способность за секунду ответить: почему `@Transactional` не работает при вызове `this.otherMethod()` внутри того же класса. Почему Spring Boot по умолчанию использует CGLIB, а не JDK Proxy. Что произойдёт если пометить `@Transactional` класс `final`. Почему для `@Cacheable` порядок важен относительно `@Transactional`. Каждый из этих вопросов — прямое применение знания механики proxy.

Разберём: ментальную модель — оба паттерна вводят «прослойку» между клиентом и объектом, но с разной целью (Facade упрощает, Proxy контролирует). Facade — мотивация, механика, JDK/Spring примеры (JdbcTemplate, RestTemplate), масштабирование на архитектуру (BFF, API Gateway), различия с Adapter/Mediator/Decorator, anti-pattern God Facade. Proxy — виды (virtual, protection, remote, smart), static vs dynamic. Глубоко: JDK Dynamic Proxy как работает (Proxy.newProxyInstance генерирует `$Proxy0`, кэширует, ограничения — только интерфейсы). CGLIB — bytecode manipulation через subclass, ограничения (не final, не private, не static). ByteBuddy как современная альтернатива. Spring AOP — как выбирает JDK vs CGLIB, механика `@Transactional` через `TransactionInterceptor`, self-invocation problem как классическая ловушка и четыре способа обойти. Прочие Spring-аннотации (`@Async`, `@Cacheable`, `@PreAuthorize`, `@Retryable`) — все proxy-based, порядок interceptor'ов важен. Разница Proxy vs Decorator (контроль доступа vs добавление функциональности, invisible vs client-composed). Практика: когда что использовать, какие ошибки типичны, как отлаживать.

Понятие абстракции как таковой — в 113. `@Transactional` внутренности — в 33. Spring аннотации — в 60. Здесь фокус на самих паттернах и их реализации.

## Ментальная модель

Оба паттерна вводят промежуточный объект между клиентом и настоящей реализацией. Отличаются целью и видимостью для клиента.

**Facade** предоставляет новый упрощённый интерфейс к сложной подсистеме. Клиент **знает** что за фасадом много всего — просто не хочет с этим разбираться напрямую. Метод `orderFacade.placeOrder(...)` явно намекает «здесь сложная операция». Клиент не пишет `if (inventory.available()) { payment.charge(...); shipping.book(...); }` — он вызывает фасад, который координирует всё.

**Proxy** имеет **тот же интерфейс** что и настоящий объект. Клиент **не знает** что говорит с proxy — думает что напрямую с real object. Прокси прозрачно вставляется в вызов и добавляет своё поведение (кэш, security, lazy loading, транзакция, удалённый вызов). `userService.findById(1L)` — клиент не в курсе что перед реальным `findById` был проверен cache, открыта транзакция и залогирован аудит.

Ключевое различие: **Facade — про упрощение (видимо), Proxy — про контроль (невидимо)**. Клиент Facade знает что фасад скрывает сложность. Клиент Proxy думает что имеет дело с оригиналом.

## Facade: когда есть сложная подсистема

Мотивация проста. Есть подсистема — 5-10 классов, каждый со своим API. Типовая операция требует вызовов нескольких классов в правильном порядке, с правильной обработкой ошибок и compensation при частичных сбоях. Каждый клиент, желающий выполнить эту типовую операцию, вынужден повторять всю оркестрацию. Изменение подсистемы (добавили аудит, изменили порядок вызовов) — приходится править всех клиентов.

Классический пример — обработка заказа. Без фасада контроллер знает про пять сервисов, все compensation flows, все edge cases:

```java
public OrderResult placeOrder(OrderRequest req) {
    if (!inventory.hasStock(req.getItems())) return OrderResult.error("...");
    ReservationId reservation = inventory.reserve(req.getItems());
    try {
        PaymentResult payResult = payment.charge(req.getUserId(), req.getTotalAmount());
        if (!payResult.isSuccess()) {
            inventory.releaseReservation(reservation);
            return OrderResult.error("Payment failed");
        }
        ShipmentId shipment;
        try {
            shipment = shipping.createShipment(req);
        } catch (ShippingException e) {
            payment.refund(payResult.getTransactionId());
            inventory.releaseReservation(reservation);
            throw e;
        }
        notification.sendOrderConfirmation(req.getUserId(), shipment);
        audit.log("ORDER_PLACED", req.getUserId(), reservation, shipment);
        return OrderResult.success(shipment);
    } catch (Exception e) {
        audit.log("ORDER_FAILED", req.getUserId(), e.getMessage());
        throw e;
    }
}
```

Сто строк логики в контроллере. Копия в каждом месте где нужно `placeOrder` (другой endpoint, batch job, admin panel). Изменения (например, новый шаг — начисление бонусов) — править везде.

С фасадом всё меняется. Вся оркестрация — в одном классе `OrderFacade`, контроллер тонкий:

```java
@Service
public class OrderFacade {
    private final InventoryService inventory;
    private final PaymentGateway payment;
    private final ShippingService shipping;
    private final NotificationService notification;
    private final AuditLog audit;

    public OrderResult placeOrder(OrderRequest req) {
        // вся сложная логика тут (одно место)
    }
}

@RestController
public class OrderController {
    private final OrderFacade orderFacade;
    
    @PostMapping("/orders")
    public OrderResult place(@RequestBody OrderRequest req) {
        return orderFacade.placeOrder(req);
    }
}
```

Facade **не является членом подсистемы** — это отдельный класс, знающий про неё. Клиент **может** пользоваться и напрямую подсистемой, если нужен fine-grained control (например, административная панель, которой нужно резервирование без немедленной оплаты). Facade не запрещает прямой доступ, он **предлагает** удобный.

## Facade в JDK и Spring

Спринг-разработчик работает с фасадами постоянно, часто не задумываясь.

**`java.net.URL`** — фасад над `URLConnection`, `HttpClient`, sockets, DNS resolver. `new URL("...").openStream()` — одна строка, за ней десяток классов и вся сетевая машинерия.

**`JOptionPane.showMessageDialog()`** — фасад над JDialog, JLabel, JButton, Frame, layout managers. Всё разложено на классы, но для типовой «покажи messagebox» одна статическая функция.

**Spring `JdbcTemplate`** — фасад над JDBC (Connection, Statement, ResultSet, обработка ошибок, cleanup). Raw JDBC требует правильно управлять всеми ресурсами через try-with-resources, конвертировать SQLException в понятные типы, mapping'и. JdbcTemplate прячет всё это:

```java
User u = jdbc.queryForObject(
    "SELECT * FROM users WHERE id=?", 
    userRowMapper, id
);
```

Против raw JDBC в 15 строк с открытием/закрытием connection, prepared statement, result set. Одинаковая работа, разная эргономика.

**Spring `RestTemplate` / `WebClient`** — фасад над HTTP-клиентом. Спрятаны connection pooling, timeouts, error handling, serialization.

**Spring `RabbitTemplate` / `KafkaTemplate`** — фасад над messaging (channels, connections, acknowledgments, retries).

**Spring `TransactionTemplate`** — фасад над `PlatformTransactionManager` для программного управления транзакциями.

Общий паттерн: все шаблоны Spring `*Template` — это **Facade + Template Method**. Скелет операции фиксирован (Template Method), доступ упрощён (Facade).

## Facade в микросервисах: BFF и API Gateway

Идея масштабируется на уровень системы.

**Backend for Frontend (BFF)** — микросервис-фасад перед N доменными сервисами. Каждому UI (мобильное приложение, веб, админ-панель) — свой BFF, агрегирующий нужные ему данные из нескольких сервисов. Мобильному приложению нужен user info + recent orders + notifications в одном запросе — вместо трёх вызовов из клиента, один вызов в BFF, который сам делает три параллельных запроса к доменным сервисам и собирает ответ.

**API Gateway** (nginx, Kong, Spring Cloud Gateway) — сетевой фасад перед всеми backend'ами. Единая точка входа, TLS termination, аутентификация, rate limiting, routing.

Отличие BFF от Gateway. Gateway тонкий — только cross-cutting concerns (auth, rate limit, routing). BFF толстый — содержит бизнес-агрегацию для конкретного клиента.

В КНП: `isnaknpgateway` — Gateway (Spring Cloud Gateway или nginx-based), тонкий сетевой фасад.

## Facade против Adapter, Mediator, Decorator

Эти паттерны часто путают на собеседовании, разница принципиальна.

**Adapter** делает **несовместимое совместимым**. Есть класс с интерфейсом X, клиент ожидает интерфейс Y. Adapter — обёртка, конвертирующая X → Y.

```java
class LegacyLogger {
    void writeMessage(String severity, String message) { }
}

interface Logger {
    void info(String msg);
    void error(String msg);
}

class LegacyLoggerAdapter implements Logger {
    private final LegacyLogger legacy;
    public void info(String msg) { legacy.writeMessage("INFO", msg); }
    public void error(String msg) { legacy.writeMessage("ERROR", msg); }
}
```

Adapter меняет форму интерфейса (1-to-1). Facade объединяет N сервисов за одним упрощённым API (N-to-1). Adapter говорит «переведи с языка X на Y». Facade говорит «делай сложные вещи одной командой».

**Mediator** координирует **peer-to-peer** взаимодействие между **равными** участниками. Никто не главный, mediator в центре. Пример: чат-комната — каждый пользователь пишет в комнату, комната рассылает всем. Facade **однонаправленный** — клиент вызывает фасад, фасад вызывает подсистему. Клиент **выше** уровнем чем подсистема. Mediator — про decoupling peers, Facade — про упрощение доступа.

**Decorator** — про добавление ответственности, а не упрощение. Оба паттерна оборачивают, но Decorator сохраняет тот же интерфейс что и wrapped, Facade имеет свой уникальный. Разбор Decorator подробнее ниже, в контексте сравнения с Proxy.

## Anti-pattern: God Facade

Опасность фасада — он становится «God object» с 30 методами, знающий всё про всю систему.

Признаки: 500+ строк в фасаде, множественные несвязанные операции в одном классе (`placeOrder`, `updateUserProfile`, `generateReport`), все запросы приложения идут через один фасад, тесты с десятками mock'ов.

Fix — разделить по доменам: `OrderFacade`, `UserFacade`, `ReportFacade`. Внутри фасада делегировать бизнес-логику **application services** (в терминах DDD). Facade — только orchestration, не бизнес-правила. Расчёт скидки не в фасаде, а в `DiscountService`, к которому фасад обращается.

Правило: начинай без фасада, добавляй когда чувствуешь боль. Facade — рефакторинг, не изначальный дизайн.

## Facade: когда не использовать

Подсистема из 2-3 методов — прямой вызов проще, фасад — overhead. Клиент реально нуждается в fine-grained control (например, performance-critical low-level DB access, где JdbcTemplate добавляет неприемлемый overhead). Фасад добавляет indirection без упрощения — просто forwarder методов один-в-один.

## Proxy: контроль доступа к объекту

Мотивация Proxy — есть объект, к которому нужно контролировать доступ. Причины разнообразны:

1. **Ленивая инициализация** — real object дорого создавать, откладываем до первого использования.
2. **Контроль доступа** — auth check перед вызовом.
3. **Логирование / audit** — записать что кто вызвал.
4. **Кэширование** — вернуть cached если есть, иначе позвать real.
5. **Транзакции** — открыть/закрыть транзакцию вокруг метода.
6. **Remote invocation** — real object на другой машине, proxy общается по сети.
7. **Reference counting / smart pointer** — вести счётчик использований.
8. **Read/write разделение** — proxy шлёт reads на реплику, writes на master.

Общая идея: **proxy имеет тот же интерфейс что real object**, но добавляет поведение. Клиент вызывает proxy думая что общается с настоящим объектом.

Формальное определение GoF: *Provide a surrogate or placeholder for another object to control access to it*.

## Виды Proxy

Классификация по цели контроля.

**Virtual Proxy** — ленивое создание. Пока никто не вызвал реальную операцию — real object не существует. Экономит ресурсы если объект дорогой, но не всегда используется.

```java
class ImageProxy implements Image {
    private final String filename;
    private RealImage realImage;   // lazy

    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename);   // expensive: reads file
        }
        realImage.display();
    }
}
```

Hibernate lazy loading — классический virtual proxy. `@ManyToOne(fetch = FetchType.LAZY)` возвращает не User, а proxy. Пока не обратился к полю (`user.getName()`) — реальный SELECT не выполнен. Минус — доступ вне scope сессии (после `session.close()`) → `LazyInitializationException`, потому что proxy пытается сделать query а session уже закрыт.

**Protection Proxy** — контроль доступа по правам. Проверяет authorization перед делегированием.

```java
class SecureBankAccount implements BankAccount {
    private final BankAccount real;
    private final User currentUser;
    
    public void transfer(long amount, String to) {
        if (!currentUser.hasRole("ADMIN") && amount > 10000) {
            throw new SecurityException("Large transfer requires admin");
        }
        real.transfer(amount, to);
    }
}
```

Spring Security `@PreAuthorize` — protection proxy. Читает `Authentication` из `SecurityContext`, evaluates SpEL expression, при false бросает `AccessDeniedException`.

**Remote Proxy** — real object на другой машине. Proxy сериализует args, шлёт по сети, десериализует response.

```java
class UserServiceStub implements UserService {   // Feign / gRPC generated
    public User getUser(Long id) {
        HttpResponse resp = httpClient.get("http://user-service/users/" + id);
        return jackson.readValue(resp.body(), User.class);
    }
}
```

Feign clients, gRPC stubs, RMI, EJB remote — всё remote proxy. Клиент `@Autowired UserServiceClient` не подозревает что под капотом HTTP.

**Smart Proxy** — дополнительное поведение: reference counting, кэширование, логирование, метрики.

```java
class CachingUserServiceProxy implements UserService {
    private final UserService real;
    private final Cache<Long, User> cache;
    
    public User getUser(Long id) {
        return cache.get(id, () -> real.getUser(id));
    }
}
```

Spring `@Cacheable` — smart proxy. `Collections.synchronizedList(list)` — тоже smart proxy (synchronization).

Есть ещё редкие варианты: **Copy-on-Write Proxy** (real object shared, proxy делает копию только при modification для оптимизации памяти), **Firewall Proxy** (network-level protection), **Synchronization Proxy** (thread-safe wrapper).

## Static vs Dynamic Proxy

**Static Proxy** — пишешь proxy-класс руками. Как все примеры выше. Явно видно что происходит, compile-time type safety, легко отлаживать. Минус — boilerplate: N методов интерфейса × 2-3 строки delegate каждый. Изменил интерфейс — правь proxy. Не работает для универсального case («любой сервис с @Transactional»).

**Dynamic Proxy** — класс proxy создаётся на лету (в runtime или compile-time через bytecode manipulation). Обрабатывает **любой** метод через центральный `InvocationHandler` / `MethodInterceptor`. Не надо писать boilerplate — интерфейс и логика interceptor'а достаточно.

Две основные реализации в Java: **JDK Dynamic Proxy** (`java.lang.reflect.Proxy`) — только для интерфейсов, встроен в JDK. **CGLIB** — bytecode manipulation, работает с классами через наследование. Spring использует обе для AOP, `@Transactional`, `@Async`, `@Cacheable`, `@PreAuthorize`.

## JDK Dynamic Proxy изнутри

API прост:

```java
Object proxy = Proxy.newProxyInstance(
    classLoader,
    new Class<?>[]{MyInterface.class},
    (proxy, method, args) -> {
        // ДО, ПОСЛЕ, ВОКРУГ
        return method.invoke(realObject, args);
    }
);

MyInterface p = (MyInterface) proxy;
p.doSomething();
```

Что делает JDK при первом вызове `Proxy.newProxyInstance()` для данной комбинации интерфейсов:

1. **Генерирует bytecode** класса `$Proxy0` (или следующий номер), который implements все указанные интерфейсы, extends `java.lang.reflect.Proxy`, для каждого метода интерфейса имеет реализацию вызывающую `handler.invoke(this, method, args)`.
2. **Кэширует** класса в `ProxyGenerator` — второй раз для тех же интерфейсов используется тот же класс.
3. **Загружает** через указанный ClassLoader.
4. **Instantiates** proxy object.

Сгенерированный класс выглядит примерно так:

```java
public final class $Proxy0 extends Proxy implements MyInterface {
    private static Method m_doSomething;
    // ...
    
    static {
        m_doSomething = MyInterface.class.getMethod("doSomething");
    }
    
    public $Proxy0(InvocationHandler h) { super(h); }
    
    @Override
    public void doSomething() {
        try {
            h.invoke(this, m_doSomething, null);
        } catch (Throwable t) {
            throw new UndeclaredThrowableException(t);
        }
    }
    // hashCode, equals, toString тоже delegated в handler
}
```

Посмотреть сгенерированные классы можно через `-Djdk.proxy.ProxyGenerator.saveGeneratedFiles=true` — сохранит `$Proxy0.class` в текущей директории.

Плюсы JDK Proxy: встроен в JDK, нет зависимостей. Быстрая генерация класса. Простая ментальная модель — proxy implements interface. Минусы: **только для интерфейсов**. Если у тебя `class UserService` без интерфейса — JDK proxy не сработает. Reflection overhead на `method.invoke()` (значительно меньше в JDK 9+, но всё ещё дороже direct call).

**Ключевое ограничение**: работает только с методами объявленными в интерфейсе. Если class implements MyInterface + добавляет public method не из интерфейса — proxy имеет только методы из MyInterface:

```java
interface UserService {
    User findById(Long id);
}

class UserServiceImpl implements UserService {
    public User findById(Long id) { ... }
    public User findByEmail(String email) { ... }   // не в интерфейсе!
}

UserService proxy = (UserService) Proxy.newProxyInstance(...);
proxy.findById(1L);         // ok
((UserServiceImpl) proxy).findByEmail("...");   // ClassCastException!
// Proxy НЕ instanceof UserServiceImpl
```

## CGLIB: bytecode generation через subclass

Что если класс не имеет интерфейса? JDK proxy бесполезен. Тут появляется CGLIB (Code Generation Library) — генерирует **subclass** твоего класса, overriding методы через `MethodInterceptor`.

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UserService.class);
enhancer.setCallback((MethodInterceptor) (obj, method, args, methodProxy) -> {
    System.out.println("Called: " + method.getName());
    return methodProxy.invokeSuper(obj, args);
});
UserService proxy = (UserService) enhancer.create();
```

CGLIB создаёт класс типа `UserService$$EnhancerByCGLIB$$abc123`:
- Extends UserService.
- Overrides каждый non-final public method.
- Каждый override делегирует в MethodInterceptor.

Схематично:

```java
public class UserService$$EnhancerByCGLIB extends UserService {
    private MethodInterceptor interceptor;
    
    @Override
    public User findById(Long id) {
        return (User) interceptor.intercept(this, 
            /* Method obj */, new Object[]{id}, /* MethodProxy */);
    }
    // ... все другие public methods
}
```

`MethodProxy.invokeSuper()` вызывает `super.findById(id)` через быстрый bypass (сгенерированный FastClass, не reflection) — CGLIB быстрее JDK Proxy на hot path.

Плюсы CGLIB: работает без интерфейса. Быстрее JDK Proxy для method invocation. Более гибкий (перехват hashCode, equals, toString раздельно).

Минусы: **не может proxy final classes** (нельзя extends). **Не может proxy final methods** (нельзя override). **Не может proxy private methods** (нельзя override). Конструктор parent класса вызывается при создании proxy — если parent имеет side effects, они выполнятся; часто нужен default constructor или Objenesis для обхода. Дополнительная зависимость (в Spring 5+ CGLIB встроен в `spring-core` в repackaged форме `org.springframework.cglib.*`).

**Классическая ловушка CGLIB и final**:

```java
class MyService {
    public final String getName() { return "..."; }   // final!
    public void doWork() { }
}

MyService proxy = /* CGLIB proxy */;
proxy.doWork();     // intercepted
proxy.getName();    // вызывается напрямую, БЕЗ interceptor
```

`@Transactional` метод помеченный `final` — proxy не может override → аннотация игнорируется. Silent bug — код компилируется, работает, но транзакция не открывается.

## ByteBuddy: современная альтернатива

ByteBuddy — библиотека для bytecode manipulation. Более гибкая и быстрая чем CGLIB. Активно развивается (CGLIB заброшена, последний релиз ~2019). Используется Mockito, Hibernate, некоторые Spring internals в новых версиях.

```java
Class<?> dynamicType = new ByteBuddy()
    .subclass(UserService.class)
    .method(ElementMatchers.named("findById"))
    .intercept(MethodDelegation.to(new MyInterceptor()))
    .make()
    .load(UserService.class.getClassLoader())
    .getLoaded();

UserService proxy = (UserService) dynamicType
    .getDeclaredConstructor().newInstance();
```

Плюсы: type-safe DSL, работает с Java 21+ модулями, активно поддерживается, лучшая performance. Минусы: отдельная зависимость, более сложный API.

Spring может переключиться на ByteBuddy в будущих версиях, но пока CGLIB — default.

## Spring AOP: практика Proxy

Spring использует Proxy pattern для всех аспектов:

- `@Transactional` → `TransactionInterceptor`.
- `@Async` → `AsyncExecutionInterceptor`.
- `@Cacheable` / `@CacheEvict` → `CacheInterceptor`.
- `@PreAuthorize` / `@Secured` → `MethodSecurityInterceptor`.
- `@Retryable` (Spring Retry) → `RetryInterceptor`.
- Custom aspects через `@Aspect` + `@Around` / `@Before` / `@After`.

При bean creation Spring: создаёт real object → проверяет есть ли применимые advisors (например `@Transactional` на методе) → если да, оборачивает в proxy → injects proxy в других beans, не real. Клиент `@Autowired UserService us` получает **proxy**, не real object.

**Выбор JDK vs CGLIB**. Bean implements interface → JDK Proxy (default в некоторых конфигурациях). Bean не implements → CGLIB. Force CGLIB через `spring.aop.proxy-target-class=true` или `@EnableTransactionManagement(proxyTargetClass = true)`. **Spring Boot с 2.0 по умолчанию использует CGLIB** — упрощает жизнь разработчику (не надо думать про интерфейсы, всегда работает одинаково).

## Как работает `@Transactional` через Proxy

```java
@Service
public class UserService {
    @Autowired UserRepository repo;
    
    @Transactional
    public User update(Long id, String name) {
        User u = repo.findById(id).orElseThrow();
        u.setName(name);
        return repo.save(u);
    }
}
```

Spring оборачивает `UserService` в CGLIB proxy. Клиент получает `UserService$$EnhancerBySpringCGLIB$$abc`.

При вызове `userService.update(1L, "new name")`:

1. Proxy `update(...)` перехватывает вызов.
2. `TransactionInterceptor.invoke()`:
   - Читает `@Transactional` метаданные (propagation, isolation, readOnly, timeout).
   - Через `PlatformTransactionManager` открывает транзакцию:
     - `dataSource.getConnection()`.
     - `conn.setAutoCommit(false)`.
     - Binds connection в `TransactionSynchronizationManager` (ThreadLocal).
3. Вызывает `super.update(1L, "new name")` (real метод).
4. Real метод исполняется, использует connection из ThreadLocal.
5. После return:
   - Если exception (RuntimeException / Error) → `conn.rollback()`.
   - Иначе → `conn.commit()`.
6. `conn.close()` (реально возвращается в HikariCP pool).

Клиент не знает про транзакцию. Аннотация + proxy делают всё. Детально — файл 33 про transactional internals.

## Self-invocation problem: классическая ловушка №1

Одна из самых частых ошибок работы с Spring proxy — вызов transactional метода изнутри того же класса:

```java
@Service
public class UserService {
    
    public void publicMethod() {
        internalMethod();   // прямой вызов, не через proxy!
    }
    
    @Transactional
    public void internalMethod() {
        // ...
    }
}
```

`publicMethod` вызывает `internalMethod` через `this` (не через proxy). Транзакция **не открывается** — proxy не в пути вызова. Silent bug — код работает, но без транзакционных гарантий.

Почему так? Proxy оборачивает bean снаружи. Внутренний вызов `this.internalMethod()` — прямая ссылка на real object, минует proxy.

```
Client → Proxy → real UserService.publicMethod()
                          │
                          ▼ (this.internalMethod — прямой вызов на real)
                     UserService.internalMethod()   ← proxy не в пути
                     @Transactional игнорируется!
```

Четыре способа обойти:

**1. Self-injection** — уродливо, но работает. Spring injects proxy сам к себе:

```java
@Autowired UserService self;

public void publicMethod() {
    self.internalMethod();   // через proxy
}
```

**2. `AopContext`** — доступ к текущему proxy через ThreadLocal:

```java
@EnableAspectJAutoProxy(exposeProxy = true)

public void publicMethod() {
    ((UserService) AopContext.currentProxy()).internalMethod();
}
```

Требует включения `exposeProxy = true`, что не default.

**3. Разделить на два бина** — часто самое чистое решение:

```java
@Service 
class OrderFacade {
    @Autowired OrderService orderService;
    public void placeOrder() {
        orderService.doTransactionalWork();   // через proxy
    }
}

@Service 
class OrderService {
    @Transactional
    public void doTransactionalWork() { }
}
```

**4. AspectJ** (compile-time weaving, не proxy) — работает без proxy indirection. Аспект встраивается прямо в bytecode. Self-invocation работает. Но сложная настройка (compile-time агент, отдельная configuration).

Правило: `@Transactional` работает только для внешних вызовов через proxy. Внутренние вызовы (`this.xxx()`) не пойдут через interceptor.

## Другие ограничения Spring Proxy

Все ограничения вытекают из механики proxy — интерсептируется только то, что видно снаружи.

**private methods** — не proxy'ятся. Ни JDK Proxy (интерфейс не имеет private), ни CGLIB (private нельзя override). `@Transactional` на private методе — silent no-op.

**final methods** (при CGLIB) — не proxy'ятся, нельзя override.

**final classes** — не могут быть CGLIB-обёрнуты. `BeanCreationException` при старте.

**static methods** — не proxy'ятся вообще. Static не привязан к экземпляру, proxy — instance-based.

**package-private methods** — technically могут proxy'иться CGLIB, но зависит от версии и настроек Spring. Не полагайся.

Правило: `@Transactional`, `@Async`, `@Cacheable` только на **public** методах, non-final.

Особая история с Kotlin — все классы `final` by default. Kotlin plugin `all-open` автоматически открывает `@Component` классы для Spring proxy.

## Отладка: как понять что там proxy и что реально происходит

Быстрая проверка наличия proxy через `bean.getClass().getName()`:

```java
@Autowired UserService us;

@PostConstruct
void init() {
    System.out.println(us.getClass().getName());
    // Возможные варианты:
    //   UserService$$EnhancerBySpringCGLIB$$abc123    ← CGLIB proxy
    //   com.sun.proxy.$Proxy42                          ← JDK proxy
    //   com.example.UserService                         ← real (no proxy)
}
```

Если видишь имя оригинального класса без `EnhancerBy...` или `$Proxy...` — proxy не создан, аннотация не сработает. Причин может быть несколько: класс не является Spring bean (`new UserService()` вместо `@Autowired`); класс `final` (CGLIB не может extends); нет ни одной аспект-аннотации на bean; bean создаётся factory-методом который возвращает не proxy; `spring.aop.auto=false` в конфигурации.

**Проверка что вызов действительно прошёл через interceptor**. Поставить breakpoint в `org.springframework.transaction.interceptor.TransactionInterceptor.invoke()` (для @Transactional) или `AsyncExecutionInterceptor.invoke()` (для @Async) — при вызове метода breakpoint должен сработать. Если не срабатывает — либо proxy не создан, либо self-invocation, либо метод private/final/static.

**Stacktrace-анализ**. При исключении внутри `@Transactional` метода полный stacktrace должен содержать примерно такую цепочку:

```
at UserService.update(UserService.java:42)              ← real метод
at UserService$$FastClassBySpringCGLIB.invoke(...)      ← CGLIB dispatch
at org.springframework.cglib.proxy.MethodProxy.invoke   ← CGLIB proxy machinery
at org.springframework.aop.framework.CglibAopProxy...   ← Spring proxy adapter
at TransactionInterceptor.invokeWithinTransaction       ← вот здесь tx открылась
at TransactionInterceptor.invoke(TransactionInterceptor.java:...)
at ReflectiveMethodInvocation.proceed(...)              ← chain of advisors
at UserService$$EnhancerBySpringCGLIB$$abc.update(...)  ← точка входа
at UserController.updateUser(UserController.java:...)   ← клиент
```

Ключевые фреймы — `TransactionInterceptor.invokeWithinTransaction` (транзакция реально открыта) и `EnhancerBySpringCGLIB` (клиент попал в proxy). Если этих фреймов нет — proxy не сработал.

**Actuator endpoint `/actuator/beans`** покажет все bean'ы с их фактическими типами. Для больших приложений это единственный способ быстро узнать какие бины прокси'нуты, а какие нет.

**Отладка через `-Xlog:jni+resolve=debug`** или через `-verbose:class` при старте — покажет когда и какие $Proxy / EnhancerByCGLIB классы генерируются. Полезно понять точку создания.

**`AopUtils.isAopProxy(bean)`** — utility метод Spring для программной проверки. `AopUtils.isJdkDynamicProxy(bean)` и `AopUtils.isCglibProxy(bean)` — уточнить тип. `AopProxyUtils.ultimateTargetClass(bean)` — получить оригинальный класс скрытый за proxy (полезно в тестах).

## AspectJ vs Spring AOP: реальные trade-off'ы

Spring AOP — proxy-based, с ограничениями обсуждёнными выше (public non-final, no self-invocation, no private, no static). AspectJ — bytecode weaving, работает **без** proxy, все ограничения снимаются.

**Compile-time weaving (CTW)**. Специальный компилятор `ajc` (или `iajc` Maven/Gradle plugin) заменяет обычный `javac`. Читает исходники Java + AspectJ (`.aj` файлы или `@Aspect` в обычных классах), генерирует `.class` файлы с уже встроенными аспектами. Bytecode уже содержит вставки — при загрузке нет runtime overhead на dispatch.

Выигрыш производительности: 5-15% по сравнению с proxy-based (нет reflection, нет через-interceptor dispatch, JIT inlining работает лучше на плоском bytecode). Минус: build медленнее (ajc ощутимо медленнее javac), IDE integration хуже (не все IDE понимают .aj-файлы), диагностика сложнее (стектрейсы содержат синтезированные методы типа `foo_aroundBody0`).

**Load-time weaving (LTW)**. Java agent модифицирует bytecode прямо при `ClassLoader.defineClass()`. Обычный javac, но при запуске JVM с `-javaagent:aspectjweaver.jar` byte code изменяется on the fly. Плюс: build обычный, минус: overhead на classloading (первый раз каждый класс проходит через weaver), сложнее debugging (stack traces с weaved-методами).

**Когда AspectJ имеет смысл**:

- **Self-invocation необходимо**. Legacy код где рефакторинг на два бина невозможен, а `@Transactional` нужно внутри одного класса.
- **Аспекты на private / final методах**. Например аудит-логирование всех методов класса включая private helpers.
- **Аспекты на конструкторах**. Proxy не может перехватить `new UserService()`. AspectJ — может (через `execution(UserService.new(..))`).
- **Аспекты на static методах**. То же, proxy не работает, AspectJ работает.
- **Аспекты на final классах / не Spring beans**. Например, логировать все методы JDK классов (не рекомендуется, но технически возможно с AspectJ).
- **Performance-critical hot paths** — 5-15% выигрыш JIT inlining vs interceptor dispatch стоит сложности.

Для типичного Spring enterprise приложения Spring AOP достаточно. AspectJ — по конкретной необходимости, не «по умолчанию». Сложность настройки и debugging обычно не окупается.

Комбинация возможна: Spring использует `@AspectJ` синтаксис (`@Aspect`, `@Around`, pointcut expressions) но с proxy-механикой. То есть аннотации те же, а под капотом proxy. Полный AspectJ включается через `<aop:aspectj-autoproxy proxy-target-class="true"/>` + `-javaagent:aspectjweaver.jar`.

## @Async: механика, executors, return types, virtual threads

`@Async` — аннотация для асинхронного выполнения метода. Proxy при вызове не выполняет метод immediately — оборачивает в `Runnable`/`Callable`, submit в `TaskExecutor`, возвращает `Future`/`CompletableFuture`/void. Реальный метод выполняется в другом треде.

**Требования на возвращаемый тип**:

- **`void`** — fire-and-forget. Клиент вызвал и забыл, никаких гарантий на результат или исключение (exceptions глотаются, дефолтно логируются `AsyncUncaughtExceptionHandler`, кастомизируется).
- **`Future<T>`** — старый Java API. Клиент может `.get()` для дождаться, `.cancel()` для отмены. Устаревающий вариант, обычно избегается.
- **`CompletableFuture<T>` / `ListenableFuture<T>`** — modern. Композиция через `.thenApply`, `.thenCompose`, `.exceptionally`. Стандарт для новых async методов.
- Любой другой тип — Spring запустит асинхронно, но return value будет игнорирован (получишь `null` синхронно). Обычно baг.

**Настройка TaskExecutor**. Дефолтный `SimpleAsyncTaskExecutor` создаёт **новый тред на каждый вызов** — категорически неприемлемо в prod (можно исчерпать треды за минуты). Надо явно настроить:

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(
            new ThreadPoolExecutor.CallerRunsPolicy()   // fallback
        );
        executor.initialize();
        return executor;
    }
    
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async method {} failed", method.getName(), ex);
        };
    }
}
```

Ключевые параметры. `corePoolSize` — базовое число тредов, всегда живых. `maxPoolSize` — максимум при нагрузке. `queueCapacity` — размер очереди задач когда все core threads заняты. `rejectedExecutionHandler` — что делать когда и очередь заполнена, и maxPoolSize достигнут. `CallerRunsPolicy` — выполнить в calling thread (даёт backpressure). `AbortPolicy` (дефолт) — бросить `RejectedExecutionException` (клиент падает). `DiscardPolicy` — молча выбросить (обычно плохо, теряются задачи).

**Множество executors для разных задач**. Часто в приложении несколько классов async работ: быстрые (email отправка) и тяжёлые (генерация отчётов). Смешивать в одном пуле — тяжёлые блокируют быстрые. Правильно — разные executors:

```java
@Bean("emailExecutor")
Executor emailExecutor() { /* small pool */ }

@Bean("reportExecutor")
Executor reportExecutor() { /* large pool with big queue */ }

// Использование:
@Async("emailExecutor")
public void sendEmail(...) { }

@Async("reportExecutor")
public CompletableFuture<Report> generateReport(...) { }
```

**Virtual threads (Java 21+)** меняют картину. Вместо platform thread pool можно использовать unbounded virtual thread executor:

```java
@Bean
AsyncTaskExecutor asyncTaskExecutor() {
    return new TaskExecutorAdapter(
        Executors.newVirtualThreadPerTaskExecutor()
    );
}
```

Каждый `@Async` вызов создаёт новый virtual thread — миллионы возможны (см. файл 110). Никаких пулов, queues, rejection policies. Но остаются ограничения VT — pinning на synchronized (до Java 24), JDBC-драйверы должны быть VT-friendly (PG 42.7+, HikariCP 5.1+). И connection pool БД всё равно потолок для DB-heavy async работ.

**Все proxy-ограничения применимы**: self-invocation, private, final, static, void return type (для fire-and-forget) — как в @Transactional.

**Тонкость с exception handling**. Для методов возвращающих `CompletableFuture` — исключения ловятся через `.exceptionally(ex -> ...)` или `.handle((result, ex) -> ...)`. Для `void` — `AsyncUncaughtExceptionHandler` (глобальный). Забывчивость этого — типовой prod bug: async метод падает, никто не знает.

## @Cacheable: генерация ключей, cache manager, edge cases

```java
@Cacheable("users")
public User findById(Long id) {
    return repo.findOne(id);
}
```

Механика внутри `CacheInterceptor`: (1) достаёт `CacheManager` бин, находит cache по имени `"users"`; (2) вычисляет **key** через `KeyGenerator` (default — `SimpleKeyGenerator`, использует все параметры метода); (3) `cache.get(key)` → если hit, возвращает cached value без вызова метода; (4) если miss, вызывает real method, `cache.put(key, result)`, возвращает.

**Генерация ключей** — самая частая источник ошибок. `SimpleKeyGenerator` работает так: если один параметр — используется он сам (при условии что имеет корректный equals/hashCode); если несколько — создаётся `SimpleKey` который делает equals/hashCode по массиву параметров. Проблемы:

- **Параметры без `equals`/`hashCode`** (например custom класс без переопределения) — cache будет always miss, потому что каждый вызов даёт объект с новым identity hash.
- **Массивы** как параметры — `hashCode()` массива основан на identity, не на содержимом. `findByIds(new long[]{1,2,3})` два раза подряд — два разных ключа.
- **null параметр** — включается в ключ, работает, но может быть неожиданно.

Правильный подход — явно задавать key через SpEL:

```java
@Cacheable(value = "users", key = "#id")
public User findById(Long id) { }

@Cacheable(value = "users", key = "#user.email")
public User findByExample(User user) { }

@Cacheable(value = "orders", key = "#userId + ':' + #status")
public List<Order> findByUserAndStatus(Long userId, Status status) { }
```

**CacheManager** — абстракция над реальным cache (`ConcurrentMapCacheManager` для in-memory, `CaffeineCacheManager` для Caffeine, `RedisCacheManager` для распределённого). Без явного CacheManager `@Cacheable` **не работает** (Spring Boot автоматически создаёт `ConcurrentMapCacheManager` если ничего другого нет). Классическая ошибка — забыть `@EnableCaching` — тогда даже с CacheManager proxy не создаётся, аннотация игнорируется.

**Условное кэширование** через `condition` и `unless`:

```java
@Cacheable(value = "users", condition = "#id > 0", unless = "#result == null")
public User findById(Long id) { }
```

`condition` — SpEL, вычисляется **до** вызова метода; если false, кэш пропускается, метод вызывается напрямую. `unless` — SpEL, вычисляется **после** вызова; если true, результат в кэш не кладётся. Разница ключевая: `condition` может использовать только параметры, `unless` — ещё и `#result`.

**Cache stampede** — классическая проблема кэшей. Cache miss на популярном ключе → N параллельных запросов одновременно → все N делают тяжёлый метод → все N кладут в cache. При популярных ключах и медленных методах — DB под ударом. Решение: `sync = true`:

```java
@Cacheable(value = "users", key = "#id", sync = true)
public User findById(Long id) { }
```

`sync=true` — Spring использует synchronized блок вокруг load; N параллельных вызовов ждут первого, дальше все получают cached значение. Работает только для однопроцессного in-memory cache. Для Redis нужны отдельные механизмы (Redis SETNX + TTL, distributed lock).

**`@CacheEvict`** — proxy удаляет из cache. `allEntries=true` — очищает весь named cache. `beforeInvocation=true` — удаляет до вызова метода (полезно если метод может упасть, но кэш всё равно надо инвалидировать).

**`@CachePut`** — proxy **всегда** вызывает метод и кладёт результат в cache. Отличие от `@Cacheable` — не читает из cache, только обновляет. Полезно для update-методов.

**Комбинация `@Caching`** — несколько cache-операций на одном методе:

```java
@Caching(
    evict = {
        @CacheEvict(value = "usersByEmail", key = "#user.email"),
        @CacheEvict(value = "usersById", key = "#user.id")
    },
    put = @CachePut(value = "usersById", key = "#user.id")
)
public User updateUser(User user) { }
```

Обновляем пользователя — инвалидируем два кэша (по email и по id), плюс кладём свежую версию в usersById.

**Тонкость с TTL**. `@Cacheable` не задаёт TTL — это ответственность CacheManager. Spring Cache abstraction не имеет унифицированного способа задать TTL per-cache — надо через конфигурацию конкретного провайдера:

```java
// Caffeine
@Bean
CacheManager cacheManager() {
    CaffeineCacheManager mgr = new CaffeineCacheManager("users", "orders");
    mgr.setCaffeine(Caffeine.newBuilder()
        .expireAfterWrite(Duration.ofMinutes(10))
        .maximumSize(10_000));
    return mgr;
}

// Redis (Spring Data Redis)
@Bean
CacheManager cacheManager(RedisConnectionFactory f) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10));
    return RedisCacheManager.builder(f).cacheDefaults(config).build();
}
```

## @PreAuthorize: SpEL, security context, тонкости

```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.name")
public User getUserProfile(Long userId) { }
```

Механика `MethodSecurityInterceptor`: (1) достаёт `Authentication` из `SecurityContextHolder` (ThreadLocal-based); (2) evaluates SpEL expression через `MethodSecurityExpressionHandler` в контексте текущего authentication + параметров метода; (3) если результат false → `AccessDeniedException`; (4) иначе → делегирует real method.

**SpEL context** содержит: `authentication` (текущий Authentication), `principal` (principal того же authentication), параметры метода по имени (`#userId`, `#user.email` и т.д.). Плюс кастомные bean expressions через `@authz.hasPermission(#target)` где `authz` — Spring bean с методом `hasPermission`.

**Требование к именам параметров**. Spring по умолчанию использует debug-информацию из bytecode для получения имён параметров. Без `-parameters` флага компилятора (Java 8+) или без явных `@Param` annotations — параметры называются `arg0`, `arg1` и т.д. `@PreAuthorize("#userId == ...")` тогда не работает. Fix: включить `-parameters` в compiler options (Maven `maven-compiler-plugin` `<parameters>true</parameters>`, Gradle `options.compilerArgs += "-parameters"`).

**`@PostAuthorize`** — оценка **после** выполнения метода, доступ к `returnObject`:

```java
@PostAuthorize("returnObject.owner == authentication.name")
public Document getDocument(Long id) { }
```

Метод выполнится, но результат клиенту не отдастся если expression false. Полезно когда нельзя проверить permission без загрузки объекта. Опасность — тяжёлый метод выполняется зря если authorization fail. Обычно `@PreAuthorize` предпочтительнее.

**`@PreFilter` / `@PostFilter`** — фильтрация коллекций входа/выхода:

```java
@PostFilter("filterObject.owner == authentication.name")
public List<Document> listAll() { }
```

Из возвращённого списка Spring оставит только те объекты, у которых `owner == authentication.name`. `filterObject` — специальный variable в SpEL, означает текущий элемент коллекции. Работает через iteration и remove — на больших коллекциях медленно (O(n) с SpEL evaluation каждого элемента). Лучше фильтровать в SQL.

**MethodSecurityInterceptor chain**. Внутри Spring Security цепочка advisors: `AuthorizationManagerBeforeMethodInterceptor` (для `@PreAuthorize`), `AuthorizationManagerAfterMethodInterceptor` (для `@PostAuthorize`), `PreFilterAuthorizationMethodInterceptor`, `PostFilterAuthorizationMethodInterceptor`. Каждый advisor работает через отдельный `AuthorizationManager` — можно кастомизировать логику решения.

**Кастомный `PermissionEvaluator`** для сложных сценариев:

```java
@Component
public class DocumentPermissionEvaluator implements PermissionEvaluator {
    public boolean hasPermission(Authentication auth, Object target, Object perm) {
        Document d = (Document) target;
        return d.getOwnerId().equals(currentUserId(auth)) 
            || auth.getAuthorities().contains(new SimpleGrantedAuthority("ADMIN"));
    }
    // ...
}

@PreAuthorize("hasPermission(#doc, 'read')")
public void view(Document doc) { }
```

Централизует permission logic в одном месте, аннотации остаются семантическими (`'read'`, `'write'`, `'delete'`).

## @Retryable: механика, backoff, recovery

```java
@Retryable(
    value = TransientException.class, 
    maxAttempts = 3, 
    backoff = @Backoff(delay = 1000, multiplier = 2)
)
public String callFlakyAPI() { }
```

Механика `RetryOperationsInterceptor` через `RetryTemplate`: (1) выполняет метод; (2) при exception проверяет `RetryPolicy` (тип исключения соответствует, попыток ещё осталось); (3) если retry — `BackoffPolicy.backOff()` (sleep согласно стратегии), затем повтор; (4) если нет retry (все попытки исчерпаны, exception не тот тип) — пробрасывает исключение или вызывает `@Recover` метод.

**Стратегии backoff**:

- **FixedBackoffPolicy** — постоянная задержка между попытками (delay=1000 → 1s, 1s, 1s).
- **ExponentialBackoffPolicy** — экспоненциальный рост (delay=1000, multiplier=2 → 1s, 2s, 4s, 8s).
- **UniformRandomBackoffPolicy** — случайная задержка между `minBackoff` и `maxBackoff`.
- **ExponentialRandomBackoffPolicy** (jittered) — экспоненциальный + рандомизация. Обычно правильный выбор для распределённых систем: избегает thundering herd когда много клиентов ретраятся одновременно (см. файл 106).

**`@Recover`** — метод-fallback когда retry исчерпан:

```java
@Retryable(value = TransientException.class, maxAttempts = 3)
public String callAPI(Long id) { }

@Recover
public String recover(TransientException e, Long id) {
    log.warn("Retry exhausted for id={}, using default", id);
    return "default-value";
}
```

Требования к `@Recover` методу: тот же тип возвращаемого значения; первый параметр — тип исключения; остальные параметры совпадают с retry-методом. Spring находит recover по match'у.

**RetryListener** для observability и custom logic:

```java
@Bean
RetryListener retryListener() {
    return new RetryListener() {
        public <T, E extends Throwable> void onError(
                RetryContext ctx, RetryCallback<T,E> cb, Throwable ex) {
            log.warn("Retry attempt {} failed: {}", ctx.getRetryCount(), ex.getMessage());
        }
    };
}
```

Полезно для метрик (сколько retries по каким методам), circuit breaker integration.

**Anti-patterns**. Ретрай на **всех** exception (`value = Exception.class`) — плохо, включает бизнес-ошибки которые не должны ретраится (invalid input, authorization denied). Ретрай на **non-idempotent** операции без deduplication — двойная оплата, двойной email. Слишком много попыток без backoff — thundering herd на downstream. Синхронный retry с большим delay блокирует calling thread — если это HTTP request handler, клиент ждёт всё время + может отвалиться по своему timeout, а ретрай продолжится впустую.

**Circuit breaker дополняет retry** (Resilience4j `@Retry` + `@CircuitBreaker`). Ретрай для transient failures, circuit breaker для сустаinеd failures — если downstream реально лежит, retry не поможет, надо на время перестать пытаться.

## Порядок interceptor'ов: точная механика

Когда на методе несколько аспект-аннотаций — Spring собирает **цепочку advisors**, каждый обрабатывается по очереди. Внутренне это `ReflectiveMethodInvocation` со списком interceptor'ов, каждый вызывает `invocation.proceed()` чтобы передать управление следующему.

Порядок определяется через `@Order` или интерфейс `Ordered` / `PriorityOrdered` на самом advisor'е. Меньший order = **внешний** (раньше в цепочке). Больший order = **внутренний**.

Default приоритеты — многие Spring-аннотации по умолчанию `LOWEST_PRECEDENCE` (Integer.MAX_VALUE). Одинаковый приоритет → недетерминированный порядок → баги воспроизводимые «через раз».

Правильно — явно задавать через `@EnableXxx(order = N)`:

```java
@EnableCaching(order = 1)                  // самый внешний
@EnableTransactionManagement(order = 2)     // внутри cache
@EnableAsync(order = 3)                     // самый внутренний
```

**Правильный порядок обычно cache → tx → метод**. Обоснование: при cache hit не должна открываться транзакция (пустая работа с БД), проверка кэша — cheap операция, должна быть первой.

Если поставить наоборот (tx снаружи cache): cache hit → открыта пустая транзакция → cache вернул значение → tx commit'ится «зря». На каждый cache hit — расход connection из пула, WAL запись (для tx с ACID guarantees), overhead. Cache теряет смысл — вместо экономии БД добавляется overhead. Реальный prod bug: команда добавила @Cacheable на существующий @Transactional метод, throughput упал вместо роста. Виной был порядок.

**@Async особый случай**. `@Async` возвращает CompletableFuture сразу, поэтому если оно **внутри** @Transactional — транзакция закрывается **до** реального выполнения метода в другом треде. Транзакция не действует внутри async метода. Обычно async **снаружи** tx — сам async метод открывает свою транзакцию.

Диагностика: включить debug логирование `org.springframework.aop.framework.CglibAopProxy` — покажет реальную цепочку interceptor'ов для каждого метода.

## Proxy vs Decorator: глубокая разница

Оба паттерна оборачивают объект тем же интерфейсом. Разница — в **отношении к жизненному циклу** и **осведомлённости клиента**.

**Управление жизненным циклом**. В Proxy — proxy owns реальный объект. Клиент получает только proxy, real создаётся либо lazily внутри proxy (Virtual Proxy), либо инъектится в proxy извне (Smart Proxy), либо не существует локально вовсе (Remote Proxy). Клиент никогда не видит real напрямую. Пример: Hibernate lazy proxy — вы получили User, но реального ResultSet-mapped объекта ещё нет, он материализуется при первом обращении к полю.

В Decorator — клиент owns и wrapped, и decorator. Клиент явно пишет `new BufferedInputStream(new FileInputStream("f.txt"))`. Обе стороны видны, часть цепочки. Клиент может расформировать decorator и работать с оригиналом.

**Композиция**. Proxy обычно один — один proxy на один real. Даже когда Spring применяет много аспектов, это одна цепочка interceptors внутри одного CGLIB proxy, не N вложенных proxy. Композиция происходит через AOP infrastructure, не через client-side wrapping.

Decorator создан для композиции. `new EncryptedStream(new BufferedStream(new GzipStream(new FileStream("f"))))` — четыре уровня, каждый добавляет своё поведение. Порядок важен, каждый комбинирует с предыдущим.

**Прозрачность**. Клиент Proxy думает что говорит с оригиналом. Аннотация `@Transactional` должна работать «магически» — код `userService.update(...)` выглядит обычным вызовом. Прозрачность — цель.

Клиент Decorator знает про composition. Он явно собирает — если забудет `BufferedInputStream`, чтение будет медленным (каждый byte напрямую с диска). Осведомлённость — тоже цель, потому что клиент выбирает поведение.

**Реальный tricky пример: Hibernate**. Lazy loading — proxy: клиент вызывает `user.getOrders()`, proxy незаметно делает SELECT. Клиент не знает про SQL. Идеальный proxy.

А `Hibernate.initialize(user.getOrders())` — принудительная инициализация proxy — это уже клиент явно взаимодействует с proxy механикой. В строгом смысле не paradigm-clean.

**Ошибка понимания**. Многие думают что `@Transactional` работает как decorator (клиент оборачивает). На самом деле клиент не оборачивает — Spring делает это автоматически при создании bean. Клиент **не знает** что перед его userService стоит proxy. Это принципиально отличает Proxy от Decorator.

Таблица сводит различия:

| | Proxy | Decorator |
|-|-------|-----------|
| Намерение | Control access | Add behavior |
| Клиент знает | Нет (proxy invisible) | Да (клиент wraps) |
| Real object owner | Proxy (или его создатель) | Клиент |
| Composition | Обычно один | Часто цепочка |
| Классические use cases | Cache, Lazy, Remote, Security | I/O streams, GUI toolkit |
| Spring examples | @Transactional, @Async, Hibernate LAZY | Java IO, GUI wrapping |
| Cost для клиента | Нулевой (transparent) | Знание композиции |

Правило: Proxy отвечает на «кто может» (управляет доступом). Decorator отвечает на «что дополнительно» (расширяет поведение).

## Performance implications: цена proxy

Proxy не бесплатен. Overhead в трёх местах:

**Method dispatch через interceptor chain**. Обычный вызов метода в Java — ~1-2 наносекунды (плюс возможный JIT inlining в ноль). Через Spring proxy — 50-200 ns на пустую цепочку (один interceptor), больше при нескольких аспектах. Значимо для tight loops с миллионами вызовов, незаметно для типичных enterprise методов с БД в 5-50 ms.

**Reflection cost** для JDK Proxy. `method.invoke()` внутри `InvocationHandler` использует reflection — не так дёшево как direct call. С Java 9+ MethodHandle-based dispatch значительно ускорил, но всё ещё дороже CGLIB (который использует прямые вызовы через сгенерированный bytecode). Реально: JDK Proxy ~500 ns на вызов, CGLIB ~100 ns.

**Megamorphic call sites**. JIT-оптимизация inlining лучше работает на **monomorphic** call sites — когда через одну точку вызова всегда проходит один тип. Proxy — новый класс на каждый bean (`$Proxy0`, `$Proxy1`, `EnhancerByCGLIB$abc`, `EnhancerByCGLIB$def`), все имплементят одинаковый интерфейс. Если клиент работает с interface через много разных proxy — call site становится megamorphic, JIT не может inline, dispatch дорогой (см. файл 112 про JIT).

Практическое влияние: обычно ничтожно. Enterprise workload — DB, network, disk доминируют. Proxy overhead в 100-500 ns на фоне 10 ms SQL — 0.005%. Тюнить имеет смысл только для действительно hot paths (миллион+ вызовов в секунду).

Способы оптимизации если проблема реальна: (1) кэшировать результаты дорогих proxy-вызовов; (2) переходить на AspectJ (weaving в bytecode, дешевле dispatch); (3) убрать proxy где не нужен (иногда `@Transactional` на trivial методе излишен).

## Тестирование proxy-based beans

Тесты обычно не заботятся о proxy — вызывают методы обычным способом. Но есть моменты которые надо понимать.

**Unit tests (без Spring context)**. Обычно `new UserService(mockedRepo, ...)` — тестируется real class. Аннотации `@Transactional`/`@Cacheable` **не работают** (нет proxy). Это правильно для unit test — мы тестируем логику, не Spring infrastructure. Если нужна логика вокруг транзакции — интеграционный тест.

**`@SpringBootTest` / `@DataJpaTest`**. Полный Spring context поднят, все аспекты работают. Здесь proxy реален. Можно тестировать что `@Transactional` действительно rollback'ит при exception.

**`@MockBean` vs `@SpyBean`**. `@MockBean` заменяет bean на Mockito mock (без реального класса). Все аспекты пропадают — mock не проходит через proxy. `@SpyBean` — оборачивает **real bean** (уже proxy'ннный) в spy, сохраняя всю Spring machinery. Полезно когда нужно частично мокнуть поведение, но сохранить `@Transactional`.

**Верификация `@Transactional` в тестах**:

```java
@Test
void update_onError_rollsBack() {
    // given
    User u = repo.save(new User("test"));
    
    // when
    assertThrows(RuntimeException.class, () -> service.updateWithFailure(u.getId()));
    
    // then — rollback произошёл, изменения не сохранились
    User reloaded = repo.findById(u.getId()).orElseThrow();
    assertEquals("test", reloaded.getName());   // не изменилось
}
```

Работает потому что `service` в тесте — proxy (Spring context поднят), `@Transactional` работает.

**AopUtils для программной верификации в тесте**:

```java
@Test
void userService_isProxy() {
    assertTrue(AopUtils.isAopProxy(userService));
    assertTrue(AopUtils.isCglibProxy(userService));
    assertEquals(UserService.class, AopProxyUtils.ultimateTargetClass(userService));
}
```

Полезно как sanity check в критичных проектах — тест провалится если кто-то случайно сломает proxy (сделал класс final).

## Практические сценарии: где что использовать

**Facade** — когда клиенту нужно N связанных операций подсистемы. Классический пример: обработка заказа с несколькими сервисами. Также — обёртка над третьесторонним API (легче тестировать через свою абстракцию), разделение orchestration от business logic, agregation в BFF/API Gateway. Правило: начинай без фасада, добавляй когда чувствуешь боль от дублирования orchestration.

**Proxy** — для прозрачного добавления cross-cutting поведения (log, cache, tx, auth) через Spring AOP. Для lazy loading тяжёлых объектов (Hibernate). Для remote calls (Feign, gRPC). Для thread-safe wrapping. Для reference counting (редко в Java, чаще в C++).

**Не Proxy** когда: клиент должен явно управлять композицией — Decorator; когда меняется форма интерфейса — Adapter; когда нужна координация peers — Mediator.

**Комбинация в реальных системах**. Часто и вместе:

```
Client
   ↓
[CGLIB Proxy for @Transactional]         ← proxy adds tx
   ↓
[Real OrderFacade]                        ← facade orchestrates
   ↓ ↓ ↓
[Proxy for @Cacheable] → [Real UserService]        
[Proxy for @Transactional] → [Real PaymentGateway]
[Proxy for @Retryable + @CircuitBreaker] → [Real ExternalApiClient]
```

Facade — верхний уровень (business orchestration). Proxy — по всей системе для cross-cutting concerns (tx, cache, retry, security, метрики).

## Отладка в проде: реальные сценарии

**Симптом «@Transactional не работает — транзакция не открывается»**. Пошаговая диагностика:

1. `bean.getClass().getName()` — есть ли `EnhancerBySpringCGLIB` или `$Proxy`? Если нет — proxy не создан. Причины: класс `final`; `@Transactional` только на реализации без интерфейса + `spring.aop.proxy-target-class=false`; bean создаётся через `new` вместо DI; `@EnableTransactionManagement` не включён; `PlatformTransactionManager` бин не создан.
2. Proxy есть, но не работает? — проверить не self-invocation ли. Логи по строке `this.transactionalMethod()` — прямой вызов через `this`, минует proxy. Fix — один из четырёх способов (self-inject, AopContext, разделение на два бина, AspectJ).
3. Метод не public? — `@Transactional` игнорируется без ошибок. Проверить модификатор.
4. Breakpoint в `TransactionInterceptor.invokeWithinTransaction()` — при вызове метода проходит через? Если нет — proxy не в пути. Если да — транзакция открыта, ищи проблему дальше (rollback rules — по умолчанию только `RuntimeException` и `Error`; checked exception не роллбэчит).

**Симптом «@Cacheable не кэширует, каждый вызов идёт в БД»**. Кроме proxy-проверок:

1. `CacheManager` bean создан? — при отсутствии `@Cacheable` молча ничего не делает (не бросает ошибку). Проверить `@EnableCaching`.
2. Cache с указанным именем существует? — `spring.cache.cache-names` или явное определение в CacheManager.
3. Ключ правильно генерируется? — включить debug для `org.springframework.cache.interceptor.CacheAspectSupport`, увидеть какие ключи вычисляются.
4. Параметр без нормального `equals/hashCode`? — каждый вызов даёт новый ключ, cache always miss.

**Симптом «@Async работает синхронно, метод блокирует calling thread»**. Стандартные проверки плюс:

1. `@EnableAsync` включён?
2. Self-invocation? — вызов async метода изнутри того же класса выполняется синхронно.
3. Return type правильный? Void или Future/CompletableFuture. Любой другой тип — синхронное выполнение.
4. `TaskExecutor` настроен? Если используется дефолтный `SimpleAsyncTaskExecutor` — работает асинхронно, но создаёт новый тред на каждый вызов (opasно в prod).

**Симптом `LazyInitializationException` в контроллере**. Hibernate lazy proxy пытается сделать query, но session закрыт (за пределами `@Transactional` метода в service layer). Классика.

Решения по возрастанию правильности: (1) `FETCH JOIN` в query — тяжёлая ассоциация грузится сразу, никаких lazy proxy'ев не нужно; (2) DTO projection — не возвращать entity в контроллер, конвертировать в DTO внутри @Transactional; (3) `@Transactional(readOnly=true)` на всём service методе, возвращающем данные; (4) `open-in-view=true` (OSIV) — session остаётся открытой всё время обработки request'а, lazy loading работает в контроллере (**anti-pattern** — сложно контролировать N+1 queries, отладка становится непонятной; но иногда используется как быстрый fix legacy).

**Custom AOP с `@Aspect`** для timing-логирования:

```java
@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service..*(..))")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            if (duration > 100) {
                log.warn("Slow method {}: {} ms", pjp.getSignature(), duration);
            }
        }
    }
}
```

Spring создаёт proxy для beans в `com.example.service`. Каждый вызов проходит через `logAround`, timing для медленных методов пишется в лог.

Pointcut expressions селектят где применить: `execution(...)` — по сигнатуре метода; `within(...)` — по классу/пакету; `@annotation(...)` — по наличию аннотации; `args(...)` — по типам параметров. Advice: `@Before` (до), `@After` (после, finally), `@AfterReturning` (только при success), `@AfterThrowing` (только при exception), `@Around` (полный контроль, вызывает `pjp.proceed()`).

## Заключение

Facade и Proxy — два фундаментальных структурных паттерна, на которых практически стоит весь enterprise Spring. Facade упрощает доступ к сложной подсистеме (JdbcTemplate над JDBC, OrderFacade над сервисами, BFF/Gateway над микросервисами). Клиент знает что за фасадом сложность. Proxy контролирует доступ к объекту с тем же интерфейсом (@Transactional, @Async, @Cacheable, @PreAuthorize через AOP). Клиент не знает что говорит с proxy.

**Виды Proxy**: Virtual (lazy — Hibernate LAZY loading), Protection (auth — Spring Security @PreAuthorize), Remote (Feign, gRPC stubs), Smart (cache — @Cacheable, tx — @Transactional).

**Реализации Dynamic Proxy**: JDK Dynamic Proxy (только для интерфейсов, встроен в JDK, генерирует $Proxy0 через reflection), CGLIB (bytecode manipulation через subclass, работает без интерфейсов, ограничения final/private/static), ByteBuddy (современная альтернатива, активно развивается).

**Spring AOP** — proxy-based. Bean с интерфейсом → JDK Proxy или CGLIB (в Boot 2.0+ CGLIB default). Bean без интерфейса → CGLIB. Все аспект-аннотации (@Transactional, @Async, @Cacheable, @PreAuthorize, @Retryable) реализованы через interceptor'ы в proxy chain.

**Self-invocation problem** — самая частая ловушка. `this.method()` минует proxy. Fix: self-injection, AopContext, разделение на два бина, AspectJ (bytecode weaving).

**Ограничения Spring Proxy**: private / final / static методы не proxy'ятся. Silent bug — код работает, аннотация игнорируется. Правило: аспект-аннотации только на public non-final методах. Для Kotlin — `all-open` plugin.

**Порядок interceptor'ов** важен когда несколько аннотаций. Cache обычно снаружи Transaction (cache hit не должен открывать tx). Указывать явно через `@Order` или `@EnableXxx(order=N)`.

**Proxy vs Decorator**: proxy контролирует доступ (invisible для клиента, proxy владеет real object), decorator добавляет функциональность (клиент явно оборачивает, часто в цепочку).

**Отладка**: `bean.getClass().getName()` для проверки наличия proxy. Breakpoint в TransactionInterceptor для проверки прохождения. Silent bugs (self-invocation, final method) — самые коварные.

Концепция абстракции как таковой — в 113. `@Transactional` детали — в 33. Spring аннотации подробно — в 60. Здесь была глубина конкретно по Facade и Proxy: механика, реализации, Spring применение, типовые ошибки и способы их обходить.
