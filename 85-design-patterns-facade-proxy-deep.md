# 85. Design Patterns: Facade и Proxy — глубоко

Файл про **два фундаментальных GoF-паттерна** и их реальное применение в Java/Spring: **Facade** (упрощение интерфейса к сложной подсистеме) и **Proxy** (представитель другого объекта). Углублённо: JDK Dynamic Proxy vs CGLIB vs ByteBuddy internals, self-invocation problem, реальные Spring-механизмы (@Transactional, @Async, @Cacheable, @PreAuthorize) все построены на Proxy.

Связано с: `05-spring-framework-ioc-di.md`, `33-transactional-internals.md` (@Transactional через proxy), `34-transactional-advanced.md`, `60-spring-annotations-detailed.md` (@Async, @Cacheable), `67-gateway-detailed.md` (Gateway как proxy).

---

## 0. Ментальная модель

**Оба паттерна — про введение "прослойки"** между клиентом и настоящим объектом. Разница — **зачем**:

```
Facade (упрощение):                    Proxy (управление доступом):
                                       
      Client                                 Client
        │                                      │
        ▼                                      ▼
   ┌────────┐  ← простой интерфейс        ┌────────┐  ← ТОТ ЖЕ интерфейс
   │ Facade │     скрывает сложность      │ Proxy  │     что и Real
   └────┬───┘                             └────┬───┘
        │                                      │
   ┌────┼────┬────┐                            │  ← дополнительное поведение
   ▼    ▼    ▼    ▼                            ▼     (auth, cache, log, tx)
   Sub  Sub  Sub  Sub                       ┌────────┐
   1    2    3    4                         │  Real  │
                                            └────────┘
```

- **Facade** — новый **упрощённый** интерфейс, скрывающий N подсистем.
- **Proxy** — **тот же** интерфейс, что и у real object, но с добавлением логики (cache, security, lazy load, remote).

Ключевое различие: клиент Facade **знает** что за фасадом много всего. Клиент Proxy — **не знает** что говорит с proxy, думает что напрямую с real object.

---

## 1. Facade — детально

### 1.1 Мотивация

Есть сложная подсистема: 5-10 классов, каждый с 10-30 методами. Клиент, чтобы сделать типовую операцию, должен:
1. Знать какие классы вызывать.
2. В правильном порядке.
3. С правильными параметрами.
4. Правильно обрабатывать промежуточные состояния.
5. Правильно откатывать при ошибке.

Каждый клиент повторяет эту логику. Изменение подсистемы = изменение всех клиентов.

**Facade** — один класс с методом `doTypicalOperation(...)`, скрывающий всю сложность.

### 1.2 Формальное определение (GoF)

*Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.*

### 1.3 Классический пример — order processing

**Без фасада**:
```java
public class OrderController {
    private InventoryService inventory;
    private PaymentGateway payment;
    private ShippingService shipping;
    private NotificationService notification;
    private AuditLog audit;

    public OrderResult placeOrder(OrderRequest req) {
        // 1. Проверить наличие
        if (!inventory.hasStock(req.getItems())) {
            return OrderResult.error("Out of stock");
        }
        
        // 2. Зарезервировать
        ReservationId reservation = inventory.reserve(req.getItems());
        
        try {
            // 3. Списать деньги
            PaymentResult payResult = payment.charge(
                req.getUserId(), req.getTotalAmount());
            if (!payResult.isSuccess()) {
                inventory.releaseReservation(reservation);
                return OrderResult.error("Payment failed: " + payResult.getReason());
            }
            
            // 4. Создать shipment
            ShipmentId shipment;
            try {
                shipment = shipping.createShipment(req);
            } catch (ShippingException e) {
                payment.refund(payResult.getTransactionId());
                inventory.releaseReservation(reservation);
                throw e;
            }
            
            // 5. Notify
            notification.sendOrderConfirmation(req.getUserId(), shipment);
            
            // 6. Audit
            audit.log("ORDER_PLACED", req.getUserId(), reservation, shipment);
            
            return OrderResult.success(shipment);
        } catch (Exception e) {
            audit.log("ORDER_FAILED", req.getUserId(), e.getMessage());
            throw e;
        }
    }
}
```

100 строк в контроллере. Знает про 5 сервисов, все compensation flows, все ошибки. Копия в каждом месте где нужно `placeOrder`.

**С фасадом**:
```java
@Service
public class OrderFacade {
    private final InventoryService inventory;
    private final PaymentGateway payment;
    private final ShippingService shipping;
    private final NotificationService notification;
    private final AuditLog audit;

    public OrderResult placeOrder(OrderRequest req) {
        // Вся сложная логика тут (одно место).
        // ...
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

Контроллер тонкий. Facade инкапсулирует всё.

### 1.4 UML

```
    ┌──────────┐
    │  Client  │
    └────┬─────┘
         │
         ▼
    ┌──────────┐            ┌─── Subsystem ────┐
    │  Facade  │────────►   │  ┌────────┐      │
    │          │            │  │Inventory│      │
    │+doA()    │            │  └────────┘      │
    │+doB()    │            │  ┌────────┐      │
    │+doC()    │──►────►    │  │Payment │      │
    └──────────┘            │  └────────┘      │
                            │  ┌────────┐      │
                            │  │Shipping│      │
                            │  └────────┘      │
                            └──────────────────┘
```

Facade **не является членом подсистемы** — это отдельный класс, знающий про неё. Клиент **может** пользоваться и напрямую подсистемой, если нужен fine-grained control.

### 1.5 Facade в JDK и Spring

Примеры где Facade **уже** применён:

**`java.net.URL`** — фасад над `URLConnection`, `HttpClient`, `Sockets`, `DNS resolver`. `new URL("...").openStream()` — одна строка, за ней десяток классов.

**`JOptionPane.showMessageDialog()`** — фасад над `JDialog`, `JLabel`, `JButton`, `Frame`, layout managers.

**`javax.faces.context.FacesContext`** — фасад над JSF processing infrastructure.

**Spring `JdbcTemplate`** — фасад над JDBC (Connection, Statement, ResultSet, ошибки, cleanup):
```java
// Без фасада (raw JDBC):
try (Connection conn = ds.getConnection();
     PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id=?")) {
    ps.setLong(1, id);
    try (ResultSet rs = ps.executeQuery()) {
        if (rs.next()) {
            return mapUser(rs);
        }
        return null;
    }
} catch (SQLException e) {
    throw new DataAccessException(e);
}

