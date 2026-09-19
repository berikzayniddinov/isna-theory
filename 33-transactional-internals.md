# 33. @Transactional изнутри: прокси, PlatformTransactionManager

Как Spring на самом деле реализует `@Transactional`. Все детали, все подводные камни.

---

## 1. Общая картина

Когда ты пишешь:
```java
@Service
class OrderService {
    @Transactional
    public void createOrder(Order o) {
        repo.save(o);
        publisher.publish(o);
    }
}
```

Что Spring делает под капотом:

1. При создании бина `OrderService` Spring **оборачивает его в CGLib-прокси** (subclass).
2. Прокси-класс переопределяет метод `createOrder`, добавляя вокруг:
   - `beginTransaction()`.
   - Вызов оригинального метода.
   - `commit()` при success или `rollback()` при exception.
3. Везде, где ты `@Autowired OrderService` — получаешь **прокси**, не оригинал.

Разберём каждый компонент.

---

## 2. Три ключевых участника

### 2.1 TransactionInterceptor

`org.springframework.transaction.interceptor.TransactionInterceptor` — AOP-совет (advice), который выполняется вокруг каждого `@Transactional`-метода.

Внутри — `TransactionAspectSupport.invokeWithinTransaction()` (главный метод):
1. Извлечь `TransactionAttribute` (propagation, isolation, rollbackFor).
2. Взять нужный `PlatformTransactionManager`.
3. Открыть tx (`getTransaction()`).
4. Вызвать оригинальный метод.
5. При success → `commit()`.
6. При exception → проверить rollback rules → `rollback()` или `commit()`.

### 2.2 TransactionAttributeSource

Читает мета-информацию о транзакции для метода. Основная реализация — `AnnotationTransactionAttributeSource`:
- Ищет `@Transactional` на методе, потом на классе, потом на interface.
- Кэширует результат в `Map<Method, TransactionAttribute>`.

### 2.3 PlatformTransactionManager

Абстракция над конкретным механизмом транзакций.

Реализации:
- **`DataSourceTransactionManager`** — для plain JDBC.
- **`JpaTransactionManager`** — для JPA/Hibernate.
- **`JtaTransactionManager`** — для XA / distributed.
- **`ChainedTransactionManager`** — «best-effort» для нескольких (без реального 2PC).
- **`RabbitTransactionManager`** — для Rabbit.
- **`KafkaTransactionManager`** — для Kafka.

Spring Boot автоконфигурит **`JpaTransactionManager`** если есть JPA (spring-boot-starter-data-jpa).

Интерфейс:
```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition);
    void commit(TransactionStatus status);
    void rollback(TransactionStatus status);
}
```

---

## 3. Как создаётся прокси

### 3.1 BeanPostProcessor

`InfrastructureAdvisorAutoProxyCreator` — `BeanPostProcessor`, который на `postProcessAfterInitialization` смотрит: «этому бину нужен прокси?».

Если бин имеет метод с `@Transactional` (напрямую или через классовую аннотацию) → да.

### 3.2 Что выбирает Spring

- **JDK Dynamic Proxy** — если бин реализует интерфейсы. Прокси реализует эти интерфейсы.
- **CGLib Proxy** — если бин не имеет интерфейсов. Создаёт **subclass**.

Настройка:
```yaml
spring.aop.proxy-target-class: true       # всегда CGLib (default для Spring Boot)
```

По умолчанию Spring Boot использует CGLib. Плюс: работает даже если у бина есть интерфейсы.

### 3.3 Структура CGLib-прокси

```
Оригинал:
  OrderService
    public void createOrder(Order o) { ... }
    private void internalMethod() { ... }

Прокси (CGLib):
  OrderService$$SpringCGLIB$$0 extends OrderService
    public void createOrder(Order o) {
        interceptor.invoke(this, method, args, methodProxy);
        // → TransactionInterceptor выполняет tx-логику
        // → потом super.createOrder(o)
    }
    // internalMethod не переопределяется (private)
```

Прокси имеет **тот же интерфейс что оригинал** — можно инжектить как `OrderService`.

### 3.4 Ограничения CGLib

