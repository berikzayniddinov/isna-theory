# 35. @Transactional и JPA: PersistenceContext, flush timing, LazyInit, OSIV

## Куда мы идём и зачем

В двух предыдущих файлах разобрали @Transactional infrastructure — CGLib proxies, TransactionInterceptor, PlatformTransactionManager, TransactionSynchronizationManager. Разобрали advanced aspects — rollback rules, listeners, savepoints, testing. Всё это applicable для plain JDBC приложений через DataSourceTransactionManager. Но JPA plus Hibernate добавляют ещё один слой абстракции — PersistenceContext который сам по себе complex system со своими flushing rules, cache mechanisms, lifecycle events.

Разница между разработчиком «работающим с JPA» и «понимающим JPA» очень заметна именно здесь. Первый пишет entities plus repositories и надеется. Второй знает что PersistenceContext это first-level cache with identity map — same entity instance returned regardless of how loaded, знает что flush timing зависит от FlushMode plus operation type, знает что LazyInitializationException происходит потому что Hibernate proxy tries to load associated data after session closed, знает что OSIV filter opens session за границами transaction создавая множество subtle problems.

В этом файле разберём именно эту JPA-specific механику. Как связываются JPA transaction и PersistenceContext. Как SharedEntityManagerCreator работает и почему @PersistenceContext injection даёт proxy а не real EntityManager. Как отличаются TRANSACTION_SCOPED vs EXTENDED PersistenceContexts. Как flush работает — automatic triggers, query-time flush, explicit em.flush. FlushMode implications и readOnly interaction. LazyInit deep dive — proxy generation Hibernate, session lifecycle, что реально пытается load. Hibernate 6 changes стрiктность. Cascade и его interaction с transactions. Bulk operations и почему они минуют PersistenceContext. OSIV filter internals plus tradeoffs. Долгие транзакции и practical impacts.

## Ключевое связывание: transaction равно PersistenceContext равно Connection

При старте @Transactional method в JPA приложении Spring выполняет каскад operations соединяющих несколько abstractions в one atomic unit.

Первый шаг — JpaTransactionManager.doBegin инвестируется через TransactionAspectSupport.createTransactionIfNecessary. Внутри doBegin создаётся JpaTransactionObject который будет хранить всё state транзакции. Проверяется isExistingTransaction — если да и propagation позволяет, используется existing (REQUIRED). Если нет и propagation требует — создаётся новая.

Второй шаг — EntityManager обеспечение. Через entityManagerFactory.createEntityManager() создаётся новый JPA EntityManager. Это heavyweight object в Hibernate — allocates internal PersistenceContext cache, prepares statement caches, sets up interceptors. Обычно один EntityManager используется на всю transaction. Some frameworks pool EntityManagers но Hibernate типично creates fresh для каждой transaction.

Третий шаг — начало JPA transaction. em.getTransaction().begin() marks EntityManager as being in transactional state. Внутри Hibernate это switch flush mode к AUTO (from optional MANUAL for pre-transaction state), initialize dirty checking snapshots как entities load, enable various transactional interceptors.

Четвёртый шаг — получение underlying Connection. Через em.unwrap(Connection.class) или equivalent Hibernate internal API извлекается JDBC Connection. Applied все JDBC settings — setAutoCommit(false), setTransactionIsolation если specified, setReadOnly если readOnly transaction.

Пятый шаг — resource binding. TransactionSynchronizationManager.bindResource связывает EntityManagerHolder wrapping этот EntityManager с EntityManagerFactory как key. Также bindResource связывает ConnectionHolder wrapping Connection с DataSource как key. Это устанавливает thread-local mapping — везде в коде где потом запрашивается EntityManager для этого EntityManagerFactory или Connection для этого DataSource, будут возвращены same instances связанные с current transaction.

Complete flow visualized:
```
@Transactional method
    │
    ▼
JpaTransactionManager.doBegin
    │
    ├─ EntityManagerFactory.createEntityManager()
    ├─ em.getTransaction().begin()
    ├─ Извлечь Connection из EM
    ├─ conn.setAutoCommit(false)
    ├─ conn.setTransactionIsolation (если задан)
    ├─ Bind EntityManagerHolder в TSM для EMF
    ├─ Bind ConnectionHolder в TSM для DataSource
    │
    ▼
Метод выполняется
    │
    │  em.persist(x) → добавляет в PersistenceContext
    │  em.find(...) → PersistenceContext cache
    │  setter на managed → dirty marked
    ▼
JpaTransactionManager.doCommit
    │
    ├─ em.flush()               → SQL: INSERT/UPDATE/DELETE
    ├─ em.getTransaction().commit()  → JDBC commit
    ├─ em.close()
    └─ unbind from TSM
```