// С JdbcTemplate:
User u = jdbc.queryForObject("SELECT * FROM users WHERE id=?", 
    userRowMapper, id);
```

**Spring `RestTemplate` / `WebClient`** — фасад над HTTP-клиентом.

**Spring `RabbitTemplate` / `KafkaTemplate`** — фасад над messaging.

**Spring `TransactionTemplate`** — фасад над PlatformTransactionManager.

Все шаблоны Spring "*Template" — это по сути **Facade + Template Method**.

### 1.6 Facade в микросервисах — BFF и API Gateway

Идея масштабируется на уровень системы:

**Backend for Frontend (BFF)** — микросервис-facade перед N доменными сервисами. Каждому UI (mobile app, web, admin panel) — свой BFF, aggregating нужные ему данные.

```
                     ┌─── mobile BFF ───┐
Mobile app ────────► │  aggregates:      │────► User Service
                     │  - User info      │────► Order Service  
                     │  - Recent orders  │────► Notification
                     │  - Notifications  │
                     └───────────────────┘

                     ┌─── admin BFF ────┐
Admin panel ──────►  │  aggregates:      │────► User Service
                     │  - Full user data │────► Audit Log
                     │  - Audit log      │────► Metrics
                     │  - Metrics        │
                     └───────────────────┘
```

**API Gateway** (nginx, Kong, Spring Cloud Gateway) — сетевой facade перед всеми backend'ами. Единая точка входа, TLS, аутентификация, rate limiting, routing.

Отличие BFF от Gateway:
- Gateway — тонкий: routing, cross-cutting concerns (auth, rate limit).
- BFF — толстый: содержит бизнес-агрегацию для конкретного клиента.

В КНП: `isnaknpgateway` — Gateway (Spring Cloud Gateway или nginx-based).

### 1.7 Facade vs Adapter — важное различие

Часто путают на собесе.

**Adapter** — делает **несовместимое совместимым**. У нас есть класс с интерфейсом X, а клиент ожидает интерфейс Y. Adapter — обёртка, конвертирующая X → Y.

```java
// Third-party класс (не можем менять):
class LegacyLogger {
    void writeMessage(String severity, String message) { /* ... */ }
}

// Наш интерфейс:
interface Logger {
    void info(String msg);
    void error(String msg);
}

// Adapter:
class LegacyLoggerAdapter implements Logger {
    private final LegacyLogger legacy;
    
    public void info(String msg) { legacy.writeMessage("INFO", msg); }
    public void error(String msg) { legacy.writeMessage("ERROR", msg); }
}
```

**Facade** — **упрощает** уже совместимое. У нас есть подсистема с валидным API, но сложным. Facade — не адаптирует, а объединяет.

Ключевое:
- Adapter меняет форму интерфейса (не количество).
- Facade **скрывает N интерфейсов за одним** упрощённым (меняет количество и уровень).

Adapter говорит: «переведи с языка X на язык Y».  
Facade говорит: «делай сложные вещи одной командой».

### 1.8 Facade vs Mediator

**Mediator** — координирует **peer-to-peer** взаимодействие между **равными** участниками. Никто не главный, mediator в центре. Пример: чат-комната — каждый пользователь пишет в комнату, комната рассылает всем.

**Facade** — **однонаправленный** доступ клиента к подсистеме. Клиент → facade → subsystem. Клиент **выше** уровнем чем подсистема.

Mediator — про decoupling peers. Facade — про упрощение доступа.

### 1.9 Facade vs Decorator

Про Decorator подробнее в §2 (сравнение с Proxy). Кратко: Decorator сохраняет тот же интерфейс что и wrapped. Facade имеет свой уникальный интерфейс.

### 1.10 Anti-pattern: God Facade

Опасность: facade становится "God object" — толстый класс с 30 методами и знанием всей системы.

Признаки:
- 500+ строк в facade.
- Множественные несвязанные операции в одном классе.
- Все запросы идут через один facade.
- Тесты facade огромные, с 10 mocks.

Fix:
- Разделить по доменам: `OrderFacade`, `UserFacade`, `ReportFacade`.
- Внутри facade делегировать бизнес-логику **приложенческим сервисам** (application services в DDD).
- Facade — только orchestration, не бизнес-правила.

### 1.11 Тесты Facade

Facade — orchestration → тесты интеграционные (mock подсистем):

```java
@ExtendWith(MockitoExtension.class)
class OrderFacadeTest {
    @Mock InventoryService inventory;
    @Mock PaymentGateway payment;
    @Mock ShippingService shipping;
    @Mock NotificationService notification;
    @Mock AuditLog audit;
    
    @InjectMocks OrderFacade facade;
    
    @Test
    void placeOrder_success_callsAllSubsystems() {
        when(inventory.hasStock(any())).thenReturn(true);
        when(inventory.reserve(any())).thenReturn(new ReservationId(1L));
        when(payment.charge(any(), any())).thenReturn(PaymentResult.success("txn-1"));
        when(shipping.createShipment(any())).thenReturn(new ShipmentId(2L));
        
        OrderResult result = facade.placeOrder(request);
        
        assertTrue(result.isSuccess());
        verify(notification).sendOrderConfirmation(any(), any());
        verify(audit).log(eq("ORDER_PLACED"), any(), any(), any());
    }
    
