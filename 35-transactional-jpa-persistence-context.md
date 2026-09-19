# 35. @Transactional плюс JPA: PersistenceContext, flush, LazyInit

## Ключевое связывание transaction равно PersistenceContext

При старте @Transactional метода в JPA приложении Spring выполняет несколько связанных операций. Открывает JPA-транзакцию через JpaTransactionManager. Создаёт или берёт из pool EntityManager. Привязывает его к текущему потоку через TransactionSynchronizationManager. Все инъекции @PersistenceContext EntityManager в коде получают тот же EM через прокси механизм.

```
@Transactional method
    │
    ▼
JpaTransactionManager.doBegin
    │
    ├─ EntityManagerFactory.createEntityManager()
    ├─ em.getTransaction().begin()
    ├─ bind (EntityManager, ResourceHolder) to thread
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
    └─ unbind from thread
```

SharedEntityManagerCreator это ключевой инфраструктурный класс. Когда Spring инжектит через @PersistenceContext, разработчик получает не настоящий EntityManager а прокси (SharedEntityManagerCreator). При каждом вызове прокси смотрит thread-local — есть ли связанный EM? Да — возвращает связанный (тот же для всей transaction). Нет — создаёт временный EM на один вызов (не рекомендуется потому что вне transaction).

Из этого следует важное — вне @Transactional операции с EM могут не работать или работать некорректно. Отдельные операции могут выполниться как auto-committed, но нет consistency между multiple operations.

## Три сценария вызова

Внутри @Transactional. Стандартный правильный use case:
```java
@Transactional
public void save(Order o) {
    em.persist(o);              // managed
    o.setStatus(NEW);           // dirty
    // выход → flush → INSERT + UPDATE (одна transaction)
}
```

Работает как ожидается — все operations в одной transaction, атомарность гарантирована.

Вне @Transactional в web контроллере с open-in-view. Spring Boot по default включает Open-Session-In-View (OSIV) — session открыта на весь HTTP запрос:
```yaml
spring.jpa.open-in-view: true    # default
```

Что делает — OpenEntityManagerInViewInterceptor открывает EM в начале HTTP request, закрывает в конце. Внутри есть session, LazyInit работает даже вне @Transactional service methods.

Проблемы OSIV. Скрывает N+1 problem — SELECT-ы летят из controller или view слоя незаметно. Держит DB connection на весь HTTP request — если service делает внешние вызовы, connection зря простаивает во время external call. Скрывает архитектурные ошибки — Entity утекает в controller что нарушает layering.

Правильная практика — spring.jpa.open-in-view: false plus аккуратный дизайн где service возвращает DTO а не Entity. Entity живёт только в service слое.

Вне @Transactional и без OSIV. em.find(...) вернёт detached object (запрос выполнится но связь с session потеряется сразу). em.persist(...) бросит TransactionRequiredException. Правильное поведение — операции с БД должны быть в transaction.

## Flush когда именно

Внутри @Transactional-метода Hibernate накапливает изменения (dirty checking plus persist/remove) в PersistenceContext. Реальные SQL-запросы летят в БД на flush.

Автоматический flush в FlushMode.AUTO (default) происходит в нескольких moments. Перед commit обязательно — гарантия что все pending changes записаны до финализации transaction. Перед выполнением query — чтобы query увидел изменения текущей transaction. Иногда перед find() если сущность изменена (сложные rules чтобы поддерживать consistency).

Пример поведения:
```java
@Transactional
public void updateAndCount() {
    Order o = em.find(Order.class, 1L);
    o.setStatus(NEW);              // dirty, не в БД

    long count = em.createQuery(
        "SELECT count(o) FROM Order o WHERE o.status = :s", Long.class)
        .setParameter("s", NEW)
        .getSingleResult();
    // ← перед этим query flush → UPDATE + SELECT
    // count увидит текущий изменённый Order

    // на выходе flush (уже сделан), commit
}
```

Manual flush через явный em.flush() полезен в нескольких сценариях. Проверить что INSERT прошёл валидацию БД до конца метода — если constraint violation, exception раньше а не на commit. Получить @GeneratedValue(IDENTITY) id немедленно для использования в других операциях.

FlushMode variants. AUTO — automatic flush (default). COMMIT — flush только на commit, queries могут вернуть устаревшие данные within transaction. MANUAL — flush только явный em.flush(), настраивается для read-only оптимизации.

readOnly и flush имеют важное взаимодействие. @Transactional(readOnly = true) в Hibernate устанавливает Session.setDefaultReadOnly(true) plus Session.setFlushMode(FlushMode.MANUAL). Отсюда нет dirty checking, нет UPDATE даже если изменил поле, изменение молча теряется.

Caveat критический — если в read-only методе случайно entity.setX(...) — ничего не происходит с БД. Не паника когда «изменение не сохранилось» — проверить readOnly setting.