- **Не может проксировать `final`-классы** — нельзя extend.
- **Не может `final`-методы** — нельзя override.
- **Не может `private`-методы** — не переопределяются в subclass.
- **Не может `static`-методы** — не наследуются.
- Требует no-arg constructor (или пустой super) — CGLib создаёт instance для прокси.

### 3.5 JDK Dynamic Proxy vs CGLib

|| JDK | CGLib |
|---|---|---|
| Требует | Интерфейс | Extendable class |
| Ограничения | Только методы интерфейса | final/private/static не работают |
| Скорость | Немного быстрее | Немного медленнее |
| Default в Boot | Нет | Да |

---

## 4. Self-invocation — почему @Transactional не срабатывает

**Классический баг**.

```java
@Service
class OrderService {
    public void processAll(List<Order> orders) {
        for (Order o : orders) {
            this.processOne(o);      // ← вызов через this!
        }
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void processOne(Order o) {
        repo.save(o);
    }
}
```

Ожидание: `processOne` создаёт новую tx на каждой итерации.
Реальность: **@Transactional не срабатывает**. Всё в одной tx (или без tx).

### 4.1 Почему

Прокси перехватывает **внешние вызовы** — то есть вызовы через ссылку на прокси.

Когда `processAll` вызван → идёт через прокси → но внутри метода `this` = **оригинальный объект** (не прокси!). Вызов `this.processOne(o)` — прямой вызов метода оригинала, минуя прокси, минуя транзакционную логику.

```
Внешний вызов          Внутренний вызов
──────────►            (this.method)
                        ─────────►
    ┌──────────┐              ┌──────────┐
    │  Proxy   │  ──super─►   │ Original │──►┐
    └──────────┘              └──────────┘   │
        ↑                          │         │
        │                          └─────────┘   ← this.processOne → напрямую!
    call site                      минует прокси
```

### 4.2 Как обойти

**A) Извлечь в другой бин (правильно)**:
```java
@Service
class OrderService {
    @Autowired OrderProcessor processor;

    public void processAll(List<Order> orders) {
        for (Order o : orders) {
            processor.processOne(o);   // ← через прокси processor
        }
    }
}

@Service
class OrderProcessor {
    @Transactional(propagation = REQUIRES_NEW)
    public void processOne(Order o) { ... }
}
```

**B) Инжектить самого себя (некрасиво)**:
```java
@Service
class OrderService {
    @Autowired OrderService self;   // ← прокси

    public void processAll(List<Order> orders) {
        for (Order o : orders) {
            self.processOne(o);       // ← через прокси!
        }
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void processOne(Order o) { ... }
}
```

Внимание: circular dependency в Spring 2.6+ по умолчанию запрещён → `@Lazy`:
```java
@Autowired @Lazy OrderService self;
```

**C) `AopContext.currentProxy()` (страшно)**:
```java
((OrderService) AopContext.currentProxy()).processOne(o);
```

Требует `@EnableAspectJAutoProxy(exposeProxy = true)`. Некрасиво, но работает.

**D) AspectJ compile-time weaving** — модифицирует байткод, все вызовы (внешние + внутренние) идут через advice. Не через прокси. Сложно настроить, редко используется.

### 4.3 Реальный кейс ИСНА

Memory `knp-fo-sync-notification-bugs`: 6 багов, включая `@Transactional` мёртв из-за self-invocation → LazyInit + грязные commits.

Урок: **любой раз когда вижу `this.methodWithAnnotation()` — красная лампа**. Аннотация НЕ работает.

---

## 5. Ограничения @Transactional (кроме self-invocation)

### 5.1 @Transactional на private / protected / package-private

**Не работает** для CGLib-прокси (нельзя переопределить private).

Для `protected` — технически можно, но Spring не сканирует. По контракту — **только public**.

```java
@Transactional
private void x() { ... }   // ← НЕ РАБОТАЕТ, не будет ошибки, но tx не будет
```

Тихо ломается.

### 5.2 @Transactional на final method

CGLib не может override `final`. **Не работает**.

### 5.3 @Transactional на static

Static методы не наследуются. **Не работает**.