    @Test
    void placeOrder_paymentFails_releasesReservation() {
        when(inventory.hasStock(any())).thenReturn(true);
        when(inventory.reserve(any())).thenReturn(new ReservationId(1L));
        when(payment.charge(any(), any())).thenReturn(PaymentResult.failed("insufficient funds"));
        
        OrderResult result = facade.placeOrder(request);
        
        assertFalse(result.isSuccess());
        verify(inventory).releaseReservation(any());   // rollback
        verify(shipping, never()).createShipment(any());
    }
}
```

Тесты verify правильный orchestration flow, включая compensation actions.

### 1.12 Когда НЕ использовать Facade

- Подсистема из 2-3 методов — прямой вызов проще, facade — overhead.
- Клиент ДЕЙСТВИТЕЛЬНО нуждается в fine-grained control (например performance-critical low-level DB access).
- Facade добавляет indirection без упрощения (просто forwarder).

Правило: **начинай без facade, добавляй когда чувствуешь боль**. Facade — рефакторинг, не изначальный дизайн.

---

## 2. Proxy — детально

### 2.1 Мотивация

Есть объект, к которому нужно контролировать доступ. Причины:
1. **Ленивая инициализация** — real object дорого создавать, откладываем до первого использования.
2. **Контроль доступа** — auth check перед вызовом.
3. **Логирование / audit** — записать что кто вызвал.
4. **Кэширование** — вернуть кэш если есть, иначе вызвать real object.
5. **Транзакции** — открыть/закрыть транзакцию вокруг метода.
6. **Remote invocation** — real object на другой машине, proxy общается по сети.
7. **Reference counting / smart pointer** — вести счётчик использований.
8. **Read/write разделение** — proxy шлёт reads на реплику, writes на master.

Общая идея: **proxy имеет тот же интерфейс что real object**, но добавляет поведение.

### 2.2 Формальное определение (GoF)

*Provide a surrogate or placeholder for another object to control access to it.*

### 2.3 Виды Proxy

**Virtual Proxy** — ленивое создание. Пока никто не вызвал реальную операцию — real object не существует.
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

Hibernate lazy loading — virtual proxy. Пока не обратился к полю связанной entity — real query не выполнен. `user.getOrders().size()` триггерит SQL SELECT.

**Protection Proxy** — контроль доступа по правам.
```java
class SecureBankAccount implements BankAccount {
    private final BankAccount real;
    private final User currentUser;
    
    public void transfer(long amount, String to) {
        if (!currentUser.hasRole("ADMIN") && amount > 10000) {
            throw new SecurityException("Large transfer requires admin role");
        }
        real.transfer(amount, to);
    }
}
```

Spring Security `@PreAuthorize` — protection proxy.

**Remote Proxy** — real object на другой машине. Proxy общается по сети.
```java
class UserServiceStub implements UserService {   // Feign / gRPC-generated
    public User getUser(Long id) {
        HttpResponse resp = httpClient.get("http://user-service/users/" + id);
        return jackson.readValue(resp.body(), User.class);
    }
}
```

Feign, gRPC stubs, RMI, EJB remote — remote proxy.

**Smart Proxy** — доп. поведение при доступе: reference counting, caching, logging, metrics.
```java
class CachingUserServiceProxy implements UserService {
    private final UserService real;
    private final Cache<Long, User> cache;
    
    public User getUser(Long id) {
        return cache.get(id, () -> real.getUser(id));
    }
}
```

Spring `@Cacheable` — smart proxy.

**Copy-on-Write Proxy** — real object shared, proxy делает копию только при modification. Optimize memory.

**Firewall Proxy** — protection от external attacks (network level).

**Synchronization Proxy** — thread-safe wrapper над non-thread-safe object. `Collections.synchronizedList(list)` — этот паттерн.

### 2.4 Классический пример — Virtual Proxy

```java
interface Image {
    void display();
}

class RealImage implements Image {
    private final String filename;
    
    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();   // expensive
    }
    
    private void loadFromDisk() {
        System.out.println("Loading " + filename);
        // 500 ms disk read
    }
    
    public void display() {
        System.out.println("Displaying " + filename);
    }
}

class ImageProxy implements Image {
    private final String filename;
    private RealImage realImage;
    
    public ImageProxy(String filename) {
        this.filename = filename;
        // Ничего не загружаем!
    }
    
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename);
        }
        realImage.display();
    }
}

// Использование
Image img = new ImageProxy("photo.jpg");   // instant
// ... user не открыл вкладку ...
// realImage не создан, память экономится
img.display();   // теперь загружаем
```

Клиент **не знает** что говорит с proxy. Интерфейс `Image` тот же.

---

## 3. Static vs Dynamic Proxy

### 3.1 Static Proxy

Пишешь proxy-класс руками (как выше `ImageProxy`, `CachingUserServiceProxy`).

**Плюсы**:
- Явно видно что происходит.
- Compile-time type safety.
- Легко отлаживать.

**Минусы**:
- Boilerplate: N методов интерфейса × 2-3 строки delegate каждый.
- Изменяется интерфейс → надо править proxy.
- Не работает для универсального case (например, "любой сервис под @Transactional").

### 3.2 Dynamic Proxy

Класс proxy создаётся **на лету** (в runtime или compile-time через bytecode manipulation). Обрабатывает **любой** метод через central `InvocationHandler` / `MethodInterceptor`.

Две реализации в Java:
- **JDK Dynamic Proxy** (`java.lang.reflect.Proxy`) — только для интерфейсов.
- **CGLIB** — bytecode manipulation, работает с классами (наследование).

Spring использует их для AOP, @Transactional, @Async, @Cacheable, @PreAuthorize.

---

## 4. JDK Dynamic Proxy — как работает изнутри

### 4.1 API

```java
Object proxy = Proxy.newProxyInstance(
    classLoader,                    // где загрузить сгенерированный класс
    new Class<?>[]{MyInterface.class},   // какие интерфейсы реализовать
    new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) 
                throws Throwable {
            // Тут любая логика ДО, ПОСЛЕ, ВОКРУГ.
            System.out.println("Called: " + method.getName());
            return method.invoke(realObject, args);   // сам вызов real
        }
    }
);

MyInterface p = (MyInterface) proxy;
p.doSomething();
```

### 4.2 Что происходит внутри

JDK при первом вызове `Proxy.newProxyInstance(...)` для данной комбинации interfaces:

1. **Генерирует bytecode** класса `$Proxy0` (или следующий номер), который:
   - `implements` все указанные интерфейсы.
   - Extends `java.lang.reflect.Proxy`.
   - Для каждого метода интерфейса — реализация, которая вызывает `handler.invoke(this, method, args)`.
2. **Cache** класса в `ProxyGenerator` — второй раз для тех же интерфейсов используется тот же класс.
3. **Загружает** через указанный `ClassLoader`.
4. **Instantiates** proxy object.

Сгенерированный класс (упрощённо):
```java
public final class $Proxy0 extends Proxy implements MyInterface {
    private static Method m_doSomething;
    private static Method m_hashCode;
    private static Method m_equals;
    private static Method m_toString;
    
    static {
        m_doSomething = MyInterface.class.getMethod("doSomething");
        // ...
    }
    