## LazyInitializationException глубже

Обсуждали в файле 15 в контексте JPA performance. Здесь связь с transaction management:
```java
@Transactional(readOnly = true)
public Order load(Long id) {
    return repo.findById(id).orElseThrow();
}

// controller:
Order o = svc.load(1L);
o.getItems().forEach(...);         // ← LazyInit!
```

Причина — orderItems это LAZY collection представленная Hibernate proxy (PersistentBag). При обращении пытается выполнить SELECT для loading данных.

Но session закрыта (transaction закончилась при выходе из svc.load). Нет source для SELECT — LazyInitializationException.

Решения различаются по quality.

DTO в service — правильный подход:
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

DTO конструируется внутри transaction где session активна. LAZY доступ работает. Наружу возвращается plain DTO без Hibernate proxies.

JOIN FETCH или @EntityGraph — форсированно загрузить relationships перед закрытием session. Тоже работает но Entity утекает наружу.

OSIV — session до конца HTTP request. Работает но скрывает problems.

FetchType.EAGER — всегда загружать. Худший подход — избыточные queries, performance degradation.

Hibernate 6 стало строже с LazyInit. С Hibernate 6 (Spring Boot 3) LazyInit ловится чаще. Раньше некоторые случаи молчали, теперь explicit exceptions.

Реальный кейс из КНП memory knp-fo-sync-notification-bugs. @Transactional был мёртв из-за self-invocation. Значит session никогда не открывалась. Hibernate 6 стал жёстче — LazyInit проявился где Hibernate 5 молчал. Урок — не полагаться на «работает в Hibernate 5». Правильные transactions plus DTO обязательны.

getById vs findById в Spring Data. findById(id) возвращает Optional<T> и делает SELECT сразу. getReferenceById(id) возвращает T как прокси без SELECT — LazyInit если использовать вне transaction. Практика — findById по default. getReferenceById только когда «мне нужна ссылка не читая», обычно для FK установления.

## Cascade и transaction

Cascade расширяет операции persist/remove на связанные объекты все в одной transaction:
```java
@Entity
class Order {
    @OneToMany(mappedBy="order", cascade = ALL, orphanRemoval = true)
    List<OrderItem> items;
}

@Transactional
public void createOrder(Order o) {
    o.setItems(List.of(item1, item2, item3));
    repo.save(o);
    // на flush → INSERT Order + 3 × INSERT OrderItem — одна transaction
}
```

Если что-то не пройдёт валидацию БД — rollback всей transaction. Никаких «половинных» состояний. Атомарность гарантирована.

## Bulk operations

Bulk UPDATE и DELETE операции имеют важную особенность:
```java
@Modifying
@Query("UPDATE Order o SET o.status = :new WHERE o.status = :old")
int bulkUpdate(...);
```

Bulk UPDATE не проходит через PersistenceContext. Managed-объекты в текущей session остаются со старым значением. Cache становится stale по отношению к БД.

Правило после bulk operation:
```java
em.flush();   // отправить pending changes до bulk
em.clear();   // очистить cache
repo.bulkUpdate(OLD, NEW);
// managed-объекты теперь detached
```

Или изолировать bulk operation в отдельном service или transaction чтобы не смешивать с regular ORM operations.

## Transaction propagation в JPA практика

REQUIRED default для обычных сервисов. Всё в своих transactions.

REQUIRES_NEW для independent audit. Даже если main transaction откатывается, audit record сохраняется независимо:
```java
@Service
class OrderService {
    @Autowired AuditService audit;

    @Transactional
    public void createOrder(Order o) {
        repo.save(o);
        audit.log("Order created", o);   // отдельная transaction
    }
}

@Service
class AuditService {
    @Transactional(propagation = REQUIRES_NEW)
    public void log(...) {
        auditRepo.save(new AuditEntry(...));
    }
}
```

Caveat — два connections в pool одновременно (suspended outer plus new inner). При нагрузке pool истощается.

NESTED для «попыток». Внутренняя операция может упасть без impact на внешнюю:
```java
@Transactional
public void createOrder(Order o) {
    repo.save(o);
    try {
        experimentalFeature.apply(o);   // NESTED
    } catch (Exception e) {
        log.warn(...);
    }
    // если experimental упал → savepoint откачен, createOrder продолжается
}

@Transactional(propagation = NESTED)
public void apply(Order o) { ... }
```

SUPPORTS для методов работающих и в transaction и без:
```java
@Transactional(propagation = SUPPORTS)
public List<Order> list() {
    return repo.findAll();
}
```

Редко полезно. Обычно readOnly = true plus REQUIRED достаточно.

## Session per request vs @Transactional

Важное различие — OSIV это session per HTTP request. @Transactional это session per transaction.

С OSIV включённым session открыта весь request. @Transactional открывает transaction внутри. По окончании transaction session НЕ закрывается — продолжается до конца request.