Отсюда fundamental invariant. Внутри одного @Transactional метода — one EntityManager, one PersistenceContext, one Connection. Всё operations разделяют этот shared context. Изменения одной сущности видны везде в этом контексте немедленно. External changes невидимы (isolation semantics).

## SharedEntityManagerCreator: тонкий трюк с injection

Когда Spring injects EntityManager через @PersistenceContext, что реально получает developer? Не настоящий EntityManager а прокси. Класс SharedEntityManagerCreator в spring-orm module создаёт этот прокси через JDK dynamic proxy механизм — прокси implements EntityManager interface.

Реализация прокси through InvocationHandler. Каждый method call на прокси intercepted. Handler lookup в TransactionSynchronizationManager для EntityManagerHolder. Если found (внутри transaction) — extract associated EntityManager и invoke method на нём. Если not found (вне transaction) — new EntityManager создаётся для single call, method invoked, EntityManager closed. Второй case rare — обычно используется transactional context.

Почему такая архитектура. Enables singleton scope injection. EntityManager сам по себе НЕ thread-safe — instance должен быть per-thread или per-transaction. Если бы injected direct EntityManager — could не be singleton bean. Проще сделать proxy которая routes к correct EntityManager based на current thread state.

Также enables scoping flexibility. TRANSACTION_SCOPED (default) — EntityManager tied к transaction, closed на commit/rollback. EXTENDED — EntityManager survives across transactions (rare, mostly для conversations). Proxy transparent для business code.

Реализация scoping — TransactionScopedEntityManagerHolder vs regular ResourceHolder. TSM tracks with transaction. Extended tracked differently. Business code sees same EntityManager interface — implementation hidden.

Практическое последствие. Вне @Transactional operations с EntityManager могут не работать as expected. При отсутствии transaction proxy создаёт temporary EntityManager per call. em.persist requires transaction — throws TransactionRequiredException. em.find works но returns detached entity — no cache, no dirty tracking. Разные calls через proxy возвращают different EntityManagers — consistency между calls broken.

Практика — все operations с EntityManager должны быть внутри @Transactional. Explicit boundaries. Никаких «случайно» работающих calls outside transaction.

## Flush timing: когда SQL реально идёт в БД

Внутри @Transactional-метода Hibernate не выполняет SQL немедленно на каждое persist/setter/remove. Изменения накапливаются в PersistenceContext как «pending changes». Реальный SQL flush в БД происходит только в специфических моменты.

Automatic flush в FlushMode.AUTO (default). Первый trigger — перед commit транзакции. Guaranteed — все pending changes должны быть persisted перед physical commit. JpaTransactionManager.doCommit invokes em.flush before conn.commit.

Второй trigger — перед query execution. Hibernate assumes что query может depend на pending changes. Если Order updated к status=NEW но не flushed, а потом SELECT count WHERE status=NEW — count должен include this Order. Отсюда flush перед query чтобы query видел consistent state.

Третий trigger — иногда перед find(). Не гарантированно, но Hibernate может flush если знает что modified entity fetched. Complexity rules — not always predictable. Practically — usually flush happens при query but не при simple id-based find.

Пример поведения:
```java
@Transactional
public void updateAndCount() {
    Order o = em.find(Order.class, 1L);
    o.setStatus(NEW);              // dirty, не в БД пока

    long count = em.createQuery(
        "SELECT count(o) FROM Order o WHERE o.status = :s", Long.class)
        .setParameter("s", NEW)
        .getSingleResult();
    // ← перед этим query flush → UPDATE + SELECT
    // count увидит текущий изменённый Order

    // на выходе flush (уже сделан), commit
}
```

Manual flush через em.flush() полезен в нескольких сценариях. Проверить что INSERT прошёл валидацию БД до конца method — если constraint violation, exception возникает immediately а не на commit. Получить @GeneratedValue(IDENTITY) id немедленно — id assigned БД только при INSERT, до flush id null. Force ordering — если нужно guarantee что INSERT before subsequent operation.

