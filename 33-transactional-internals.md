# 33. @Transactional изнутри: прокси и PlatformTransactionManager

## Общая картина

Когда разработчик пишет простую аннотацию:
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

За кулисами Spring делает довольно сложную работу. При создании bean OrderService Spring оборачивает его в CGLib-прокси (subclass). Прокси-класс переопределяет метод createOrder добавляя вокруг оригинальной логики код управления транзакцией — beginTransaction, вызов оригинала, commit при успехе или rollback при exception. Везде где используется @Autowired OrderService возвращается прокси не оригинальный объект.

Понимание этого механизма критически важно потому что multiple классические баги @Transactional происходят именно из ограничений proxy подхода. Разберём каждый компонент детально.

## Три ключевых участника

TransactionInterceptor это AOP-совет (advice) выполняющийся вокруг каждого @Transactional-метода. Класс org.springframework.transaction.interceptor.TransactionInterceptor реализует MethodInterceptor интерфейс. Внутри TransactionAspectSupport.invokeWithinTransaction главный метод выполняющий последовательность. Извлекает TransactionAttribute (propagation, isolation, rollbackFor). Берёт нужный PlatformTransactionManager. Открывает транзакцию через getTransaction. Вызывает оригинальный метод. При success — commit. При exception — проверяет rollback rules и либо rollback либо commit.

TransactionAttributeSource читает мета-информацию о transaction для метода. Основная реализация AnnotationTransactionAttributeSource ищет @Transactional аннотацию сначала на методе, потом на классе, потом на interface. Кэширует результат в Map Method to TransactionAttribute чтобы не парсить аннотации на каждый вызов.

PlatformTransactionManager это абстракция над конкретным механизмом transactions. Interface с тремя методами. getTransaction принимает TransactionDefinition и возвращает TransactionStatus представляющий open transaction. commit фиксирует изменения. rollback откатывает.

Реализации PlatformTransactionManager для разных сценариев. DataSourceTransactionManager для plain JDBC — работает с DataSource напрямую. JpaTransactionManager для JPA plus Hibernate — управляет EntityManager plus underlying JDBC connection. JtaTransactionManager для XA distributed transactions — работает через JTA API. ChainedTransactionManager для best-effort объединения нескольких (deprecated). RabbitTransactionManager для Rabbit. KafkaTransactionManager для Kafka.

Spring Boot автоконфигурит JpaTransactionManager если есть spring-boot-starter-data-jpa dependency. Это самый частый случай в enterprise Spring приложениях.

## Как создаётся прокси

InfrastructureAdvisorAutoProxyCreator это BeanPostProcessor следящий за созданием beans. Метод postProcessAfterInitialization проверяет — «этому bean нужен прокси?». Если bean имеет метод с @Transactional (напрямую или через классовую аннотацию) — да, оборачивается в прокси.

Выбор типа прокси. JDK Dynamic Proxy используется если bean реализует interfaces — прокси реализует те же interfaces. CGLib Proxy используется если bean не имеет interfaces — создаётся subclass через bytecode generation.

Настройка через свойство:
```yaml
spring.aop.proxy-target-class: true       # всегда CGLib (default в Spring Boot)
```

По default Spring Boot использует CGLib даже когда bean имеет interfaces. Причина — CGLib subclass proxy работает предсказуемо независимо от interface implementation. JDK proxy может создать subtle issues когда bean инжектится по class type а не interface type.

Структура CGLib прокси:
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

Прокси имеет тот же тип что оригинал (subclass) — можно инжектить как OrderService в других beans.

Ограничения CGLib подхода. Не может проксировать final классы потому что нельзя extend. Не может final методы потому что нельзя override. Не может private методы потому что не переопределяются в subclass. Не может static методы потому что не наследуются. Требует no-arg constructor или пустой super() потому что CGLib создаёт instance прокси который extends оригинал.

Различия JDK Dynamic Proxy и CGLib. JDK требует interface, CGLib требует extendable class. JDK ограничения только методами interface, CGLib — final/private/static не работают. JDK немного быстрее в invocation, CGLib немного медленнее из-за bytecode dispatch. Default в Spring Boot — CGLib потому что более predictable behavior.

## Self-invocation классический баг