### 5.4 @Transactional внутри @PostConstruct

```java
@Component
class InitBean {
    @PostConstruct
    @Transactional
    public void init() { ... }   // ← НЕ работает
}
```

Причина: `@PostConstruct` вызывается **до** того, как бин обёрнут в прокси. Транзакции не будет.

Fix: `ApplicationRunner` или `@EventListener(ContextRefreshedEvent.class)`.

### 5.5 @Transactional на конструкторе

Не поддерживается.

---

## 6. TransactionSynchronizationManager

Ключевой класс, связывающий транзакцию с текущим потоком.

```java
public abstract class TransactionSynchronizationManager {
    private static final ThreadLocal<Map<Object, Object>> resources;
    private static final ThreadLocal<TransactionSynchronization> synchronizations;
    private static final ThreadLocal<String> currentTransactionName;
    private static final ThreadLocal<Integer> currentTransactionIsolationLevel;
    private static final ThreadLocal<Boolean> currentTransactionReadOnly;
    private static final ThreadLocal<Boolean> actualTransactionActive;
}
```

Всё через `ThreadLocal`. Отсюда:

- **Транзакция привязана к потоку** — если ты переключаешь поток (`@Async`, virtual thread, ExecutorService) — теряешь tx.
- Внутри метода: `TransactionSynchronizationManager.isActualTransactionActive()` — проверить есть ли tx.

### 6.1 Resources

Ключевое: **Connection / EntityManager привязаны к потоку через resources**.

```
Thread-local resources map:
  DataSource → ConnectionHolder(Connection)
  EntityManagerFactory → EntityManagerHolder(EntityManager)
```

При `getConnection()` в JDBC template — Spring смотрит thread-local, если есть tx → возвращает связанный Connection. Иначе новый.

Отсюда: все DAO / repository / native SQL внутри одной tx-метода получают **тот же Connection**.

### 6.2 Suspension для REQUIRES_NEW

При REQUIRES_NEW:
1. Spring снимает текущие resources из thread-local (складывает в `SuspendedResources`).
2. Создаёт новую tx с новым Connection.
3. По завершении новой tx → восстанавливает старые resources.

Это требует **два connection одновременно** — легко исчерпать пул.

---

## 7. Жизненный цикл транзакции (детально)

Разберём что происходит от вызова до commit.

### 7.1 Entering @Transactional method

```
Call: proxy.createOrder(order)
    │
    ▼
CglibAopProxy.intercept
    │
    ▼
ReflectiveMethodInvocation.proceed
    │
    ▼
TransactionInterceptor.invoke
    │
    ▼
TransactionAspectSupport.invokeWithinTransaction
    │
    ├─ getTransactionAttribute(method)
    │    → находит @Transactional, создаёт TransactionAttribute
    │
    ├─ determineTransactionManager(txAttr)
    │    → берёт JpaTransactionManager (по имени или default)
    │
    ├─ createTransactionIfNecessary
    │    → txManager.getTransaction(txAttr)
    │       → DataSource.getConnection()
    │       → conn.setAutoCommit(false)
    │       → conn.setTransactionIsolation(txAttr.isolation)
    │       → bind to TransactionSynchronizationManager
    │    → возвращает TransactionInfo
    │
    ▼
Original method execution
```

### 7.2 Original method

Ваш код. Внутри может:
- Использовать JdbcTemplate — берёт связанный Connection.
- Использовать EntityManager — берёт связанный EntityManager.
- Вызвать другой @Transactional метод — участвует в текущей tx (REQUIRED).
- Bросить exception.

### 7.3 Exiting — success

```
Success return
    │
    ▼
TransactionAspectSupport.commitTransactionAfterReturning
    │
    ▼
txManager.commit(status)
    │
    ├─ triggerBeforeCommit()          → Synchronization callbacks
    ├─ doCommit(txStatus)
    │    → em.flush()                  (для JPA)
    │    → conn.commit()
    ├─ triggerAfterCommit()
    ├─ triggerAfterCompletion(COMMITTED)
    │
    └─ cleanupAfterCompletion()
         → conn.close() (returned to pool)
         → unbind from TransactionSynchronizationManager
```