FlushMode вариации. AUTO — automatic flush per rules описанным выше (default). COMMIT — flush только на transaction commit. Queries могут вернуть stale data within transaction — pending changes не visible in queries. MANUAL — flush только explicit em.flush(). Настраивается для read-only scenarios для performance.

FlushMode.MANUAL activated automatically при @Transactional(readOnly = true). Hibernate implementation — Session.setDefaultReadOnly(true) plus Session.setFlushMode(FlushMode.MANUAL). Rationale — read-only method не должен modify data, no flush needed.

Critical caveat readOnly plus flush interaction. Если в read-only method случайно entity.setField(...) — изменение помечается dirty в PersistenceContext. Но flush не выполняется (MANUAL mode). Changes silently lost. Никакого exception, никаких warnings. Может быть источник sneaky bugs если readOnly установлен на method который эволюционировал to include modifications.

Более фундаментально, readOnly hint. Some frameworks (Spring Data plus Hibernate 6) enforce более strict — attempts to modify managed entity бросают exception. Behavior depends на version и configuration. Awareness always necessary.

## LazyInit: proxy generation plus session lifecycle

Уже обсуждали в файле 15 в контексте JPA performance. Здесь связь с transaction management deep dive.

Начнём с того, что физически представляет собой lazy relation. При маппинге:
```java
@Entity
class Order {
    @OneToMany(mappedBy="order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
    // ...
}
```

Hibernate generation создаёт proxy для items collection. Не настоящий ArrayList — специальный PersistentBag (Hibernate implementation) implementing List interface. Все methods intercepted. При first access — proxy triggers loading из БД через session, replacing internal state с actual data. Subsequent accesses use loaded data.

Ключевой момент — proxy needs live session для loading. Session это Hibernate abstraction над EntityManager — они one-to-one mapped. При закрытии EntityManager (после transaction end) — associated Session закрывается. Proxy references к closed session — invalid.

Сценарий producing LazyInit:
```java
@Transactional(readOnly = true)
public Order load(Long id) {
    return repo.findById(id).orElseThrow();
}

// controller:
Order o = svc.load(1L);
o.getItems().forEach(...);         // ← LazyInit!
```

Что happens step-by-step. svc.load starts @Transactional method. EntityManager created, transaction begins. findById triggers SELECT of Order without items. items PersistentBag created as proxy (empty state). Order returned из method. Transaction ends — EntityManager closed, Session closed. items proxy references к now-closed session.

Возврат в controller. o.getItems returns items proxy. forEach starts iteration. First iteration triggers proxy loading. Proxy tries to use session — session closed — LazyInitializationException thrown.

Решения различаются по quality.

Первое и правильное — DTO в service. Загружаем всё нужное внутри transaction:
```java
@Transactional(readOnly = true)
public OrderDto load(Long id) {
    Order o = repo.findById(id).orElseThrow();
    return new OrderDto(
        o.getId(),
        o.getItems().stream().map(...).toList()   // загружаем внутри transaction
    );
}
```

DTO конструируется внутри transaction где session активна. LAZY access works — trigger loading proxy успевает в active session. Наружу возвращается plain DTO без Hibernate proxies. No LazyInit possible.

Второе — JOIN FETCH или @EntityGraph. Force eager loading в query:
```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);
```

Один query loads Order plus все items. По возврату из transaction items уже loaded. Access outside transaction OK. Но Entity still leaks наружу — architectural issue но not runtime error.

Третье — OSIV. Session до конца HTTP request. Обсуждается ниже — часто скрывает problems чем solves.

Четвёртое — FetchType.EAGER на relation. Не рекомендуется потому что excessive queries — все associations always loaded даже когда не нужны. Performance disaster on scale.

Hibernate 6 стало значительно строже с LazyInit. Некоторые cases который в Hibernate 5 работали (например если entity уже в session cache) в Hibernate 6 throw exception. Semantics более consistent но breaks код который relied на old lenient behavior.

Реальный кейс из КНП memory knp-fo-sync-notification-bugs. @Transactional был мёртв из-за self-invocation (см. файл 33). Значит session никогда не открывалась. В Hibernate 5 некоторые operations всё ещё работали через various fallbacks. В Hibernate 6 стал жёстче — LazyInit проявился в самых unexpected places. Урок — не полагаться на «работает в Hibernate 5». Правильные transactions plus DTOs обязательны.