Наиболее частая ошибка при работе с @Transactional. Проявляется когда метод внутри класса вызывает другой метод того же класса с @Transactional:
```java
@Service
class OrderService {
    public void processAll(List<Order> orders) {
        for (Order o : orders) {
            this.processOne(o);      // ← вызов через this
        }
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void processOne(Order o) {
        repo.save(o);
    }
}
```

Ожидаемое поведение — processOne создаёт новую транзакцию на каждой итерации. Реальность — @Transactional не срабатывает. Всё выполняется в одной transaction (или без transaction вообще).

Причина в природе proxy. Прокси перехватывает external calls — вызовы через reference на прокси. Когда processAll вызывается извне — идёт через прокси. Внутри метода this равно оригинальному объекту не прокси. Вызов this.processOne напрямую вызывает метод оригинала минуя прокси и минуя transactional advice.

```
Внешний вызов          Внутренний вызов
──────────►            (this.method)
                        ─────────►
    ┌──────────┐              ┌──────────┐
    │  Proxy   │  ──super─►   │ Original │──►┐
    └──────────┘              └──────────┘   │
        ↑                          │         │
        │                          └─────────┘   ← this.processOne напрямую!
    call site                      минует прокси
```

Способы обойти self-invocation. Первый и правильный — извлечь вызываемый метод в другой bean:
```java
@Service
class OrderService {
    @Autowired OrderProcessor processor;

    public void processAll(List<Order> orders) {
        for (Order o : orders) {
            processor.processOne(o);   // через прокси processor
        }
    }
}

@Service
class OrderProcessor {
    @Transactional(propagation = REQUIRES_NEW)
    public void processOne(Order o) { ... }
}
```

Разделение responsibilities плюс обход self-invocation одним архитектурным решением.

Второй способ — инжектить bean самого себя. Некрасиво но работает:
```java
@Service
class OrderService {
    @Autowired OrderService self;   // прокси через DI

    public void processAll(List<Order> orders) {
        for (Order o : orders) {
            self.processOne(o);       // через прокси
        }
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void processOne(Order o) { ... }
}
```

Caveat — circular dependency в Spring 2.6+ по default запрещён. Требуется @Lazy для self reference. Считается code smell — extract to another class обычно чище.

Третий способ — AopContext.currentProxy(). Ещё более wtf-style но иногда встречается:
```java
((OrderService) AopContext.currentProxy()).processOne(o);
```

Требует @EnableAspectJAutoProxy(exposeProxy = true). Работает но обычно проще extract to another class.

Четвёртый способ — AspectJ compile-time weaving. Модифицирует bytecode на этапе compilation — все вызовы (external plus internal) идут через advice. Не через прокси. Мощнее но требует Maven/Gradle plugin, специальные IDE setup, extra complexity. Редко используется в enterprise.

Реальный кейс из КНП memory knp-fo-sync-notification-bugs — 6 багов включая @Transactional мёртв из-за self-invocation. Приводил к LazyInit и «грязным» commits. Урок универсальный — любой раз когда видишь this.methodWithAnnotation() внутри класса это красная лампа. Аннотация НЕ работает. Fix через extract to another bean.

## Другие ограничения @Transactional

Помимо self-invocation существуют другие сценарии когда @Transactional silently не работает.

@Transactional на private methods не срабатывает. CGLib не может override private методы в subclass — они не visible. Не будет ошибки, просто transaction не создаётся. Тихо ломается:
```java
@Transactional
private void x() { ... }   // НЕ работает, no error
```

@Transactional на protected или package-private технически возможен для CGLib но Spring не сканирует их. По контракту только public. Правило — @Transactional только на public методах.

@Transactional на final method. CGLib не может override final. Не работает. Ошибки может не быть, просто silent failure.

@Transactional на static методе. Static методы не наследуются subclass. Не работает.

@Transactional на @PostConstruct метод не работает:
```java
@Component
class InitBean {
    @PostConstruct
    @Transactional
    public void init() { ... }   // НЕ работает
}
```

Причина — @PostConstruct вызывается до того как bean обёрнут в прокси. На момент вызова proxy ещё не существует, transaction не создаётся. Fix через ApplicationRunner или @EventListener(ContextRefreshedEvent.class) для запуска init logic после completion context setup.

