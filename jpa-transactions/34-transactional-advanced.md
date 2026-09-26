# 34. Продвинутое @Transactional: rollback rules, listeners, savepoints, тестирование

## Куда мы идём и зачем

В предыдущем файле мы разобрали механику @Transactional на уровне CGLib proxy generation, TransactionInterceptor invocation chain, PlatformTransactionManager implementations, thread-local resource binding через TransactionSynchronizationManager. Понимание этой механики позволяет предсказывать поведение системы в стандартных сценариях. Но реальные production приложения сталкиваются с более тонкими проблемами. Почему checked exception в transactional методе приводит к commit? Как правильно опубликовать событие после успешного commit? Как реализовать «попытку с возможностью отката» без обрыва внешней транзакции? Как правильно тестировать transactional code?

Разница между разработчиком «знающим @Transactional» и «понимающим @Transactional» именно здесь. Первый ставит аннотацию и надеется. Второй знает что RollbackForRuleAttribute сравнивает depth exception class в hierarchy для нахождения наиболее specific rule. Знает что TransactionalEventListener использует ApplicationEventPublisher plus TransactionSynchronization для phase-based execution. Знает что NESTED propagation работает через JDBC savepoints с ROLLBACK TO SAVEPOINT SQL. Знает что @DataJpaTest автоматически включает @Transactional plus @Rollback с определёнными rules.

В этом файле разберём каждый advanced аспект детально. Rollback rules internals — как RuleBasedTransactionAttribute решает rollback или commit. Программный контроль через TransactionTemplate — когда declarative approach недостаточен. TransactionSynchronization и TransactionalEventListener как build events tied к transaction lifecycle. Savepoints и как NESTED propagation работает на JDBC level. Read-only optimizations на JPA plus JDBC уровнях. Timeout handling и его caveats. Testing framework с deep dive в @Transactional aspects and @Sql fixtures.

## Rollback rules: почему checked exception коммитится

Наиболее контр-интуитивное поведение @Transactional связано с default rollback rules. Rule простая — только RuntimeException и Error приводят к rollback. Checked exceptions приводят к COMMIT транзакции даже если exception propagates наверх. Классический баг:
```java
@Transactional
public void save(Order o) throws IOException {
    repo.save(o);
    externalCall();       // бросает IOException
}
```

Ожидание — при IOException транзакция откатывается. Реальность — IOException это checked exception, транзакция COMMIT-ится, order остаётся в БД. Только потом exception пробрасывается наверх. Complete mess-состояние — операция «упала» но частично сохранилась.

Внутри Spring эта logic реализована в RuleBasedTransactionAttribute которая implementation of TransactionAttribute. При completeTransactionAfterThrowing в TransactionAspectSupport вызывается txAttr.rollbackOn(ex). Реализация в DefaultTransactionAttribute (простой вариант) проверяет ex instanceof RuntimeException || ex instanceof Error. RuleBased добавляет explicit rules.

Historical причина такого поведения from Java EE традиции. Checked exceptions считались business exceptions — «expected alternative flow». Recovery путь — try/catch обрабатывает. Unchecked считались system exceptions — «unexpected failure». Автоматический rollback имел смысл для последних но не для первых.

Practically правило редко подходит. Business exceptions в enterprise apps часто ARE reasons для rollback — validation failure, precondition violation, integrity constraint. Что checked или unchecked дизайнерский выбор конкретной library.

rollbackFor атрибут явно расширяет rollback rules:
```java
@Transactional(rollbackFor = Exception.class)
public void save(Order o) throws IOException {
    // ...
}
```

Внутри RuleBasedTransactionAttribute хранит список RollbackRuleAttribute. Каждое имеет exception class. При exception matches (или его parent) — rollback triggered. Rules matched через isAssignableFrom checking. Class hierarchy проверяется — если exception is IOException и rule matches IOException plus rule matches Exception, оба match. Выбирается наиболее specific — depth in class hierarchy наименьший.