    public $Proxy0(InvocationHandler h) {
        super(h);
    }
    
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

Смотреть сгенерированные классы:
```
-Djdk.proxy.ProxyGenerator.saveGeneratedFiles=true
```

Сохранит `$Proxy0.class` в текущей директории. Открой в javap — увидишь.

### 4.3 Плюсы JDK Proxy

- **Встроен в JDK** — нет зависимостей.
- **Быстрая генерация** класса.
- **Простой mental model** — proxy implements interface.

### 4.4 Минусы JDK Proxy

- **Только для интерфейсов!** Если у тебя `class UserService` (без интерфейса) — JDK proxy не сработает.
- Reflection overhead на `method.invoke()` (значительно меньше в JDK 9+, но всё ещё дороже direct call).
- Все методы интерфейса — final в proxy (нельзя частично override).

### 4.5 Ограничения — важно на собесе

Работает **только с методами объявленными в интерфейсе**. Если class implements MyInterface + добавляет public method — proxy имеет только методы из MyInterface.

```java
interface UserService {
    User findById(Long id);
}

class UserServiceImpl implements UserService {
    public User findById(Long id) { /* ... */ }
    public User findByEmail(String email) { /* ... */ }   // не в интерфейсе!
}

UserService proxy = (UserService) Proxy.newProxyInstance(...);
proxy.findById(1L);         // ✅ ok
((UserServiceImpl) proxy).findByEmail("...");   // ❌ ClassCastException! Proxy НЕ instanceof UserServiceImpl
```

---

## 5. CGLIB — bytecode generation

### 5.1 Мотивация

Что если класс **не имеет интерфейса**? JDK proxy бесполезен. Тут CGLIB.

**CGLIB** (Code Generation Library) — генерирует **subclass** нашего класса, overriding методы через `MethodInterceptor`.

### 5.2 API

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UserService.class);   // наследуем
enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, 
                            MethodProxy proxy) throws Throwable {
        System.out.println("Called: " + method.getName());
        return proxy.invokeSuper(obj, args);   // вызов оригинального метода
    }
});
UserService proxy = (UserService) enhancer.create();
```

### 5.3 Что генерируется

CGLIB создаёт класс `UserService$$EnhancerByCGLIB$$abc123` который:
- Extends `UserService`.
- Overrides каждый non-final public method.
- Каждый override делегирует в `MethodInterceptor`.

Схематично:
```java
public class UserService$$EnhancerByCGLIB extends UserService {
    private MethodInterceptor interceptor;
    
    @Override
    public User findById(Long id) {
        return (User) interceptor.intercept(this, 
            /* Method obj */, new Object[]{id}, 
            /* MethodProxy */);
    }
    
    // ... все другие public methods
}
```

`MethodProxy.invokeSuper()` — вызывает `super.findById(id)` через быстрый bypass (сгенерированный FastClass, не reflection).

### 5.4 Плюсы CGLIB

- **Работает без интерфейса**.
- **Быстрее JDK Proxy** для method invocation (нет reflection на hot path).
- Более гибкий (можно перехватывать hashCode, equals, toString раздельно).

### 5.5 Минусы CGLIB

- **Не может proxy final classes** (нельзя extends).
- **Не может proxy final methods** (нельзя override).
- **Не может proxy private methods** (нельзя override).
- Конструктор parent класса вызывается при создании proxy (если parent имеет side effects — они выполнятся; часто нужен default constructor).
- Дополнительная зависимость (в Spring 5+ CGLIB встроен в `spring-core` в repackaged форме `org.springframework.cglib.*`).

### 5.6 Ограничения CGLIB на final

```java
class MyService {
    public final String getName() { return "..."; }   // final!
    public void doWork() { /* ... */ }
}

MyService proxy = /* CGLIB proxy of MyService */;
proxy.doWork();     // ✅ intercepted
proxy.getName();    // ❌ вызывается напрямую, БЕЗ interceptor
```

Классические баги: `@Transactional` метод помечен `final` — proxy не может override → аннотация игнорируется.

---

## 6. ByteBuddy — современная альтернатива

**ByteBuddy** — библиотека для bytecode manipulation. Более гибкая и быстрая чем CGLIB. Активно развивается (CGLIB — заброшена, последний release ~2019).

Использует Mockito, Hibernate, некоторые Spring internals в новых версиях.

Пример:
```java
Class<?> dynamicType = new ByteBuddy()
    .subclass(UserService.class)
    .method(ElementMatchers.named("findById"))
    .intercept(MethodDelegation.to(new MyInterceptor()))
    .make()
    .load(UserService.class.getClassLoader())
    .getLoaded();

UserService proxy = (UserService) dynamicType.getDeclaredConstructor().newInstance();
```

Плюсы:
- Type-safe DSL.
- Работает с Java 21+ модулями.
- Активно поддерживается.
- Лучшая performance.

Минусы:
- Отдельная зависимость.
- Более сложный API.

Spring может переключиться на ByteBuddy в будущих версиях, но сейчас CGLIB — default.

---

## 7. Spring AOP — практика Proxy

### 7.1 Общая механика

Spring использует **Proxy pattern** для всех aspect'ов (aspects):
- `@Transactional` → `TransactionInterceptor`.
- `@Async` → `AsyncExecutionInterceptor`.
- `@Cacheable` / `@CacheEvict` → `CacheInterceptor`.
- `@PreAuthorize` / `@Secured` → `MethodSecurityInterceptor`.
- `@Retryable` (Spring Retry) → `RetryInterceptor`.
- Custom aspects через `@Aspect` + `@Around` / `@Before` / `@After`.

При bean creation:
1. Spring создаёт real object.
2. Проверяет: есть ли применимые advisors (например `@Transactional` на методе)?
3. Если да — **оборачивает в proxy**.
4. Injects proxy в других beans, не real.

Клиент (другой bean) `@Autowired UserService us` получает **proxy**, не real object.

### 7.2 JDK vs CGLIB — как выбирает Spring

Правила:
- Bean **implements interface** → JDK Proxy (default).
- Bean **не implements** → CGLIB.
- Force CGLIB: `spring.aop.proxy-target-class=true` или `@EnableTransactionManagement(proxyTargetClass = true)`.

Spring Boot **с 2.0** — CGLIB by default (`proxy-target-class=true` для transaction management). Изменение для удобства (не надо думать про интерфейсы).

### 7.3 Как работает `@Transactional` через Proxy

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

Клиент не знает про транзакцию. Аннотация + proxy делают всё.

Детально — `33-transactional-internals.md`.

### 7.4 Self-invocation problem — гоча №1

Классическая ошибка:
```java
@Service
public class UserService {
    
    public void publicMethod() {
        internalMethod();   // ❌ прямой вызов, не через proxy!
    }
    
