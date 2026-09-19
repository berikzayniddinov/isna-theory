# 34. @Transactional продвинутое: rollback, listeners, savepoints, testing

## Rollback rules самое коварное

Наиболее контр-интуитивное поведение @Transactional связано с rollback правилами. Spring откатывает транзакцию только на unchecked exceptions по default. RuntimeException и потомки приводят к rollback. Error класс также приводит к rollback. Checked exceptions (Exception и не-Runtime потомки) приводят к COMMIT транзакции несмотря на брошенное exception.

Классический баг:
```java
@Transactional
public void save(Order o) throws IOException {
    repo.save(o);
    externalCall();       // бросает IOException
}
```

Ожидаемое поведение — при IOException транзакция откатывается, order не сохраняется. Реальность — IOException это checked exception, транзакция COMMIT-ится, order остаётся в БД. Только потом exception пробрасывается наверх. Полная mess-состояние — операция «упала» но частично сохранилась.

Историческая причина такого поведения из Java EE традиции где checked exceptions считались business exceptions (recoverable) а unchecked считались system exceptions (unrecoverable). Business exception подразумевало что произошёл ожидаемый alternative flow — не системная ошибка. Практически это правило редко подходит и создаёт больше проблем чем решает.

rollbackFor атрибут явно указывает какие exceptions приводят к rollback:
```java
@Transactional(rollbackFor = Exception.class)
public void save(Order o) throws IOException {
    // ...
}
```

Теперь любое Exception (включая checked IOException) приводит к rollback. Rollback rules расширены и включают всё что наследуется от Exception.

Возможна конкретика — указать specific exception types:
```java
@Transactional(rollbackFor = {IOException.class, TimeoutException.class})
```

Только IOException и TimeoutException приводят к rollback среди checked exceptions. Другие checked exceptions по-прежнему приводят к commit.

noRollbackFor исключает specific exceptions из rollback:
```java
@Transactional(noRollbackFor = ExpectedBusinessException.class)
public void process() { ... }
```

Даже если бросается ExpectedBusinessException (RuntimeException) — transaction commits. Полезно для «ожидаемых» exceptions которые не должны откатывать transaction.

Best practice универсально безопасный approach — @Transactional(rollbackFor = Exception.class) для всех методов. Полностью удаляет surprise-behavior с checked exceptions.

Альтернатива — полностью custom BusinessException иерархия наследующаяся от RuntimeException:
```java
public class BusinessException extends RuntimeException { ... }

@Transactional  // default rules ок, все свои exceptions Runtime
```

Если все свои exceptions RuntimeException-based, default rollback rules работают правильно. Consistent approach через всё codebase.

Ручной setRollbackOnly когда нужно откатить transaction без бросания exception. Через TransactionAspectSupport:
```java
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
```

Или через TransactionTemplate:
```java
tx.execute(status -> {
    if (badCondition) {
        status.setRollbackOnly();
        return null;
    }
    // работа
});
```

Полезно когда нужен rollback based на условии не приводящем к exception. Например validation в service returns error result вместо throwing.

UnexpectedRollbackException это специфический caveat. Если внутренний REQUIRED метод пометил rollbackOnly а внешний метод не бросил exception — Spring на commit external transaction увидит что transaction помечена rollbackOnly и бросит UnexpectedRollbackException. Приложение может не ожидать этого exception если написано без учёта такого поведения.

## readOnly оптимизация

Атрибут readOnly даёт несколько оптимизаций для read-only операций:
```java
@Transactional(readOnly = true)
public List<Order> list() { ... }
```

Что происходит на разных уровнях. JDBC level — Connection.setReadOnly(true), PostgreSQL может оптимизировать read-only transactions используя менее restrictive locking. JPA level — Hibernate устанавливает FlushMode.MANUAL, не будет dirty checking plus automatic flush. Явно документирует намерение code — читатель видит что метод не изменяет данные.

Плюсы. Меньше memory footprint потому что нет snapshot для dirty checking comparison. Быстрее потому что нет flush operations. Некоторые БД оптимизируют read-only transactions (например могут использовать read replicas или дополнительный parallelism).