Аналогично noRollbackFor исключает specific exceptions из rollback:
```java
@Transactional(noRollbackFor = ExpectedBusinessException.class)
public void process() { ... }
```

Внутри NoRollbackRuleAttribute список. При exception matches NoRollback rule — commit несмотря на что exception RuntimeException. Priority — NoRollback overrides Rollback rule.

Best practice universal — @Transactional(rollbackFor = Exception.class) для всех write methods. Полностью удаляет surprise-behavior с checked exceptions. Или полностью custom BusinessException hierarchy наследующаяся от RuntimeException:
```java
public class BusinessException extends RuntimeException { ... }
```

Всё custom exceptions Runtime-based — default rollback rules работают правильно. Consistent approach через весь codebase лучше чем per-method rollbackFor configuration.

Ручной setRollbackOnly для явного откатывания без exception. Внутри реализовано через TransactionStatus.setRollbackOnly который устанавливает флаг rollbackOnly в TransactionInfo плюс в underlying transaction object. На commit TransactionManager проверяет флаг — если true инициируется rollback вместо commit. Из вне @Transactional через:
```java
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
```

UnexpectedRollbackException возникает когда inner REQUIRED transaction пометил rollbackOnly а outer method не бросил exception. На commit outer transaction видит помеченный rollback-only status в участвующей transaction и throws UnexpectedRollbackException. Application может не ожидать такое exception если написан без учёта rollback-only mechanism.

Правильная стратегия handling. При rollback-only situation в inner transaction — bросить exception который outer transaction обработает через rollback rules. Не полагаться на unexpected rollback exception как control flow — используй explicit throw new RuntimeException(...) или подобное.

## Программный контроль через TransactionTemplate

Иногда declarative @Transactional недостаточен. Динамические propagation в зависимости от runtime conditions. Bulk processing с per-item transactions где failure одного не блокирует others. Точный контроль над setRollbackOnly без relying на exceptions.

TransactionTemplate это convenience класс wrapping PlatformTransactionManager с fluent API:
```java
@Service
class ProcessorService {
    private final TransactionTemplate tx;
    private final OrderRepository repo;

    public ProcessorService(PlatformTransactionManager txManager, OrderRepository repo) {
        this.tx = new TransactionTemplate(txManager);
        this.tx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
        this.tx.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
        this.tx.setTimeout(10);
        this.repo = repo;
    }

    public void processAll(List<Order> orders) {
        for (Order o : orders) {
            try {
                tx.execute(status -> {
                    repo.save(o);
                    if (validate(o).isBad()) {
                        status.setRollbackOnly();
                    }
                    return null;
                });
            } catch (Exception e) {
                log.error("failed to process {}", o.getId(), e);
                // одна не пройдёт, другие продолжатся
            }
        }
    }
}
```

TransactionTemplate.execute внутри делает то же что TransactionInterceptor. Получает transaction через txManager.getTransaction. Выполняет callback. Обрабатывает exceptions и status.rollbackOnly. Commit или rollback. Возвращает результат callback.

Ключевое отличие от declarative — programmatic control per invocation. Один и тот же bulk loop может иметь per-item transactions with different settings if needed. Business logic determines transaction attributes at runtime.

TransactionCallback interface для actions с return value. TransactionCallbackWithoutResult для void actions. Lambda syntax обычно cleanest — status -> { ... } implements TransactionCallback<Object>.

Плюсы vs @Transactional. Динамический контроль — можно менять propagation, isolation, timeout на runtime. setRollbackOnly без бросания exception. Гибкая настройка per-invocation. Легче unit-тестить потому что transaction manager можно замокать (@Transactional через AOP proxy сложнее mock).

Минусы. Многословнее чем аннотация. Сложнее reason about — читатель класса не сразу видит что метод transactional. Требует передачу TransactionTemplate или PlatformTransactionManager как dependency.

Practically TransactionTemplate используется для сценариев где @Transactional недостаточен. Bulk processing per-item transactions. Dynamic configuration based на runtime conditions. Long-running processes где границы transactions определяются алгоритмом а не method boundaries.