    @Transactional
    public void internalMethod() {
        // ...
    }
}
```

`publicMethod` вызывает `internalMethod` через `this` (не через proxy). Транзакция **НЕ открывается** — proxy не в пути вызова.

Почему? Proxy оборачивает bean снаружи. Внутренний вызов `this.internalMethod()` — прямая ссылка на real object, minует proxy.

Схематично:
```
Client → Proxy → real UserService.publicMethod()
                          │
                          ▼ (this.internalMethod — прямой вызов на real object)
                     UserService.internalMethod()   ← proxy не в пути
                     @Transactional игнорируется!
```

**Fixes**:

1. **Self-injection** (уродливо, но работает):
   ```java
   @Autowired UserService self;   // Spring inject'ит proxy сам к себе
   
   public void publicMethod() {
       self.internalMethod();   // через proxy
   }
   ```

2. **AopContext**:
   ```java
   @EnableAspectJAutoProxy(exposeProxy = true)
   
   public void publicMethod() {
       ((UserService) AopContext.currentProxy()).internalMethod();
   }
   ```

3. **Разделить на два bean'а**:
   ```java
   @Service class OrderFacade {
       @Autowired OrderService orderService;
       public void placeOrder() {
           orderService.doTransactionalWork();   // через proxy!
       }
   }
   
   @Service class OrderService {
       @Transactional
       public void doTransactionalWork() { }
   }
   ```

4. **AspectJ** (compile-time weaving, не proxy) — работает без proxy indirection. Но сложная настройка.

**Правило**: `@Transactional` работает только для **внешних** вызовов через proxy. Внутренние вызовы (`this.xxx()`) не пойдут через interceptor.

### 7.5 Другие ограничения Spring Proxy

- **private methods** — не proxy'ятся. `@Transactional private` игнорируется.
- **final methods** (при CGLIB) — не proxy'ятся.
- **final classes** — не могут быть CGLIB-обёрнуты.
- **static methods** — не proxy'ятся вообще.
- **package-private methods** — technically могут proxy'иться, но зависит от Spring версии и настроек. Не полагайся.

**Правило**: `@Transactional` / `@Async` / `@Cacheable` только на **public** методах, non-final.

### 7.6 Debug — как понять что там proxy

```java
@Autowired UserService us;

@PostConstruct
void init() {
    System.out.println(us.getClass().getName());
    // Output:
    //   UserService$$EnhancerBySpringCGLIB$$abc123    ← CGLIB proxy
    //   com.sun.proxy.$Proxy42                          ← JDK proxy
    //   com.example.UserService                         ← real (no proxy)
}
```

В IDE — можно поставить breakpoint в `TransactionInterceptor.invoke` и увидеть full stacktrace.

### 7.7 AOP alternatives

**Spring AOP** — proxy-based. Ограничения выше.

**AspectJ** — bytecode weaving:
- **Compile-time weaving (CTW)** — javac + iajc modifies .class files. Слоу build, но zero runtime cost.
- **Load-time weaving (LTW)** — Java agent modifies bytecode at classloading. Runtime cost.

AspectJ mощнее (перехватывает всё, включая self-invocation), но сложнее. Для типичного Spring-приложения — Spring AOP достаточно.

---

## 8. @Async, @Cacheable, @PreAuthorize — все proxy-based

### 8.1 @Async

```java
@Async
public CompletableFuture<Report> generateReport(Long id) {
    // ... slow computation
    return CompletableFuture.completedFuture(report);
}
```

Proxy при вызове:
1. Не выполняет метод immediately.
2. Оборачивает в `Runnable` / `Callable`.
3. Submit в `TaskExecutor` (обычно `ThreadPoolTaskExecutor` или virtual thread executor).
4. Возвращает `CompletableFuture` (или void).
5. Реальный метод выполняется в другом thread'е.

Ограничения те же: self-invocation, private, final, static — не работают.

Как настроить virtual threads:
```java
@Bean
AsyncTaskExecutor asyncTaskExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

### 8.2 @Cacheable

```java
@Cacheable("users")
public User findById(Long id) {
    return repo.findOne(id);   // slow DB query
}
```

Proxy:
1. Компилирует key из method args (default — все args).
2. Проверяет cache.
3. Cache hit → возвращает cached value.
4. Cache miss → вызывает real method, кладёт в cache, возвращает.

`@CacheEvict` — proxy удаляет из cache перед/после вызова.

### 8.3 @PreAuthorize (Spring Security)

```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.name")
public User getUserProfile(Long userId) { /* ... */ }
```

Proxy:
1. Достаёт `Authentication` из `SecurityContext`.
2. Оценивает SpEL expression.
3. Если false → `AccessDeniedException`.
4. Иначе — real method.

### 8.4 @Retryable (Spring Retry)

```java
@Retryable(value = TransientException.class, 
           maxAttempts = 3, 
           backoff = @Backoff(delay = 1000, multiplier = 2))
public String callFlakyAPI() { /* ... */ }
```

Proxy:
1. Try 1 → exception (TransientException).
2. Sleep 1000ms.
3. Try 2 → exception.
4. Sleep 2000ms.
5. Try 3 → success or `ExhaustedRetryException`.

Каждая аннотация — свой interceptor в цепочке proxy.

### 8.5 Порядок interceptor'ов

Если на методе `@Transactional` + `@Cacheable` + `@Async` — в каком порядке?

Spring выстраивает **цепочку interceptor'ов** по priority. Order (низкое = ближе снаружи):
- `@Async` (обычно `Ordered.LOWEST_PRECEDENCE`).
- `@Transactional` (по умолчанию LOWEST).
- `@Cacheable` (LOWEST).

Управлять через `@Order` или `@EnableTransactionManagement(order=...)`.

**Классическая ошибка**: `@Transactional` + `@Cacheable` на одном методе.
- Если `@Cacheable` снаружи `@Transactional` → cache hit не открывает транзакцию (ok).
- Если `@Transactional` снаружи `@Cacheable` → лишняя транзакция для cache hit (проблема).

Обычно **Cache снаружи Transaction** — правильно. Проверь порядком.

---

## 9. Proxy vs Decorator — принципиально важно

Часто на собесе. Оба паттерна:
- Оборачивают объект.
- Реализуют тот же интерфейс.
- Делегируют вызовы wrapped object.

**В чём разница?**

### 9.1 Намерение

**Proxy** — **контроль доступа** к объекту (кэш, security, lazy, remote). Клиент **не знает** что говорит с proxy — думает что с real.