@Transactional на конструкторе не поддерживается вообще. Constructor вызывается создание bean — proxy может быть только вокруг methods не constructor.

## TransactionSynchronizationManager

Ключевой класс связывающий transaction с текущим потоком. Всё через ThreadLocal:
```java
public abstract class TransactionSynchronizationManager {
    private static final ThreadLocal<Map<Object, Object>> resources;
    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations;
    private static final ThreadLocal<String> currentTransactionName;
    private static final ThreadLocal<Integer> currentTransactionIsolationLevel;
    private static final ThreadLocal<Boolean> currentTransactionReadOnly;
    private static final ThreadLocal<Boolean> actualTransactionActive;
}
```

Транзакция привязана к потоку через thread-local storage. Отсюда критическое следствие — при переключении потока (@Async, virtual thread, ExecutorService) транзакция теряется. Изменения не будут в контексте новой transaction.

Внутри метода можно проверить активна ли транзакция:
```java
TransactionSynchronizationManager.isActualTransactionActive()
```

Resources хранят per-transaction ресурсы. Ключевое — Connection и EntityManager привязаны к потоку через resources:
```
Thread-local resources map:
  DataSource → ConnectionHolder(Connection)
  EntityManagerFactory → EntityManagerHolder(EntityManager)
```

При getConnection() в JdbcTemplate Spring смотрит thread-local — если есть транзакция возвращает связанный Connection. Иначе создаёт новый. Все DAO и repository и native SQL внутри одного transactional метода получают тот же Connection автоматически.

Suspension для REQUIRES_NEW работает через store и restore resources. При начале REQUIRES_NEW Spring снимает текущие resources из thread-local (складывает в SuspendedResources). Создаёт новую transaction с новым Connection. По завершении новой transaction — восстанавливает старые resources в thread-local.

Это требует два connection одновременно — один для suspended outer transaction, второй для inner REQUIRES_NEW. При массовом использовании легко исчерпать pool.

## Жизненный цикл транзакции

Разберём что происходит от вызова до commit подробно.

Entering @Transactional method:
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

Original method — ваш код. Внутри может использовать JdbcTemplate — берёт связанный Connection из thread-local. Использовать EntityManager — берёт связанный EntityManager. Вызвать другой @Transactional метод — участвует в текущей транзакции (REQUIRED default). Бросить exception — обрабатывается в exiting phase.

Exiting при success:
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

Exiting при exception:
```
Exception thrown
    │
    ▼
TransactionAspectSupport.completeTransactionAfterThrowing(txInfo, ex)
    │
    ├─ isRollback = txAttr.rollbackOn(ex)
    │    → RuntimeException / Error → true (default)
    │    → checked exception → false (default!)
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

Критический caveat — по default checked exceptions приводят к COMMIT транзакции даже если exception пробрасывается наверх. Runtime exceptions приводят к rollback. Настраивается через rollbackFor атрибут. Детали в следующем файле.

## @Transactional на interface vs class

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

Работает для JDK dynamic proxy потому что прокси видит аннотацию на interface. Не всегда работает для CGLib — если bean не имеет interfaces CGLib смотрит только на класс. Если проксирование через CGLib и аннотация только на interface может не подхватить.

Правило универсальное — ставь @Transactional на реализацию (class) не на interface. Работает независимо от типа proxy. Единственное правильное место — на public method concrete class или на класс целиком.

## Множественные PlatformTransactionManager

Что если у приложения несколько БД или JMS плюс БД? Каждый TransactionManager управляет одним ресурсом.

Для нескольких — multiple @Bean с named:
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

Использование — указать нужный:
```java
@Transactional("primaryTx")
public void save(...) { ... }