Ручной PlatformTransactionManager для полного low-level control. TransactionTemplate убирает try/catch/finally boilerplate — usually preferred. Direct manager use рекомендуется только для сложных custom logic which не fit в TransactionTemplate model.

## readOnly оптимизации: JDBC и JPA levels

Атрибут readOnly даёт несколько оптимизаций на разных уровнях stack. Simple annotation @Transactional(readOnly = true) has multiple effects.

JDBC level. Connection.setReadOnly(true) вызывается по началу транзакции. PostgreSQL по этому hint может оптимизировать — использовать read-only connection pool if available (например в load-balanced setup), skip write locks acquisition where safe, использовать read replica if configured (некоторые frameworks routing based на this).

JPA/Hibernate level. Session.setDefaultReadOnly(true) устанавливается. FlushMode.MANUAL устанавливается вместо AUTO. Это значит Hibernate не будет автоматически flush при query execution или commit. Dirty checking не работает — even если entity modified, changes не будут saved.

Memory footprint reduction. Hibernate обычно maintains snapshot каждого loaded entity для dirty checking comparison. При readOnly snapshot не создаётся — экономия memory особенно для queries returning many entities. Может быть significant для reports или batch reads returning 10000+ entities.

Semantic documentation. Reader кода видит @Transactional(readOnly = true) и знает что метод не modifies data. Дополнительный layer защиты и понимания.

Critical caveat. Если случайно в read-only method someone calls entity.setField(...) — no exception, но changes silently не saved потому что нет flush. Может быть источник sneaky bugs если разработчик забыл readOnly или method эволюционировал.

Практика — на всех read-only методах ставить readOnly = true. Как code review pattern облегчает понимание. Убирает возможность bugs from accidental writes. Даёт database plus ORM оптимизации.

## Timeout: как реально работает

@Transactional(timeout = 30) устанавливает 30 секунд timeout. При превышении транзакция автоматически откатывается с timeout exception.

Реализация зависит от resource. JDBC level — Connection.setQueryTimeout(30) устанавливается на каждый Statement в транзакции. Драйвер после 30 seconds пытается cancel query — success depends on driver и БД support. JPA level — em.createQuery(...).setHint("javax.persistence.query.timeout", 30000) применяется к каждому Hibernate query. Similar cancellation semantics.

Caveat — timeout не всегда работает как ожидается. Часть операций может игнорировать. Waiting на lock acquire не всегда prints timeout. Waiting на network не всегда interruptable. Query execution может ignore cancel signal.

Более надёжно установить statement_timeout и lock_timeout на уровне PostgreSQL. Это работает при уровне сервера БД. При превышении PostgreSQL cancels query возвращая error к клиенту. Application получает exception через driver.

Установка через свойства pool:
```yaml
spring.datasource.hikari.data-source-properties:
  socketTimeout: 60           # секунды - максимум на statement
```

Или на уровне session при подключении:
```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';
```

Combining application-level @Transactional timeout plus PostgreSQL statement_timeout даёт defense in depth. @Transactional timeout attempts cancel; if fails PostgreSQL forces termination.

## Savepoints и NESTED propagation

NESTED propagation позволяет попытаться операцию с возможным откатом без потери всей внешней транзакции. Реализуется через SQL savepoints — feature большинства современных баз данных.

Savepoint это named point в текущей транзакции к которому можно вернуться через ROLLBACK TO SAVEPOINT. Все changes made после savepoint откатываются, но changes до savepoint plus transaction сама remain active. По completion savepoint автоматически освобождается либо явно через RELEASE SAVEPOINT.

Внутри JDBC — Connection.setSavepoint(String name) создаёт savepoint. Connection.rollback(Savepoint) откатывает к нему. Connection.releaseSavepoint(Savepoint) освобождает.