Правило универсальное — на всех read-only методах ставь readOnly = true. Как code review pattern облегчает понимание кода.

Важный caveat — если в readOnly методе случайно выполняется setter на managed entity, изменение НЕ будет сохранено потому что нет flush. Молча теряется. Может быть сложным bug если полагаться на «случайное» изменение. Discipline — read-only методы действительно только читают.

## Timeout

Атрибут timeout ограничивает время transaction:
```java
@Transactional(timeout = 30)   // секунды
public void longOperation() { ... }
```

Через N секунд transaction автоматически откатывается — timeout exception генерируется.

Реализация зависит от resource. JDBC — Connection.setQueryTimeout(30) устанавливает на каждый statement max execution time. JPA — em.createQuery.setHint("javax.persistence.query.timeout", 30000) для каждого query.

Caveat timeout не всегда работает как ожидается. Часть операций (например ожидание блокировки) может игнорировать statement timeout. Более надёжно установить statement_timeout и lock_timeout на уровне PostgreSQL:
```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';
```

Или через свойства pool:
```yaml
spring.datasource.hikari.data-source-properties:
  socketTimeout: 30       # максимум на statement
```

## Savepoints через NESTED propagation

NESTED propagation позволяет попытаться операцию с возможным откатом без потери всей transaction:
```java
@Service
class OrderService {
    @Transactional
    public void createOrder(Order o) {
        repo.save(o);
        try {
            audit.log(o);        // может упасть — не критично
        } catch (Exception e) {
            log.warn("audit failed", e);
        }
    }
}

@Service
class AuditService {
    @Transactional(propagation = Propagation.NESTED)
    public void log(Order o) {
        // если бросит — откатится до savepoint,
        // внешняя createOrder продолжится
    }
}
```

Механизм работы. При вызове NESTED в существующей transaction Spring создаёт savepoint через SAVEPOINT sp1 SQL команду. Выполняется код метода. Success — savepoint удаляется, изменения остаются в transaction. Exception — ROLLBACK TO SAVEPOINT sp1, внешняя transaction продолжается.

Требования. БД должна поддерживать savepoints — PostgreSQL, Oracle, MySQL InnoDB поддерживают. В Spring — DataSourceTransactionManager.setNestedTransactionAllowed(true) (default true). JpaTransactionManager с Hibernate работает.

Различие NESTED и REQUIRES_NEW. NESTED использует один Connection, одну transaction с savepoint внутри. REQUIRES_NEW создаёт новый Connection и полностью независимую transaction. Если внешняя откатывается NESTED тоже откатывается (внутри одной transaction). REQUIRES_NEW независима и коммитится отдельно.

## TransactionSynchronization хуки

Spring позволяет подписаться на события transaction lifecycle:
```java
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronization() {
        @Override
        public void beforeCommit(boolean readOnly) { ... }
        @Override
        public void beforeCompletion() { ... }
        @Override
        public void afterCommit() { ... }
        @Override
        public void afterCompletion(int status) { ... }
    });
```

События. beforeCommit — до commit. Можно бросить exception чтобы откатить transaction. beforeCompletion — до close (commit или rollback). afterCommit — после успешного commit. Ошибки логируются но не влияют на transaction (она уже commited). afterCompletion(status) — после close. Status равен STATUS_COMMITTED, STATUS_ROLLED_BACK, STATUS_UNKNOWN.

Use cases. afterCommit для публикации event в Kafka или Rabbit только если transaction успешно закоммитилась. Реализует логику outbox — не публикуем event пока не commit. afterCompletion для cleanup ресурсов, metrics.

Пример:
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

## @TransactionalEventListener правильный способ