getById против findById в Spring Data. findById(id) возвращает Optional<T> и делает SELECT immediately. Entity fully loaded (except LAZY associations). getReferenceById(id) возвращает T как Hibernate proxy — no SELECT immediately. Proxy loads at first access — если outside transaction это LazyInit.

Практика — findById по default. getReferenceById только когда «нужна ссылка не читая». Классический use case — установка FK relation где нужен reference не data. Instead of full SELECT of related entity — proxy sufficient потому что only id used для FK.

## Cascade и transaction

Cascade расширяет операции persist/remove на связанные объекты все в одной transaction:
```java
@Entity
class Order {
    @OneToMany(mappedBy="order", cascade = ALL, orphanRemoval = true)
    private List<OrderItem> items;
    // ...
}

@Transactional
public void createOrder(Order o) {
    o.setItems(List.of(item1, item2, item3));
    repo.save(o);
    // на flush → INSERT Order + 3 × INSERT OrderItem — одна transaction
}
```

Механизм cascade internal. При persist(order) Hibernate traverses entity graph checking cascade types на каждой relation. CascadeType.PERSIST — persist также apply к referenced entities. CascadeType.REMOVE — remove propagates. CascadeType.ALL — все operations propagate. Cascade sensitive к direction — bidirectional relations require careful thought about ownership.

Все operations happen в one PersistenceContext, one transaction. Если что-то fails на flush (например items constraint violation) — rollback всей transaction. No «half-saved» states. Atomicity guaranteed.

orphanRemoval специальная semantic. При удалении элемента из parent collection — auto-delete child entity. Sensitive к object identity. Работает через special dirty tracking в PersistenceContext.

## Bulk operations и cache invalidation

Bulk UPDATE и DELETE операции имеют важную особенность — они минуют PersistenceContext.
```java
@Modifying
@Query("UPDATE Order o SET o.status = :new WHERE o.status = :old")
int bulkUpdate(...);
```

Regular repo.save или entity.setter goes through PersistenceContext — Hibernate tracks changes, dirty checking, generates SQL при flush. Bulk operations работают напрямую с БД — Hibernate sends UPDATE SQL immediately, БД executes, changes visible immediately.

Проблема — managed entities в текущем PersistenceContext не знают про bulk operation. Cache становится stale. Query после bulk может return incorrect data потому что PersistenceContext holds old versions.

Пример проблемы:
```java
@Transactional
public void statusUpdate() {
    Order o = em.find(Order.class, 1L);  // status = NEW
    
    orderRepo.bulkUpdate();               // UPDATE all NEW to PROCESSED
    
    System.out.println(o.getStatus());   // всё ещё NEW!
    // о cache managed, не отражает bulk change
}
```

Правило после bulk operation:
```java
em.flush();   // отправить pending changes до bulk
em.clear();   // очистить PersistenceContext
repo.bulkUpdate(OLD, NEW);
// managed-объекты now detached
// Subsequent operations create fresh managed instances из БД
```

flush перед bulk обеспечивает что pending changes committed to DB до bulk operation. clear после — invalidates cache, все managed entities become detached, future operations reload fresh state.

Или изолировать bulk operation в отдельном service или transaction чтобы не смешивать с regular ORM operations. Cleaner separation of concerns.

## OSIV filter internals

Open-Session-In-View это pattern где Hibernate session открыта на весь HTTP request. Session opens в servlet filter или Spring interceptor перед controller invocation, closes после response written. Внутри — @Transactional methods create their own transactions но share session.

Spring Boot по default enables OSIV — spring.jpa.open-in-view: true. Implementation через OpenEntityManagerInViewInterceptor или OpenEntityManagerInViewFilter — оба register OSIV mechanism.

Механика interceptor. Перед controller execution — new EntityManager created plus bound в TransactionSynchronizationManager. При запросе EntityManager (через @PersistenceContext) — proxy находит уже bound EntityManager, использует его. При @Transactional method start — transaction attached к this existing EntityManager. При @Transactional method end — transaction commits но EntityManager stays open. После controller completes — EntityManager closed by interceptor, resources released.

Effect на LazyInit. Session опen весь request lifecycle. Даже после @Transactional method returns — session still active. Lazy loading works from controller/view templates. Entities can be traversed anywhere в request.