### 7.4 Exiting — exception

```
Exception thrown
    │
    ▼
TransactionAspectSupport.completeTransactionAfterThrowing(txInfo, ex)
    │
    ├─ isRollback = txAttr.rollbackOn(ex)
    │    → RuntimeException / Error → true
    │    → checked → false (по умолчанию)
    │
    ├─ if isRollback:
    │    txManager.rollback(status)
    │      → triggerBeforeCompletion
    │      → doRollback(txStatus) → conn.rollback()
    │      → triggerAfterCompletion(ROLLED_BACK)
    │      → cleanupAfterCompletion
    │
    ├─ else:
    │    txManager.commit(status)      ← коммитит несмотря на exception!
    │
    ▼
Rethrow exception
```

**Важный кавет rollbackFor**: см. следующий файл.

---

## 8. @Transactional на interface vs class

```java
public interface OrderService {
    @Transactional
    void createOrder(Order o);
}

@Service
public class OrderServiceImpl implements OrderService {
    public void createOrder(Order o) { ... }
}
```

**Работает для JDK dynamic proxy**, потому что прокси видит аннотацию на интерфейсе.

**Не всегда работает для CGLib**: если у бина нет интерфейсов — CGLib смотрит только на класс. Если проксирование через CGLib и аннотация только на interface — может не подхватить.

**Правило**: ставь `@Transactional` **на реализацию** (класс), не на интерфейс. Работает всегда.

---

## 9. Множественные PlatformTransactionManager

Что если у тебя несколько БД или JMS + БД?

Каждый TransactionManager управляет **одним ресурсом**. Для нескольких:

### 9.1 Multiple @Bean

```java
@Bean("primaryTx")
DataSourceTransactionManager primaryTx(@Qualifier("primaryDs") DataSource ds) {
    return new DataSourceTransactionManager(ds);
}

@Bean("secondaryTx")
DataSourceTransactionManager secondaryTx(@Qualifier("secondaryDs") DataSource ds) {
    return new DataSourceTransactionManager(ds);
}
```

Использование:
```java
@Transactional("primaryTx")
public void save(...) { ... }

@Transactional("secondaryTx")
public void saveElsewhere(...) { ... }
```

### 9.2 ChainedTransactionManager (deprecated)

«Best-effort» для нескольких — начинает все параллельно, коммитит по цепочке. **Не 2PC** — атомарность не гарантируется.

```java
@Bean
ChainedTransactionManager chainedTx(
        PlatformTransactionManager jpa, PlatformTransactionManager rabbit) {
    return new ChainedTransactionManager(jpa, rabbit);
}
```

Deprecated с 3.0 — правильно **outbox pattern** или **JTA/XA**.

### 9.3 JTA / XA

Настоящий 2PC. Требует JTA-провайдера (Atomikos, Narayana) — сложно в Spring Boot.

Для микросервисов — избегай, используй Saga / Outbox (см. `32-transactions-acid-isolation-propagation.md`).

---

## 10. Программные транзакции (не через @Transactional)

Иногда нужен более гибкий контроль:

### 10.1 TransactionTemplate

```java
@Autowired PlatformTransactionManager txManager;

TransactionTemplate tx = new TransactionTemplate(txManager);
tx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
tx.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);

tx.execute(status -> {
    repo.save(order);
    if (someCondition) {
        status.setRollbackOnly();    // явный rollback без exception
    }
    return null;
});
```

Использование:
- Гранулярный контроль.
- Динамические propagation/isolation.
- `setRollbackOnly()` без бросания exception.

### 10.2 Ручной PlatformTransactionManager

```java
DefaultTransactionDefinition def = new DefaultTransactionDefinition();
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

TransactionStatus status = txManager.getTransaction(def);
try {
    // work
    txManager.commit(status);
} catch (Exception e) {
    txManager.rollback(status);
    throw e;
}
```

Мало где нужен. Обычно `TransactionTemplate` достаточно.

---

## 11. `@Transactional` метаданные (все поля)