**Decorator** — **добавление функциональности** к объекту динамически. Клиент **знает** что декорировал (сам оборачивает).

### 9.2 Управление жизненным циклом

**Proxy** — сам создаёт/управляет real object (или получает его из фабрики). Клиент не знает про real.

**Decorator** — клиент **явно** создаёт wrapped и decorator. Клиент собирает цепочку.

### 9.3 Композиция

**Decorator** — часто цепочка: `new EncryptedStream(new BufferedStream(new FileStream("f")))`. Каждый добавляет своё поведение.

**Proxy** — обычно **один** proxy. Не собирается в цепочку от клиента.

### 9.4 Пример Decorator в JDK

```java
InputStream in = new FileInputStream("f.txt");
InputStream buffered = new BufferedInputStream(in);
InputStream zipped = new GZIPInputStream(buffered);

// Клиент собирает цепочку:
// - FileInputStream (real - read from disk)
// - BufferedInputStream (add buffering)
// - GZIPInputStream (add decompression)
```

Каждый уровень **сохраняет интерфейс** InputStream, но **добавляет** поведение.

### 9.5 Резюме

| | Proxy | Decorator |
|-|-------|-----------|
| Намерение | Control access | Add behavior |
| Клиент знает | Нет (proxy invisible) | Да (клиент wraps) |
| Real object owner | Proxy | Клиент |
| Composition | Обычно один | Часто цепочка |
| Классические use cases | Cache, Lazy, Remote, Security | I/O streams, GUI toolkit |
| Spring examples | @Transactional, @Async | Java IO, Redis Cache decorators |

**Простой mnemonic**: Proxy решает **«кто может»**. Decorator решает **«что дополнительно делать»**.

---

## 10. Полная сравнительная таблица

| Pattern | Intent | Interface | Multiple targets | Client aware |
|---------|--------|-----------|------------------|--------------|
| Facade | Simplify interface to subsystem | New (simplified) | Yes (subsystem) | Yes |
| Adapter | Convert incompatible interface | New (target) | No (one adaptee) | Sometimes |
| Proxy | Control access to object | Same as target | No | No (transparent) |
| Decorator | Add responsibilities dynamically | Same as target | No | Yes (composes) |
| Mediator | Encapsulate object interactions | New | Yes (colleagues) | Yes |
| Bridge | Decouple abstraction from impl | Different (both) | No | Yes |
| Composite | Treat individual and group uniformly | Same (tree) | Yes | No |

Часто путаемые:
- **Adapter vs Facade**: Adapter меняет ФОРМУ интерфейса; Facade — упрощает СЛОЖНОСТЬ.
- **Proxy vs Decorator**: Proxy контролирует доступ (invisible); Decorator добавляет функциональность (client-composed).
- **Proxy vs Adapter**: Adapter меняет интерфейс; Proxy сохраняет интерфейс.
- **Facade vs Mediator**: Facade — вертикально (клиент выше подсистемы); Mediator — горизонтально (peers).

---

## 11. Практика: где что использовать

### 11.1 Facade когда:
- Клиенту нужно N связанных операций → objединить в один "typical use case" метод.
- Скрыть 3rd-party API за собственным domain-specific API (протестируется легче).
- Разделить orchestration от business logic.
- Уменьшить coupling между слоями.

### 11.2 Proxy когда:
- Нужно **прозрачно** добавить cross-cutting поведение (log, cache, tx, auth) — Spring AOP.
- Real object дорого создавать → lazy load (Hibernate, JPA).
- Real object на другой машине → remote proxy (Feign, gRPC).
- Нужно counted / synchronized доступ.

### 11.3 Не Proxy когда:
- Клиент **сам** должен решать композицию — используй Decorator.
- Меняешь интерфейс — используй Adapter.

### 11.4 Комбинация

Часто в реальных системах — вместе:

```
Client
   ↓
[Proxy for @Transactional]        ← proxy adds tx
   ↓
[Real OrderFacade]                 ← facade orchestrates
   ↓
[Proxy for @Cacheable]  → [Real UserService]     ← proxy adds cache
[Proxy for @Transactional] → [Real PaymentGateway]
[Proxy for @Retryable] → [Real ExternalApiClient]
```

Facade — верхний уровень (orchestration). Proxy — по всей системе для cross-cutting concerns.

---

## 12. Собесные вопросы

### Q1: Что такое паттерн Facade?

Facade — структурный паттерн. Предоставляет **упрощённый интерфейс** к сложной подсистеме. Клиент вместо знания про 5 классов подсистемы вызывает один метод facade.

Пример: `OrderFacade.placeOrder(request)` внутри координирует Inventory, Payment, Shipping, Notification, Audit. Клиент не знает про них.

Плюсы: снижение coupling, читаемость, одно место для orchestration.
Минусы: risk of "God facade" — толстый класс со всеми методами системы.

### Q2: Разница Facade и Adapter?

**Adapter** — переводит между несовместимыми интерфейсами. У нас класс с API X, клиент ожидает Y. Adapter — обёртка X → Y.

**Facade** — упрощает совместимую подсистему. Не переводит, а объединяет N сервисов за одним простым API.

Adapter меняет ФОРМУ интерфейса (1-to-1). Facade — УРОВЕНЬ (N-to-1).

### Q3: Что такое паттерн Proxy?

Proxy — структурный паттерн. Представитель другого объекта. Имеет **тот же интерфейс** что и real object, но контролирует доступ / добавляет поведение.

Виды:
- **Virtual** — lazy init (Hibernate lazy loading).
- **Protection** — auth check (Spring Security).
- **Remote** — real на другой машине (Feign, gRPC).
- **Smart** — cache, log, reference counting (Spring @Cacheable).

Клиент **не знает** что говорит с proxy.

### Q4: Разница Proxy и Decorator?

Обычная путаница. Оба wrap object с тем же интерфейсом.

**Proxy** — контроль **доступа**. Proxy владеет real object (создаёт lazy или получает из фабрики). Клиент **прозрачно** говорит с proxy.

**Decorator** — добавляет **функциональность**. Клиент **явно** wraps: `new BufferedStream(new FileStream(...))`. Часто цепочка.

Proxy: клиент видит только proxy.
Decorator: клиент собирает цепочку сам.

### Q5: JDK Dynamic Proxy vs CGLIB?