DataSourceTransactionManager для NESTED использует именно эти JDBC methods. При начале NESTED transaction — savepoint создаётся, name автоматически generated. При success — savepoint released, transaction continues. При exception matching rollback rules — rollback к savepoint, transaction продолжается, exception propagates наверх (может быть caught или propagated дальше).

Практический use case — audit logging or optional operations которые могут fail без impacting main flow:
```java
@Service
class OrderService {
    @Transactional
    public void createOrder(Order o) {
        repo.save(o);
        try {
            audit.log(o);
        } catch (Exception e) {
            log.warn("audit failed", e);
        }
    }
}

@Service
class AuditService {
    @Transactional(propagation = Propagation.NESTED)
    public void log(Order o) {
        // Если бросит exception — rollback до savepoint,
        // внешняя createOrder продолжится
        auditRepo.save(new AuditEntry(...));
    }
}
```

Разница NESTED vs REQUIRES_NEW. NESTED использует один Connection, одну transaction с savepoint внутри. Инкрементальный откат — только внутренние changes. Если внешняя откатится, внутренняя тоже откатится (potentially never committed).

REQUIRES_NEW создаёт полностью новый Connection и полностью независимую transaction. Может commit independently от outer. Requires два connections в pool одновременно. Suspended outer держит свой connection. Inner grabs new connection. При heavy usage — pool exhaustion.

Требования для NESTED. БД должна поддерживать savepoints — PostgreSQL, Oracle, MySQL InnoDB поддерживают. В Spring — DataSourceTransactionManager.setNestedTransactionAllowed(true) (default true). JpaTransactionManager с Hibernate — работает.

Ограничение — savepoints имеют overhead. Каждый create/rollback/release — SQL command. Для короткой NESTED transaction overhead может быть заметен. Not рекомендуется для fine-grained нестинга (например NESTED в цикле для каждого item).

## TransactionSynchronization и @TransactionalEventListener

Spring предоставляет TransactionSynchronization API для callbacks на transaction lifecycle events. Registered через:
```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override public void beforeCommit(boolean readOnly) { ... }
    @Override public void beforeCompletion() { ... }
    @Override public void afterCommit() { ... }
    @Override public void afterCompletion(int status) { ... }
});
```

Все sync callbacks вызываются TransactionManager в specific points transaction lifecycle. beforeCommit — до fizual commit, можно бросить exception для aborting commit. beforeCompletion — до close resources (commit или rollback). afterCommit — после successful commit, exceptions игнорируются (transaction уже completed). afterCompletion — после close resources с indication commit/rollback status.

Registration через TransactionSynchronizationManager stores callback в thread-local synchronizations Set. При transaction end TransactionManager iterates всё callbacks в этом Set и вызывает соответствующие methods. Order — usually registration order.

Common use case — publish event только после successful commit. Если публиковать перед commit и потом commit fails — событие уже опубликовано, downstream действовал на основе несуществующих данных. Реализация через afterCommit callback:
```java
@Transactional
public void save(Order o) {
    repo.save(o);
    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronizationAdapter() {
            @Override
            public void afterCommit() {
                publisher.publish(o);
            }
        });
}
```

@TransactionalEventListener это higher-level API на этой же механике. Работает через ApplicationEventPublisher. Publisher сохраняет event до commit. Registered TransactionalEventListener bound к specific phase (BEFORE_COMMIT, AFTER_COMMIT, AFTER_ROLLBACK, AFTER_COMPLETION). Framework уже handles registration synchronization internally.

Пример usage:
```java
@Service
class OrderService {
    @Autowired ApplicationEventPublisher events;

    @Transactional
    public void save(Order o) {
        repo.save(o);
        events.publishEvent(new OrderCreatedEvent(o));
    }
}

@Component
class OrderEventListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handle(OrderCreatedEvent event) {
        publisher.publish(event);
    }
}
```

Как это работает internally. При publishEvent — ApplicationEventMulticaster smart enough определить есть ли active transaction. Если да — event stored для delivery в later phase. TransactionalApplicationListener implements TransactionSynchronization callbacks. При AFTER_COMMIT — event delivered к listener method.