Более удобный подход через Spring events:
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
        publisher.publish(event);   // только после commit
    }
}
```

Публикатор просто вызывает publishEvent — сохраняется событие связанное с transaction. Listener с @TransactionalEventListener выполняется в указанной phase.

Фазы. BEFORE_COMMIT до commit — можно cancel transaction через exception. AFTER_COMMIT после success. AFTER_ROLLBACK после rollback. AFTER_COMPLETION после (commit или rollback).

Обычный @EventListener (без Transactional) срабатывает синхронно при publishEvent, до commit. Проблема если событие приводит к side effects (например отправка email) до commit — при откате side effect уже произошёл. TransactionalEventListener решает этой проблему.

Caveat — событие теряется если нет активной transaction. @TransactionalEventListener работает только в активной transaction. Если событие опубликовано вне @Transactional listener не вызовется. Можно разрешить fallback:
```java
@TransactionalEventListener(fallbackExecution = true)
```

Тогда без transaction выполняется как обычный @EventListener.

## Async обработка вне transaction

Комбинация @Async и @TransactionalEventListener позволяет обрабатывать события в отдельном thread:
```java
@Component
class OrderEventListener {
    @Async                       // + @EnableAsync в конфигурации
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void handle(OrderCreatedEvent event) {
        // выполнится в отдельном thread, вне transaction
    }
}
```

Порядок. Original method делает save plus publishEvent, потом commit. Spring вызывает listener в другом thread. Даже если listener упадёт не откатит уже commit-нутую transaction.

Полезно для heavy post-processing которое не должно блокировать main transaction path. Отправка email, обновление кэшей, генерация reports — все хорошие candidates.

## Программные транзакции TransactionTemplate

Уже упоминался кратко в предыдущем файле. Detailed usage:
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

Плюсы vs @Transactional. Динамический контроль — можно менять propagation, isolation, timeout на runtime. setRollbackOnly без бросания exception. Гибкая настройка per-invocation. Легче unit-тестить потому что transaction manager можно замокать.

Минусы. Многословнее чем аннотация. Нельзя декларативно — сложнее понять глядя на класс что метод transactional.

Практически TransactionTemplate используется для сценариев где @Transactional недостаточен. Bulk processing где каждый item в своей transaction и failure одного не блокирует другие. Dynamic configuration transaction based на runtime conditions.

## Тестирование transactions

@Transactional в тестах даёт автоматический rollback. Spring Test автоматически начинает transaction перед каждым @Test и откатывает после:
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
        // после теста rollback → БД чистая
    }
}
```

Плюсы — не нужен @BeforeEach cleanup, tests isolated automatically.

Caveat 1 REQUIRES_NEW в тесте. Если код внутри создаёт new transaction (REQUIRES_NEW) она НЕ откатится с тестовой transaction. Тестовая откатит только свою. REQUIRES_NEW transaction commits independently, test cleanup ей не поможет.

Caveat 2 тест видит state внутри transaction. assertThat(repo.findAll()) работает потому что тестовая transaction та же что и svc.createOrder — они share PersistenceContext. Если тест делает @Async или создаёт новый thread — тестовая transaction не пробросится, изменения не увидит.

@Commit явно указывает не откатывать transaction после теста:
```java
@Test
@Commit
void keepDataForDebug() { ... }
```

БД сохранит изменения после теста. Для debugging когда хочется inspect state после test.

@Sql для setup данных перед тестом:
```java
@Test
@Sql("/setup-orders.sql")
void test() { ... }
```

Полезно для загрузки test fixtures из SQL файлов.

Unit tests без Spring context быстрее. Мокать репозиторий:
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

Быстро (нет Spring boot) но нет проверки самой transaction. Комбинация unit tests для business logic plus integration tests для transaction behavior стандартная стратегия.

## Distributed transactions

Из файла 32 напоминание — не используй XA в микросервисах. Правильно outbox pattern для atomic БД plus Kafka/Rabbit. Saga для multi-service transactions с compensation.

Spring поддержки Saga из коробки нет — есть внешние библиотеки Camunda, Axon Framework для orchestration-based Saga. Choreography-based Saga через event-driven architecture можно реализовать вручную с RabbitMQ или Kafka.

## JTA обзор

Java Transaction API стандарт для distributed transactions через 2PC. UserTransaction для manual управления. TransactionManager для container-level. XAResource interface для resource providers.

В Spring Boot — JtaTransactionManager plus XA provider (Atomikos, Bitronix, Narayana). Configuration сложна. Redko используется в микросервисах потому что 2PC has performance и complexity issues.