**JDK Proxy** (`java.lang.reflect.Proxy`):
- Требует **интерфейс**.
- Встроен в JDK.
- Создаёт класс implements interface, delegates to InvocationHandler.
- Не может proxy классы без интерфейса.

**CGLIB**:
- Работает **без интерфейса** — создаёт subclass через bytecode generation.
- Не может proxy final classes/methods.
- Быстрее JDK Proxy (нет reflection на hot path).
- Требует public no-arg конструктор (или использует Objenesis).

Spring:
- Interface есть → JDK Proxy (default).
- Interface нет → CGLIB.
- `spring.aop.proxy-target-class=true` → всегда CGLIB.
- Spring Boot 2.0+ — CGLIB по умолчанию.

### Q6: Как работает `@Transactional`?

Spring оборачивает bean в proxy (JDK или CGLIB). При вызове метода помеченного `@Transactional`:

1. Proxy перехватывает вызов.
2. `TransactionInterceptor.invoke()`:
   - Читает метаданные (propagation, isolation, readOnly).
   - Через `PlatformTransactionManager` открывает транзакцию: `dataSource.getConnection()`, `setAutoCommit(false)`, binds connection в ThreadLocal.
3. Вызывает real метод.
4. После return:
   - RuntimeException → rollback.
   - Ok → commit.
5. `conn.close()` (возврат в pool).

Клиент видит обычный вызов метода. Proxy делает всю магию.

### Q7: Что такое self-invocation problem?

Классическая ошибка со Spring Proxy:
```java
@Service
class UserService {
    public void a() {
        b();   // ❌ не через proxy!
    }
    
    @Transactional
    public void b() { }
}
```

`a()` вызывает `b()` через `this` (прямая ссылка на real object) — proxy не в пути → `@Transactional` игнорируется.

Причина: proxy оборачивает bean **снаружи**. Внутренние вызовы `this.method()` минуют proxy.

Fixes:
1. Self-injection (`@Autowired UserService self`).
2. `AopContext.currentProxy()`.
3. Разделить на два bean'а (facade + service).
4. AspectJ (compile-time weaving — работает без proxy indirection).

### Q8: Почему `@Transactional` не работает на private методе?

Spring proxy (JDK или CGLIB) может перехватывать только **public** методы:
- JDK Proxy — реализует **интерфейс**, интерфейсные методы всегда public.
- CGLIB — создаёт **subclass**, private методы нельзя override.

Fix: сделать метод public. Или вынести в отдельный bean.