Caveat — событие теряется если нет активной transaction. Если publishEvent called outside @Transactional — no transaction, no way to wait for commit. Event ignored. Можно разрешить fallback:
```java
@TransactionalEventListener(fallbackExecution = true)
```

Тогда без transaction listener executes как regular @EventListener sync с publishEvent.

Async обработка через @Async на listener plus @EnableAsync в config. Listener executes в separate thread через TaskExecutor. Не блокирует main flow. При failure listener — не откатывает already-committed transaction. Полезно для heavy post-processing (send emails, update caches, generate reports).

## Тестирование transactional code

Spring Test framework имеет sophisticated support для тестирования @Transactional. Основной механизм — @Transactional на test class или method automatically rolls back после теста completion.

Как это работает. TransactionalTestExecutionListener зарегистрирован в default listeners chain. Перед test method — begins transaction через TransactionManager. После method — rolls back transaction по default. Это isolates each test — БД остаётся в pristine state между tests.

Ключевые аннотации. @Transactional на test class или method — activates automatic rollback. @Rollback(false) на method — не откатывать, commits changes (rare use). @Commit synonym для @Rollback(false). @Sql("script.sql") — execute SQL script before test для fixtures. @SqlGroup для multiple scripts.

Пример:
```java
@SpringBootTest
@Transactional
class OrderServiceIntegrationTest {
    @Autowired OrderService svc;
    @Autowired OrderRepository repo;

    @Test
    void createOrder_persistsToDb() {
        svc.createOrder(new Order(...));
        assertThat(repo.findAll()).hasSize(1);
        // после теста - rollback → БД чистая
    }
}
```

Тест и сервис в одной transaction. Repo видит изменение сервиса потому что share PersistenceContext. При end теста — rollback, всё откатывается.

Caveat REQUIRES_NEW. Если код внутри создаёт new transaction (REQUIRES_NEW) — она независима от test transaction. Commits independently. Test rollback не откатывает эту new transaction. Данные persist в БД после теста.

Практически это может быть проблема или feature. Проблема — если тест ожидает clean state, а REQUIRES_NEW commits leave data. Feature — если хочешь тестировать «отдельная transaction commits несмотря на rollback of parent». Требуется awareness of this behavior.

@DataJpaTest это специализированная annotation для JPA testing. Enables @Transactional (with rollback). Configures in-memory database (H2 usually). Autoconfigures TestEntityManager для easy setup. Ограничивает context только к JPA-related beans (faster startup).

@JdbcTest for pure JDBC testing. Similar но для JdbcTemplate без JPA.

Testcontainers approach для real database testing. Docker container с PostgreSQL запускается перед тестами. Real PostgreSQL behavior including все edge cases. Slower than H2 но более accurate. Recommended для integration tests критичных features.

Unit tests без Spring context быстрее. Мокать репозиторий через Mockito:
```java
class OrderServiceTest {
    OrderRepository repo = mock(OrderRepository.class);
    OrderService svc = new OrderService(repo);

    @Test
    void createOrder_callsRepo() {
        svc.createOrder(new Order(...));
        verify(repo).save(any());
    }
}
```

Быстро (нет Spring boot) но нет проверки actual transaction behavior. Combination unit tests для business logic plus integration tests для transaction integration стандартная стратегия.

## Диагностика типичных проблем

«Изменения не сохраняются». Standard checklist. Метод public? Класс это Spring bean не new? @Transactional присутствует и correctly? Не self-invocation (this.methodWithAnnotation)? Не бросается checked exception без rollbackFor? Не readOnly = true случайно? Не @Async (session в другом thread)?

Logging при подозрении — org.springframework.transaction в DEBUG или TRACE level. Показывает transaction lifecycle events. Отсутствие logs для метода — @Transactional не активируется.

Runtime introspection класса bean — bean.getClass().getName. CGLIB suffix means proxy created. Regular class name — Spring не обернул. Обычно означает @Transactional не detected или bean не в context.