В legacy monolith на JEE application server (WebLogic, WebSphere) XA был стандартом. Modern microservices moved away от XA. Осталось только для legacy migrations и specific enterprise environments.

## Специфика для JPA

Flush time важен для понимания where errors происходят. @Transactional method flow. Изменения managed objects накапливаются в PersistenceContext. При commit em.flush() генерирует SQL. conn.commit() фиксирует. Ошибка на flush (constraint violation, staleObject) throws exception на границе transaction method не в момент setter вызова.

Caveat в debugging. setter вроде работает без ошибки, но на выходе метода ConstraintViolationException. Причина — flush происходит в конце method. Явный em.flush() внутри может помочь ловить ошибку earlier.

Optimistic lock exception через @Version. При UPDATE Hibernate проверяет version column. Если не совпадает бросает OptimisticLockException. В @Transactional-методе — rollback plus throw. Приложение должно обработать — retry или show conflict пользователю.

Multi-tenancy иногда одна transaction работает с несколькими БД (например каждый tenant своя БД). JPA стандарт — schema-based или database-based multi-tenancy через MultiTenantConnectionProvider. Spring поддерживает.

## Диагностика проблем

«Изменения не сохраняются» — стандартный чеклист. Метод public? Класс Spring bean не new? @Transactional присутствует? Не self-invocation? Не бросается checked exception без rollbackFor? Не readOnly = true случайно? Не @Async (session в другом thread)?

«UnexpectedRollbackException» — внутренний REQUIRED пометил rollback-only. Внешний не знает и пытается continue — на commit ошибка. Fix через REQUIRES_NEW для «независимых» операций или явно проверять и бросать exception.

«Connection pool exhausted» — причины. REQUIRES_NEW внутри REQUIRED держит 2 connections одновременно. Внешние API внутри transaction connection занят долго. Утечка connection (не возвращён в pool). leak-detection-threshold в HikariCP помогает локализовать.

«Deadlock» — две transactions ждут блокировки друг друга. Fix через consistent ordering locks (всегда by id ascending), уменьшение длины transactions, использование optimistic locking вместо pessimistic где возможно.

## Полная best-practice конфигурация

Собранная воедино правильная transaction setup:
```java
@Configuration
@EnableTransactionManagement
public class TxConfig {
    // Spring Boot автоконфигурит всё для JPA
    // это только если нужен custom
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

Ключевые элементы. @Transactional(readOnly = true, timeout = N) на read methods. @Transactional(rollbackFor = Exception.class, timeout = N) на write methods. Events через publishEvent plus @TransactionalEventListener(AFTER_COMMIT) для async post-processing. @Async на listener чтобы не блокировать main transaction.

## Итоги

По default только RuntimeException и Error приводят к rollback. Checked exceptions приводят к COMMIT несмотря на exception. Используй rollbackFor = Exception.class универсально безопасно или всё через RuntimeException-based иерархию.

readOnly = true даёт JDBC read-only mode, Hibernate MANUAL flush, database оптимизации. Обязательно на всех read-only methods. Caveat — случайные setter в readOnly method silently не сохраняются.

Timeout ограничивает transaction execution time. JDBC statement timeout под капотом. Более надёжно через statement_timeout в PostgreSQL напрямую.

Savepoints через NESTED propagation для «попыток с откатом» без потери всей transaction. Один Connection в отличие от REQUIRES_NEW который требует два.

TransactionSynchronization и @TransactionalEventListener для callbacks на transaction events. AFTER_COMMIT типичный для publishing events after successful transaction.

@Async на @TransactionalEventListener для async обработки events в другом thread. Не блокирует main transaction path.

TransactionTemplate для программного transaction control когда declarative @Transactional недостаточен. Dynamic configuration per-invocation.

@Transactional в тестах даёт automatic rollback. Caveat REQUIRES_NEW не откатывается с test transaction. @Commit для explicit commit в тестах. @Sql для test fixtures.

XA/JTA избегать в микросервисах — Saga и Outbox pattern предпочтительнее.

Дальше — @Transactional plus JPA специфика с PersistenceContext, flush, LazyInit и OSIV.