Проблемы OSIV. Первая — скрывает N+1 problem. SELECT-ы летят из controller или view templates незаметно для developer. Кажется работает fast в testing но production shows accumulating queries.

Вторая — держит DB connection на весь HTTP request. Session требует connection. Даже когда request doing something else (external HTTP call, computation) connection остаётся allocated. Pool exhaustion под нагрузкой.

Третья — скрывает architectural проблемы. Entity утекает в controller, view templates access lazy relations. Layer boundaries blurred. Testing complexity — controllers implicitly depend на session state.

Правильная практика — spring.jpa.open-in-view: false плюс аккуратный дизайн. Service возвращает DTO не Entity. Все взаимодействия с БД внутри @Transactional services. Явные transaction boundaries. Никаких implicit sessions.

## Долгие транзакции: почему плохо

Классический анти-pattern — external calls внутри @Transactional:
```java
@Transactional
public void process(Order o) {
    repo.save(o);
    externalApi.call(o);        // ждём 30 секунд
    audit.log(o);
}
```

Что happens under the hood в течение 30 seconds external call. Connection в pool занят весь этот период — pool сокращается. Row locks acquired на save() удерживаются 30 seconds — блокируют others trying to modify same rows. В PostgreSQL long-running transaction блокирует VACUUM — dead tuples накапливаются не убираемые autovacuum, bloat растёт. В PgBouncer transaction-mode connection не возвращается в BouncerPool до конца.

При нагрузке — 100 concurrent такое requests exhaust connection pool. Following requests wait для connection — timeout eventually. Cascading failure — приложение становится unresponsive.

Правило абсолютное — внешние вызовы вне @Transactional:
```java
public void process(Order o) {
    Order saved = doSave(o);              // короткая transaction
    externalApi.call(saved);              // снаружи transaction
    doAudit(saved);                       // отдельная transaction
}

@Transactional
Order doSave(Order o) { return repo.save(o); }

@Transactional
void doAudit(Order o) { ... }
```

Три small transactions вместо одной long. Connection held только для БД work. External call не holds resources. Deadlock risk lower — locks released quickly.

Или outbox pattern — сохранить в БД plus запись в outbox таблицу в одной transaction. Отдельный job asynchronously reads outbox and делает external call. Атомарность через transaction БД, реальный external call decoupled по времени. Reliability preserved — if external call fails, retry from outbox.

## Практический read-only pattern

Common практический pattern для типового сервиса:
```java
@Service
class OrderService {

    @Transactional(readOnly = true)
    public List<OrderDto> list() {
        return repo.findAll().stream()
            .map(OrderDto::from)
            .toList();
    }

    @Transactional
    public Order create(OrderRequest req) {
        Order o = new Order(req);
        return repo.save(o);
    }
}
```

Read методы — readOnly = true plus DTO для return type. Написание DTO внутри transaction гарантирует что все lazy relations resolved (если нужны). Возврат DTO — no Hibernate proxies leak outside.

Write методы — regular @Transactional plus Entity возвращается только если нужно. Обычно достаточно return void или ID. DTO обеспечивает clean separation Entity (internal representation) и DTO (external contract).

## Session per test с deep understanding

@SpringBootTest plus @Transactional automatic rollback:
```java
@SpringBootTest
@Transactional
class OrderServiceTest {
    @Autowired OrderService svc;
    @Autowired OrderRepository repo;

    @Test
    void createOrder_savesEntity() {
        svc.create(new OrderRequest(...));
        assertThat(repo.findAll()).hasSize(1);
        // на конец теста rollback → БД чистая
    }
}
```

Как это работает internally. TransactionalTestExecutionListener registered в default listener chain. Перед test method invocation — starts transaction через TransactionManager. После test method — rolls back transaction.

Тест и сервис в одной transaction (REQUIRED propagation from @Transactional on service defaults to joining existing). Same EntityManager, same PersistenceContext. Repo видит изменение сервиса immediately потому что shared context.

Caveat REQUIRES_NEW. Если сервис делает REQUIRES_NEW — это отдельная transaction, откатится сама на выходе своего method. Но тестовая transaction её не увидит after её rollback — данные commited в separate transaction, не accessible через test repository queries если inner committed and outer rolled back.

Разница между integration test с REQUIRES_NEW и without. Without — все transactional, all rolled back, БД clean. With — REQUIRES_NEW branch commits independently, data persists в БД после test. Может быть проблема если following test expects clean state — order dependencies между tests.