```java
@Transactional(
    value = "",                          // qualifier для TransactionManager
    transactionManager = "",             // явное имя (алиас для value)
    propagation = Propagation.REQUIRED,
    isolation = Isolation.DEFAULT,
    timeout = -1,                        // секунды, -1 = default (обычно ∞)
    timeoutString = "",
    readOnly = false,
    rollbackFor = {},                    // Class<? extends Throwable>[]
    rollbackForClassName = {},
    noRollbackFor = {},
    noRollbackForClassName = {},
    label = {}                           // маркеры для custom advice
)
```

`readOnly`, `rollbackFor` — детально в `34-transactional-advanced.md`.

---

## 12. Как понять что @Transactional работает

### 12.1 Логирование

```yaml
logging.level:
  org.springframework.transaction: TRACE
  org.springframework.transaction.interceptor: TRACE
  org.springframework.orm.jpa: TRACE
```

Увидишь:
```
Getting transaction for [com.example.OrderService.createOrder]
Creating new transaction with name [...]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
Opened new EntityManager [...] for JPA transaction
Beginning JPA transaction on [...]
Committing JPA transaction on ...
```

Если такого лога **нет** для твоего метода — Spring не оборачивает его. Причины:
- Self-invocation.
- `private` / `final`.
- Забыл `@EnableTransactionManagement` (в Boot включён по умолчанию).
- Метод не на прокси (например, вызван через `AopUtils.getTargetClass`).

### 12.2 Проверить прокси

```java
System.out.println(orderService.getClass());
// Обычный: class com.example.OrderService
// Прокси: class com.example.OrderService$$SpringCGLIB$$0
```

Если не прокси — Spring вообще не обернул. Скорее всего нет @Transactional или бина не в контексте.

---

## 13. Собесные вопросы

1. **Как работает @Transactional?** — AOP-прокси (CGLib) оборачивает метод в begin/commit/rollback.
2. **JDK dynamic proxy vs CGLib?** — JDK для интерфейсов; CGLib создаёт subclass (default в Spring Boot).
3. **Self-invocation — почему @Transactional не работает?** — `this.method()` минует прокси, вызывает оригинал напрямую.
4. **Как обойти self-invocation?** — Вытащить в другой бин / инжектить self через @Lazy / AopContext.currentProxy().
5. **@Transactional на private?** — Не работает; CGLib не может override private.
6. **@Transactional на final метод?** — Не работает.
7. **@Transactional в @PostConstruct?** — Не работает; @PostConstruct до создания прокси.
8. **Что такое TransactionInterceptor?** — AOP-совет, оборачивает `@Transactional`-методы.
9. **Что такое PlatformTransactionManager?** — Абстракция над механизмом tx; JpaTransactionManager / DataSourceTransactionManager / JtaTransactionManager.
10. **Как транзакция привязывается к потоку?** — Через `TransactionSynchronizationManager` — ThreadLocal с Connection/EntityManager.
11. **@Transactional на interface или class?** — Ставь на реализацию (класс), работает всегда.
12. **Как проверить что @Transactional работает?** — Логи `org.springframework.transaction: TRACE` или проверить `getClass()` на CGLib-суффикс.
13. **TransactionTemplate — когда?** — Программный контроль tx, динамические настройки, `setRollbackOnly` без exception.
14. **Multiple TransactionManager — как?** — Named @Bean + `@Transactional("txName")`.
15. **ChainedTransactionManager — что и почему deprecated?** — Best-effort tx для нескольких ресурсов без 2PC; ненадёжно, используй Saga/Outbox.

---

## Итог

- **@Transactional** = **CGLib-прокси** оборачивает public-метод.
- **TransactionInterceptor** → **TransactionAspectSupport.invokeWithinTransaction()** → **PlatformTransactionManager**.
- **JpaTransactionManager** для JPA (default в Spring Boot).
- **TransactionSynchronizationManager** привязывает tx к потоку через ThreadLocal (Connection / EntityManager).
- **Self-invocation** = самая частая ошибка; прокси перехватывает только внешние вызовы.
- **Private / final / static / @PostConstruct** — не работает.
- **TransactionTemplate** для программного контроля.

Следующий — `34-transactional-advanced.md`.
