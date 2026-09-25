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

## Отладка: как понять что там proxy

Быстрая проверка через `bean.getClass().getName()`:

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

Если видишь имя оригинального класса без EnhancerBy... суффикса — proxy не создан. Аннотация не сработает. Проверять почему (final class? final method? не bean вообще?).

В IDE можно поставить breakpoint в `TransactionInterceptor.invoke()` и увидеть full stacktrace — прошёл ли вызов через interceptor. Если stacktrace не проходит через interceptor — self-invocation или другая ловушка.

## AOP alternatives: AspectJ

Spring AOP — proxy-based. Все ограничения выше. **AspectJ** — bytecode weaving, две разновидности:

- **Compile-time weaving (CTW)** — javac + iajc модифицируют .class файлы. Медленный build, но zero runtime cost.
- **Load-time weaving (LTW)** — Java agent модифицирует bytecode при classloading. Runtime cost, но не нужен специальный build.

AspectJ мощнее (перехватывает всё, включая self-invocation, private, final), но сложнее настроить. Для типичного Spring-приложения Spring AOP достаточно. AspectJ имеет смысл когда нужна перехват private/self-invocation или когда работаешь с legacy-кодом который сложно рефакторить.

## Другие Spring-аннотации: те же принципы

Все аспект-аннотации Spring работают через proxy тем же образом.

**`@Async`**:

```java
@Async
public CompletableFuture<Report> generateReport(Long id) {
    // slow computation
    return CompletableFuture.completedFuture(report);
}
```

Proxy при вызове не выполняет метод immediately — оборачивает в Runnable/Callable, submit в `TaskExecutor` (обычно `ThreadPoolTaskExecutor` или virtual thread executor), возвращает CompletableFuture. Реальный метод выполняется в другом треде.

Ограничения те же: self-invocation, private, final, static — не работают.

Настройка virtual threads (Java 21+):

```java
@Bean
AsyncTaskExecutor asyncTaskExecutor() {
    return new TaskExecutorAdapter(
        Executors.newVirtualThreadPerTaskExecutor()
    );
}
```

**`@Cacheable`**:

```java
@Cacheable("users")
public User findById(Long id) {
    return repo.findOne(id);
}
```

Proxy компилирует key из method args (default — все args), проверяет cache: hit → возвращает cached value; miss → вызывает real method, кладёт в cache, возвращает. `@CacheEvict` — proxy удаляет из cache перед/после вызова.

**`@PreAuthorize`** (Spring Security):

```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.name")
public User getUserProfile(Long userId) { }
```

Proxy достаёт `Authentication` из `SecurityContext`, evaluates SpEL expression, если false — `AccessDeniedException`, иначе — real method.

**`@Retryable`** (Spring Retry):

```java
@Retryable(value = TransientException.class, 
           maxAttempts = 3, 
           backoff = @Backoff(delay = 1000, multiplier = 2))
public String callFlakyAPI() { }
```

Proxy: try 1 → exception → sleep 1000ms → try 2 → exception → sleep 2000ms → try 3 → success или `ExhaustedRetryException`.

Каждая аннотация — свой interceptor в цепочке proxy.

## Порядок interceptor'ов: тонкость которая важна

Если на методе несколько аннотаций — `@Transactional` + `@Cacheable` + `@Async` — Spring выстраивает цепочку interceptor'ов по priority. Default порядок:

- `@Async` — `LOWEST_PRECEDENCE`.
- `@Transactional` — `LOWEST_PRECEDENCE`.
- `@Cacheable` — `LOWEST_PRECEDENCE`.

Все одинаковые → недетерминированный порядок → проблема.

Управлять через `@Order` или `@EnableTransactionManagement(order=100)` + `@EnableCaching(order=200)` — явно указать.

Классическая ошибка: `@Transactional` + `@Cacheable` на одном методе.
- Если `@Cacheable` снаружи `@Transactional` → cache hit не открывает транзакцию (правильно).
- Если `@Transactional` снаружи `@Cacheable` → лишняя транзакция для cache hit (неправильно).

Обычно **Cache снаружи Transaction** — правильно. Проверять порядок явно.

## Proxy vs Decorator: принципиальная разница

Часто на собеседовании. Оба паттерна оборачивают объект, реализуют тот же интерфейс, делегируют вызовы wrapped object. В чём разница?

**Намерение**. Proxy — контроль доступа к объекту (кэш, security, lazy, remote). Клиент не знает что говорит с proxy — думает что с real. Decorator — добавление функциональности к объекту динамически. Клиент знает что декорировал (сам оборачивает).

**Управление жизненным циклом**. Proxy сам создаёт / управляет real object (или получает его из фабрики). Клиент не знает про real. Decorator — клиент **явно** создаёт wrapped и decorator, собирает цепочку.

**Композиция**. Decorator часто цепочка: `new EncryptedStream(new BufferedStream(new FileStream("f")))`. Каждый добавляет своё поведение. Proxy обычно один, не собирается в цепочку от клиента.

**Пример Decorator в JDK**:

```java
InputStream in = new FileInputStream("f.txt");
InputStream buffered = new BufferedInputStream(in);
InputStream zipped = new GZIPInputStream(buffered);
```

Клиент собирает цепочку: FileInputStream (real — read from disk), BufferedInputStream (add buffering), GZIPInputStream (add decompression). Каждый уровень сохраняет интерфейс InputStream, добавляет поведение.

Резюме через таблицу:

| | Proxy | Decorator |
|-|-------|-----------|
| Намерение | Control access | Add behavior |
| Клиент знает | Нет (proxy invisible) | Да (клиент wraps) |
| Real object owner | Proxy | Клиент |
| Composition | Обычно один | Часто цепочка |
| Классические use cases | Cache, Lazy, Remote, Security | I/O streams, GUI toolkit |
| Spring examples | @Transactional, @Async | Java IO, Redis decorators |

Простой mnemonic: Proxy решает «кто может». Decorator решает «что дополнительно делать».

## Практика: где что использовать

**Facade** когда: клиенту нужно N связанных операций (объединить в один «typical use case» метод); скрыть 3rd-party API за собственным domain-specific API (легче тестировать); разделить orchestration от business logic; уменьшить coupling между слоями.

**Proxy** когда: нужно прозрачно добавить cross-cutting поведение (log, cache, tx, auth) — Spring AOP; real object дорого создавать → lazy load (Hibernate, JPA); real object на другой машине → remote proxy (Feign, gRPC); нужен counted / synchronized доступ.

**Не Proxy** когда: клиент сам должен решать композицию — используй Decorator; меняешь интерфейс — используй Adapter.

**Комбинация**. В реальных системах — вместе:

```
Client
   ↓
[Proxy for @Transactional]         ← proxy adds tx
   ↓
[Real OrderFacade]                  ← facade orchestrates
   ↓
[Proxy for @Cacheable] → [Real UserService]     ← proxy adds cache
[Proxy for @Transactional] → [Real PaymentGateway]
[Proxy for @Retryable] → [Real ExternalApiClient]
```

Facade — верхний уровень (orchestration). Proxy — по всей системе для cross-cutting concerns.

## Отладка в проде: типовые сценарии

**«@Transactional не работает — транзакция не открывается»**. Проверить: `bean.getClass().getName()` — есть ли `EnhancerBySpringCGLIB` в имени. Если нет — proxy не создан, разобраться почему (final class? bean vs plain new? scope issue?). Если есть — проверить не self-invocation ли (вызов `this.transactionalMethod()` из другого метода того же класса). Breakpoint в `TransactionInterceptor.invoke()` — реально ли проходит.

**«@Cacheable не кэширует, каждый вызов идёт в БД»**. Те же проверки. Плюс: правильно ли настроен `CacheManager` (без CacheManager @Cacheable молчит). Правильно ли собирается ключ (default = все args, если args non-hashable — cache не работает).

**«@Async работает синхронно»**. Skip proxy (self-invocation) — самая частая причина. Проверить `@EnableAsync` включён. Проверить конфигурация `TaskExecutor`.

**`LazyInitializationException` в контроллере**. Hibernate lazy proxy пытается сделать query, но session уже закрыт (за пределами `@Transactional` метода). Fix: либо fetch eagerly в query, либо перенести обработку внутрь transactional метода, либо использовать `@Transactional(readOnly=true)` на контроллере (не рекомендуется), либо OSIV (`open-in-view=true` — тоже anti-pattern, но иногда используется).

**Custom AOP с `@Aspect`** — пример полностью работающего:

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
            log.info("Method {} took {} ms", pjp.getSignature(), duration);
        }
    }
}
```

Spring создаёт proxy для beans в `com.example.service`. При каждом method call `LoggingAspect.logAround` вызывается, timing логируется.

Pointcut expressions (execution, within, args, @annotation) — селектят где применить.

Виды advice: `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`.

## Проверочные вопросы для собеседования

Быстрые ответы на типовые вопросы:

**Facade** — упрощение доступа к сложной подсистеме. Клиент знает что за фасадом много всего.

**Facade vs Adapter**: адаптер меняет форму интерфейса (1-to-1), фасад объединяет N сервисов за одним упрощённым API (N-to-1).

**Facade vs Mediator**: фасад однонаправленный (клиент выше подсистемы), медиатор — peer-to-peer координация равных.

**God Facade** — anti-pattern (500+ строк, все домены в одном классе). Fix — разделить по доменам.

**Spring templates** (JdbcTemplate, RestTemplate, RabbitTemplate) — все Facade + Template Method.

**Proxy** — представитель, тот же интерфейс что real. Клиент не знает что говорит с proxy.

**Виды proxy**: virtual (lazy init, Hibernate lazy loading), protection (auth, @PreAuthorize), remote (Feign, gRPC), smart (cache, log — @Cacheable).

**Static Proxy** — писать руками. **Dynamic Proxy** — генерация в runtime.

**JDK Dynamic Proxy** — только для interface'ов, через InvocationHandler, встроен в JDK.

**CGLIB** — bytecode generation, subclass, работает без interface, не может final/private/static.

**Spring Boot 2.0+** — CGLIB by default.

**`@Transactional` через proxy** — TransactionInterceptor открывает транзакцию до, commit/rollback после.

**Self-invocation problem** — this.method() минует proxy. Fix: self-inject / AopContext / разделить bean / AspectJ.

**`@Transactional` не работает на private/final/static** — proxy не может их перехватить.

**Проверка proxy** — `bean.getClass().getName()` покажет EnhancerBySpringCGLIB или $Proxy или plain class.

**Proxy vs Decorator**: proxy invisible (клиент не знает), decorator composed by client (клиент явно оборачивает).

**AOP interceptor chain порядок** — cache снаружи transaction обычно правильно.

**AspectJ** — bytecode weaving, работает с self-invocation, private, final. Сложнее настроить.

**ByteBuddy** — современный CGLIB replacement, используется в Mockito, Hibernate, потенциально в Spring будущего.

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