Solutions. Explicit cleanup через @Sql("delete_all.sql") after each test. Или использование @Rollback(true) plus все в one transaction (no REQUIRES_NEW в test scope). Или Testcontainers with fresh DB per test class.

## Реальные кейсы КНП

Memory knp-fo-sync-notification-bugs — 6 багов sync-сервиса связанных с transactions plus JPA. @Transactional мёртв из-за self-invocation — session не открывается. Hibernate 6 LazyInit ловится там где раньше молчал. printStackTrace вместо log.error — ELK-слепая зона не видно transaction ошибок. Тихий скип на ошибке (MAX offset) — потеря данных без detection.

Урок общий — правильные transactions plus structured logging plus отсутствие self-invocation. Комплекс дисциплин, каждая отдельно не достаточна.

Memory taxreport21-java21-runtime-regressions — getById plus toDto вне transaction — LazyInit. Fix — findById (SELECT сразу) plus правильные @Transactional методы. Encapsulation внутри service. Хороший пример как migration к newer version (Java 21 plus Hibernate 6) может expose latent bugs which были hidden в older lenient behavior.

Memory knp-e2e-runner-hikari-isolation-poisoning — не JPA специфично но related. isolation равное -1 (opt-in) отравлял pool на pp-pgbouncer. gate-knp краснел 18 минут. Explicit transactionIsolation в конфигурации всегда — не полагаться на opt-in defaults.

## Best practices итог

Правила которые работают в production проверенные через годы boetin с subtle issues.

@Transactional на сервис не на repository и не на controller. Encapsulation transaction logic в service layer. Repository — CRUD interface без business logic. Controller — protocol handling только.

readOnly = true на все чтения. Optimizations JDBC plus JPA plus documentation. Убирает возможность accidental writes.

rollbackFor = Exception.class универсально безопасно. Или все свои exceptions через RuntimeException иерархию — no surprises with checked exception commits.

Сервис возвращает DTO не Entity. Entity не должна утекать за границы transaction. Cleanly декупированные layers.

Внешние API вне transaction. Outbox pattern или отдельные methods. Никаких long transactions holding resources during external calls.

OSIV = false. spring.jpa.open-in-view: false в application.yml. Явные transaction boundaries лучше implicit session per request.

@TransactionalEventListener(AFTER_COMMIT) для sending events после commit. Reliability without external outbox для simple cases.

findById не getReferenceById по default. getReferenceById только для FK связи где данные не нужны.

Bulk UPDATE plus em.clear() после. Или изолировать в отдельном service чтобы не смешивать с regular ORM operations.

Timeout на каждую transaction. @Transactional(timeout = 30) explicit. Combining с PostgreSQL statement_timeout.

Проверить что нет self-invocation регулярно через code review. Красная лампа при this.methodWithAnnotation в том же классе.

Логировать org.springframework.transaction в DEBUG при отладке. Внутренние transaction events visible в logs.

## Итоги: понимание работает без сюрпризов

@Transactional равно session равно PersistenceContext равно Connection — всё привязано к потоку через TransactionSynchronizationManager thread-local. Understanding этого mapping ключ к understanding поведения.

OSIV = false и работать с DTO. Явные transaction boundaries лучше implicit session per request. Никакой magic не helps скрывать architectural issues.

readOnly = true для чтения — optimizations plus documentation. Silent no-save на случайных setters — awareness needed.

Внешние вызовы вне transaction — outbox pattern предпочтительно для async external work. Никогда не holding transaction during external I/O.

@TransactionalEventListener(AFTER_COMMIT) для events после commit. Reliability без external tools.

Self-invocation самая частая причина «transaction не работает». Fix extract to another bean regularly checked.

Hibernate 6 строже к LazyInit чем Hibernate 5. Правильные transactions обязательны, «работало раньше» не аргумент — migration к newer versions exposes latent bugs.

Итог блока @Transactional. Файлы 32-35 покрыли ACID basics и isolation levels, @Transactional internals через proxies и thread-local resource binding, advanced features с rollback rules и event listeners, JPA специфику с PersistenceContext и OSIV. Complete picture для reliable transaction handling в enterprise Spring приложениях.

Дальше — Spring Cloud с service discovery, gateway, resilience patterns и другими компонентами microservices infrastructure.