«UnexpectedRollbackException». Внутренний REQUIRED пометил rollback-only. Внешний не бросил exception но пытается commit. Spring обнаруживает conflict — throws UnexpectedRollbackException.

Fix — использовать REQUIRES_NEW для «независимых» операций которые могут rollback без impact на outer. Или явно проверять и бросать exception на error conditions чтобы outer transaction handled правильно.

«Connection pool exhausted» — стандартные причины. REQUIRES_NEW внутри REQUIRED держит 2 connections одновременно (suspended plus new). Внешние API внутри @Transactional — connection занят весь время external call. Утечка connection — не возвращён в pool из-за bug в error handling или resource management.

leak-detection-threshold в HikariCP помогает локализовать где utечёт. При установке (например 60000ms) Hikari log warning со stack trace откуда connection был запрошен если не возвращён в pool за указанное время. Обычно достаточно чтобы найти problematic code.

«Deadlock» — две transactions ждут locks друг друга. Cycle waiter graph. PostgreSQL detected через periodic checks. Одна transaction picked и aborted с deadlock_detected error.

Fix через consistent ordering locks (всегда locking rows в same order например by id ascending), уменьшение длины transactions, использование optimistic locking вместо pessimistic где возможно.

## Best-practice полная конфигурация

Собранная воедино правильная transaction setup для production Service:
```java
@Configuration
@EnableTransactionManagement
public class TxConfig {
    // Spring Boot автоконфигурит всё для JPA
    // Custom только если нужно
}

@Service
class OrderService {
    @Autowired OrderRepository repo;
    @Autowired ApplicationEventPublisher events;

    @Transactional(readOnly = true, timeout = 5)
    public Order findById(Long id) {
        return repo.findById(id).orElseThrow();
    }

    @Transactional(rollbackFor = Exception.class, timeout = 30)
    public Order create(OrderRequest req) {
        Order o = new Order(req);
        repo.save(o);
        events.publishEvent(new OrderCreatedEvent(o));
        return o;
    }
}

@Component
class OrderEventListener {
    @Async
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void publish(OrderCreatedEvent event) {
        rabbitPublisher.publish(event);
    }
}
```

Ключевые decisions. readOnly = true на read methods. rollbackFor = Exception.class на write methods для avoiding checked exception surprise. Timeout explicit values based на expected duration. Events через ApplicationEventPublisher plus @TransactionalEventListener(AFTER_COMMIT) для asynchronous post-processing без blocking main transaction.

## Итоги: что нужно помнить

По default только RuntimeException и Error приводят к rollback. Checked exceptions приводят к COMMIT несмотря на exception. Всегда rollbackFor = Exception.class или всё через RuntimeException-based иерархию.

readOnly = true даёт JDBC read-only mode, Hibernate MANUAL flush, database оптимизации. Обязательно на всех read-only methods. Silent no-save при случайных setters — awareness needed.

Timeout ограничивает transaction execution time. JDBC statement timeout под капотом. Combining application-level plus PostgreSQL statement_timeout defense in depth.

Savepoints через NESTED propagation для «попыток с откатом» без потери всей transaction. Один Connection в отличие от REQUIRES_NEW который требует два.

TransactionSynchronization и @TransactionalEventListener для callbacks on transaction events. AFTER_COMMIT typical для publishing events after successful transaction — outbox-like reliability без external tools.

@Async on @TransactionalEventListener для async post-processing events в separate thread. Не блокирует main transaction path.

TransactionTemplate для программного transaction control когда declarative @Transactional недостаточен. Dynamic configuration per-invocation, setRollbackOnly без exception, легче mock в tests.

@Transactional в тестах даёт automatic rollback. Isolates tests. Caveat REQUIRES_NEW не откатывается с test transaction. @Commit или @Rollback(false) для explicit commits в specific tests. @Sql для fixtures.

Дальше — @Transactional plus JPA специфика с PersistenceContext, flush timing, LazyInit, OSIV internals с deep dive в Hibernate mechanisms.