С OSIV выключенным session открыта ТОЛЬКО в @Transactional. Вне transaction session не существует.

Правильная практика — OSIV OFF plus все взаимодействия с БД внутри @Transactional сервисов plus возвращать DTO из сервисов. Contract чёткий, no accidental N+1, no LazyInit surprises.

## Долгие транзакции плохо

Классический анти-pattern:
```java
@Transactional
public void process(Order o) {
    repo.save(o);
    externalApi.call(o);        // ждём 30 секунд
    audit.log(o);
}
```

30 секунд open transaction. Connection в pool занят весь этот период — pool истощается быстро. Row locks удерживаются 30 секунд — deadlocks растут. Long-running transaction блокирует VACUUM в PostgreSQL — bloat накапливается. В PgBouncer transaction-mode connection не возвращается в pool до конца.

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

Или outbox pattern — сохранить в БД plus запись в outbox, отдельный job делает external call асинхронно. Атомарность через транзакцию БД, реальный external call decoupled по времени.

## Read-only pattern

Common practical pattern для типового сервиса:
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

Read методы — readOnly = true plus DTO для return type. Write методы — regular @Transactional plus Entity возвращается только если нужно. Chapter DTO обеспечивает clean separation между Entity (internal representation) и DTO (external contract).

## Session per test

@SpringBootTest plus @Transactional автоматический rollback после теста:
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

Тест и сервис в одной transaction. Repo видит изменение сервиса потому что share PersistenceContext.

Caveat REQUIRES_NEW. Если сервис делает REQUIRES_NEW — это отдельная transaction, откатится сама на выходе своего метода. Но тестовая transaction её не увидит после её rollback — данные не accessible через тест repository queries если inner transaction сделал commit и внешняя откатывается.

## Реальные кейсы КНП

Memory knp-fo-sync-notification-bugs — 6 багов sync-сервиса связанных с transactions plus JPA. @Transactional мёртв из-за self-invocation — session не открывается. Hibernate 6 LazyInit ловится там где раньше молчал. printStackTrace вместо log.error — ELK-слепая зона не видно transaction ошибок. Тихий скип на ошибке (MAX offset) — потеря данных без detection.

Урок общий — правильные transactions plus structured logging plus отсутствие self-invocation. Не отдельные проблемы а комплекс дисциплин.

Memory taxreport21-java21-runtime-regressions — getById plus toDto вне transaction — LazyInit. Fix findById (SELECT сразу) plus правильные @Transactional методы. Encapsulation внутри service.

Memory knp-e2e-runner-hikari-isolation-poisoning — не JPA специфично но связано с transactions. isolation равное -1 (opt-in) отравлял pool на pp-pgbouncer. gate-knp краснел 18 минут. Явно указывать isolation в конфигурации.

## Best practices итог

Правила которые работают в production:

@Transactional на сервис не на repository и не на controller. Encapsulation transaction logic в service layer.

readOnly = true на все чтения. Optimizations plus documentation.

rollbackFor = Exception.class универсально безопасно. Или все свои exceptions через RuntimeException иерархию.

Сервис возвращает DTO не Entity. Entity не должна утекать за границы transaction.

Внешние API вне transaction. Outbox pattern или отдельные methods.

OSIV = false. spring.jpa.open-in-view: false. Явные transaction boundaries.

@TransactionalEventListener(AFTER_COMMIT) для sending events после commit.

findById не getReferenceById по default (если только не для FK связи).

Bulk UPDATE plus em.clear() после. Или изолировать в отдельном service.

Timeout на каждую transaction. @Transactional(timeout = 30) explicit.

Проверить что нет self-invocation. Регулярный code review.

Логировать org.springframework.transaction DEBUG при отладке.

## Итоги

@Transactional равно session равно PersistenceContext равно connection — всё привязано к потоку через TransactionSynchronizationManager thread-local.

OSIV = false и работать с DTO. Явные transaction boundaries лучше implicit session per request.

readOnly = true для чтения — оптимизации plus documentation.

Внешние вызовы вне transaction — outbox pattern predпочтительно для async external work.

@TransactionalEventListener(AFTER_COMMIT) для events после commit. Обеспечивает outbox-подобную reliability без external tools.

Self-invocation самая частая причина «transaction не работает». Fix extract to another bean.

Hibernate 6 строже к LazyInit чем Hibernate 5. Правильные transactions обязательны, «работало раньше» не аргумент.

Итог блока @Transactional. Файлы 32-35 покрыли ACID basics и isolation levels, @Transactional internals через proxies, advanced features с rollback rules и listeners, JPA специфику с PersistenceContext и OSIV. Complete picture для reliable transaction handling в enterprise Spring приложениях.

Дальше — Spring Cloud с discovery, gateway, resilience patterns и другими компонентами microservices infrastructure.