Также не работает:
- final methods (CGLIB не может override).
- static methods (не proxy'ятся).
- private (не proxy'ятся).

### Q9: Как отладить: proxy сработал или нет?

```java
@Autowired UserService us;

@PostConstruct
void check() {
    System.out.println(us.getClass().getName());
}
```

- `com.example.UserService` → нет proxy (плохо, если ждём @Transactional).
- `UserService$$EnhancerBySpringCGLIB$$...` → CGLIB proxy.
- `com.sun.proxy.$Proxy42` → JDK proxy.

Ещё можно поставить breakpoint в `TransactionInterceptor.invoke()` — увидишь весь stacktrace вызова.

### Q10: Facade vs Mediator?

**Facade** — **однонаправленный**: клиент выше подсистемы, вызывает facade → facade вызывает подсистему. Не coordinate peers.

**Mediator** — **peer-to-peer**: coordinate несколько **равных** объектов, которые общаются через mediator (не напрямую). Пример: chat room — users шлют в room, room routes to other users.

Facade — снаружи → внутрь. Mediator — сбоку → сбоку (координация peers).

### Q11: Что такое Virtual Proxy?

Proxy с **ленивой инициализацией** real object'а. Пока клиент не вызвал реальную операцию — real не создан.

Классический пример: Hibernate lazy loading.
```java
@ManyToOne(fetch = FetchType.LAZY)
private User user;   // hibernate создаёт proxy, real query только при user.getName()
```

Плюсы: экономия ресурсов если объект дорог, но не всегда нужен.
Минусы: доступ вне scope (например LazyInitializationException в Hibernate после закрытия session).

### Q12: Что такое Remote Proxy?

Proxy которое **прозрачно** прокси в вызов на другой машине. Клиент вызывает proxy как обычный объект, proxy сериализует args, шлёт по сети, десериализует response.

Примеры:
- **Feign clients** (Spring Cloud) — генерирует remote proxy из annotated interface. Клиент `@Autowired UserServiceClient` — под капотом HTTP proxy.
- **gRPC generated stubs** — remote proxy к gRPC service.
- **RMI, EJB remote** — старые enterprise.

Клиент пишет `userClient.getUser(1L)` — а под капотом HTTP GET, JSON parse, error handling.

### Q13: Почему нельзя self-invocation в Spring AOP?

Потому что Spring AOP — **proxy-based**. Proxy оборачивает bean снаружи. Клиент → proxy → real.

Внутри real object'а `this` — ссылка на **сам real** (не proxy). Вызов `this.method()` не проходит через proxy.

AspectJ (не Spring AOP) использует **bytecode weaving** — модифицирует сам .class файл. Тогда self-invocation работает, потому что аспект встроен в код.

Trade-off: Spring AOP проще настроить, AspectJ мощнее.

### Q14: Что такое Protection Proxy?

Proxy который **проверяет права** перед делегированием real object. Spring Security `@PreAuthorize`:
```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { }
```

Proxy при вызове:
1. Читает current authentication из SecurityContext.
2. Evaluates SpEL expression.
3. Если false → AccessDeniedException.
4. Иначе — real method.

Клиент не пишет security check в коде — proxy делает.

### Q15: Может ли Facade быть Proxy?

Технически — да, часто комбинируются в реальном коде.

Пример: **API Gateway** — facade для микросервисов (aggregation) + proxy (auth, rate limiting, routing).

**BFF (Backend for Frontend)** — фасад для UI, часто с кэшем (smart proxy).

Формально это разные responsibilities: facade **упрощает**, proxy **контролирует**. В одном объекте могут быть оба.

### Q16: Что произойдёт если Spring bean с `@Transactional` — final class?

- Если CGLIB (default) — Spring **не сможет создать proxy** → BeanCreationException.
- Если JDK Proxy — работает (proxy не extends class, а implements interface).

Fix для CGLIB: убрать `final` с класса. Или force JDK Proxy с интерфейсом.

Похожая проблема с Kotlin — все классы `final` by default. Kotlin plugin `all-open` открывает `@Component` классы автоматически.

### Q17: Разница Proxy и Bridge?

Часто путают.

**Proxy** — тот же интерфейс что и real object, wraps один real.

**Bridge** — separates **abstraction** от **implementation**, обе могут развиваться независимо. Разные интерфейсы для абстракции и impl.

Классический пример Bridge: `Shape` (abstraction) + `Renderer` (implementation).
```java
abstract class Shape {
    protected Renderer renderer;   // bridge к implementation
    abstract void draw();
}

class Circle extends Shape {
    void draw() { renderer.renderCircle(...); }
}

interface Renderer {
    void renderCircle(...);
    void renderSquare(...);
}

class OpenGLRenderer implements Renderer { ... }
class DirectXRenderer implements Renderer { ... }
```

Shape и Renderer развиваются независимо. Разные интерфейсы.

Proxy — один интерфейс, wraps target.

### Q18: Что такое AOP alliance MethodInterceptor?

Стандартный интерфейс для интерцепторов:
```java
public interface MethodInterceptor {
    Object invoke(MethodInvocation invocation) throws Throwable;
}
```

Spring использует его как основу advisors. Каждый interceptor (transaction, cache, security, async) — реализует этот интерфейс.

Chain: `Client → Proxy → [Advisor1 → Advisor2 → ... → real method]`. Каждый advisor вызывает `invocation.proceed()` для continue chain.

### Q19: Порядок аспектов при нескольких аннотациях?

Если `@Transactional` + `@Cacheable` + `@Async` — Spring создаёт цепочку interceptor'ов, порядок через `@Order` или property.

Default order:
- `@Async` — LOWEST_PRECEDENCE.
- `@Transactional` — LOWEST_PRECEDENCE.
- `@Cacheable` — LOWEST_PRECEDENCE.

Все одинаковые → недетерминированный порядок → problem.

Правильно: `@EnableTransactionManagement(order=100)`, `@EnableCaching(order=200)` — явно указать.

Обычный правильный порядок: cache → transaction → real (cache снаружи).

### Q20: Как реализовать custom AOP с @Aspect?

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

Spring создаёт proxy для beans в `com.example.service`. При каждом method call — `LoggingAspect.logAround` вызывается, timing logged.

Pointcut expressions (execution, within, args, @annotation) — селектят где применить.

Виды advice: `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`.

---

## 13. Мини-чеклист «прочитал — знаю»

За 3-5 секунд:

- [ ] Facade — упрощение доступа к сложной подсистеме.
- [ ] Клиент facade знает что за фасадом много всего.
- [ ] Facade vs Adapter: адаптер меняет форму, facade объединяет N сервисов.
- [ ] Facade vs Mediator: facade вертикально, mediator peer-to-peer.
- [ ] Anti-pattern: God Facade.
- [ ] JdbcTemplate, RestTemplate — Spring Facades.
- [ ] BFF, API Gateway — Facade в микросервисах.
- [ ] Proxy — представитель, тот же интерфейс что real.
- [ ] Клиент proxy не знает что говорит с proxy.
- [ ] Виды proxy: virtual, protection, remote, smart, cache.
- [ ] Virtual Proxy — lazy init (Hibernate lazy loading).
- [ ] Protection Proxy — auth check.
- [ ] Remote Proxy — Feign, gRPC stubs, RMI.
- [ ] Static Proxy — писать руками.
- [ ] Dynamic Proxy — генерация в runtime.
- [ ] JDK Dynamic Proxy — только для interface'ов, через InvocationHandler.
- [ ] CGLIB — bytecode generation, subclass, работает без interface.
- [ ] CGLIB не может final class / method.
- [ ] Spring Boot 2.0+ — CGLIB by default.
- [ ] Sprint Proxy invisible: `bean.getClass().getName()` покажет.
- [ ] @Transactional, @Async, @Cacheable, @PreAuthorize — все proxy-based.
- [ ] Self-invocation problem: this.method() минует proxy.
- [ ] Fix: self-inject / AopContext / разделить bean / AspectJ.
- [ ] @Transactional не работает на private, final, static.
- [ ] Proxy vs Decorator: proxy invisible, decorator composed by client.
- [ ] Proxy vs Adapter: proxy same interface, adapter changes interface.
- [ ] AOP interceptor chain: cache → transaction → real.
- [ ] `@Around` @Aspect + pointcut — custom AOP.
- [ ] AspectJ — bytecode weaving, работает с self-invocation.
- [ ] ByteBuddy — современный CGLIB replacement.

---

## Итог

**Facade** — упрощение. Один класс скрывает N подсистем за domain-specific API. Клиент знает про facade, не про подсистему. Anti-pattern — God Facade. Spring Templates (JdbcTemplate, RestTemplate) — примеры. В микросервисах — BFF/API Gateway.

**Proxy** — представитель. Тот же интерфейс что real, но с добавлением поведения (cache, security, lazy, remote). Клиент **не знает** про proxy. Виды: Virtual (lazy), Protection (auth), Remote (network), Smart (cache/log).

**Реализация Proxy в Java**:
- **Static** — писать руками.
- **JDK Dynamic** — только для interface'ов, через `InvocationHandler`, встроен.
- **CGLIB** — bytecode subclass, работает без interface, не final classes/methods.
- **ByteBuddy** — современная альтернатива CGLIB.

**Spring AOP** — proxy-based, аспекты через interceptors:
- `@Transactional` → `TransactionInterceptor`.
- `@Async` → `AsyncExecutionInterceptor`.
- `@Cacheable` → `CacheInterceptor`.
- `@PreAuthorize` → `MethodSecurityInterceptor`.

**Ключевые ограничения Spring Proxy**:
- Only **public** methods.
- Not **final** classes/methods (CGLIB).
- Not **static** methods.
- **Self-invocation** minует proxy — fix через self-inject / AopContext / разделение bean'ов.

**Сравнение с другими паттернами**:
- **Proxy vs Decorator** — proxy invisible + controls access; decorator composed by client + adds behavior.
- **Proxy vs Adapter** — proxy same interface; adapter changes interface.
- **Facade vs Adapter** — facade объединяет; adapter переводит.
- **Facade vs Mediator** — facade vertical; mediator peer-to-peer.

Дальше в теме: практика — включи `-Djdk.proxy.ProxyGenerator.saveGeneratedFiles=true` и посмотри что реально генерируется для `@Transactional`. Открой class через javap. Пойми на glass level что делает Spring AOP. Всё встанет на место.