@Transactional("secondaryTx")
public void saveElsewhere(...) { ... }
```

Каждая транзакция работает с своим resource independently. Атомарность через ресурсы не гарантируется — если primary commit прошёл а secondary упал, primary остаётся committed.

ChainedTransactionManager был попыткой solve этой проблемы. Best-effort объединение нескольких TransactionManagers — начинает все параллельно, коммитит по цепочке. Не 2PC — атомарность не гарантируется в full sense. Deprecated с Spring 3.0. Правильное решение — outbox pattern или JTA/XA.

JTA/XA — настоящий 2PC через JTA provider (Atomikos, Narayana). Сложная настройка в Spring Boot. Для микросервисов избегай — используй Saga или Outbox.

## Программные транзакции

Иногда нужен более гибкий контроль чем даёт @Transactional. Программные API дают dynamic control.

TransactionTemplate это удобная обёртка:
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

Гранулярный контроль per-invocation. Dynamic propagation и isolation. setRollbackOnly без бросания exception когда нужен just rollback.

Ручной PlatformTransactionManager для полного low-level control:
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

Мало где нужен — TransactionTemplate обычно достаточен и проще.

## @Transactional атрибуты полностью

Все возможные атрибуты @Transactional:
```java
@Transactional(
    value = "",                          // qualifier для TransactionManager
    transactionManager = "",             // явное имя (alias для value)
    propagation = Propagation.REQUIRED,
    isolation = Isolation.DEFAULT,
    timeout = -1,                        // секунды, -1 = default (обычно бесконечно)
    timeoutString = "",
    readOnly = false,
    rollbackFor = {},                    // Class<? extends Throwable>[]
    rollbackForClassName = {},
    noRollbackFor = {},
    noRollbackForClassName = {},
    label = {}                           // маркеры для custom advice
)
```

Детали readOnly, rollbackFor, timeout — в файле 34-transactional-advanced. Здесь важно знать что все они существуют и могут настраиваться.

## Как проверить что @Transactional работает

Логирование через TRACE level Spring transaction package:
```yaml
logging.level:
  org.springframework.transaction: TRACE
  org.springframework.transaction.interceptor: TRACE
  org.springframework.orm.jpa: TRACE
```

При работающих транзакциях увидишь логи:
```
Getting transaction for [com.example.OrderService.createOrder]
Creating new transaction with name [...]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
Opened new EntityManager [...] for JPA transaction
Beginning JPA transaction on [...]
Committing JPA transaction on ...
```

Если такого лога нет для твоего метода — Spring не оборачивает его. Причины стандартные. Self-invocation. private или final метод. Забыт @EnableTransactionManagement (в Boot включён по default). Метод вызван не через proxy — например через AopUtils.getTargetClass или reflection.

Проверка прокси на runtime:
```java
System.out.println(orderService.getClass());
// Обычный класс: class com.example.OrderService
// Прокси: class com.example.OrderService$$SpringCGLIB$$0
```

CGLIB суффикс в class name — прокси есть. Обычный class без суффикса — Spring не обернул. Скорее всего нет @Transactional или bean не в контексте (создан через new вместо DI).

## Итоги

@Transactional это AOP-прокси (CGLib default) оборачивающий public метод в begin/commit/rollback logic. Прозрачно для application code но подчиняется ограничениям proxy подхода.

TransactionInterceptor это AOP advice. TransactionAspectSupport.invokeWithinTransaction главный метод. PlatformTransactionManager абстракция над transaction mechanism. JpaTransactionManager default в Spring Boot для JPA приложений.

Self-invocation классический баг. this.method внутри класса минует proxy и вызывает оригинал напрямую. Аннотация не срабатывает. Fix через extract to another bean обычно правильный подход.

Другие ограничения @Transactional. private, final, static, @PostConstruct — не работают. Правило только public methods concrete classes.

TransactionSynchronizationManager связывает transaction с потоком через ThreadLocal. Connection и EntityManager привязаны к thread. @Async и virtual threads не пробрасывают transaction автоматически.

Жизненный цикл — прокси intercept, TransactionInterceptor invoke, PlatformTransactionManager begin/commit/rollback, cleanup resources. Detailed flow важен для troubleshooting.

Multiple TransactionManagers через named beans plus @Transactional("name"). ChainedTransactionManager deprecated. JTA для настоящего 2PC редко используется.

TransactionTemplate для программного контроля когда декларативный @Transactional недостаточно. TransactionAspectSupport.currentTransactionStatus.setRollbackOnly для forceful rollback без exception.

Логи org.springframework.transaction TRACE plus проверка class name на CGLIB suffix — стандартный debugging arsenal.

Дальше — advanced аспекты @Transactional включая rollback rules, readOnly, TransactionalEventListener, savepoints и тестирование.
