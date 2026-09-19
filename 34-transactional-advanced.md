# 34. @Transactional продвинутое: rollback, listeners, savepoints, testing

Правила rollback, `TransactionSynchronization`, `@TransactionalEventListener`, savepoints, timeouts, тестирование.

---

## 1. Rollback rules — самое коварное

### 1.1 Правила по умолчанию

Spring откатывает транзакцию **только на unchecked exceptions**:
- `RuntimeException` и потомки → **rollback**.
- `Error` → **rollback**.
- **Checked exceptions** (Exception и не-Runtime потомки) → **commit** (несмотря на exception!).

Классический баг:
```java
@Transactional
public void save(Order o) throws IOException {
    repo.save(o);
    externalCall();       // бросает IOException
}
// IOException — checked → tx COMMITтится! Ордер сохранён.
```

Ожидание: rollback. Реальность: commit.

### 1.2 rollbackFor

Явно указать какие исключения → rollback:
```java
@Transactional(rollbackFor = Exception.class)
public void save(Order o) throws IOException {
    ...
}
```

Теперь любое `Exception` (включая checked) → rollback.

Или конкретно:
```java
@Transactional(rollbackFor = {IOException.class, TimeoutException.class})
```

### 1.3 noRollbackFor

Не откатывать на конкретные исключения:
```java
@Transactional(noRollbackFor = ExpectedBusinessException.class)
public void process() { ... }
```

Даже если бросит `ExpectedBusinessException` → commit.

### 1.4 Best practice

**Правило**: `@Transactional(rollbackFor = Exception.class)` — универсально безопасно.

Или полностью custom `BusinessException` иерархию:
```java
public class BusinessException extends RuntimeException { ... }

@Transactional  // default правила ок, потому что все свои исключения — Runtime
```

### 1.5 Ручной setRollbackOnly

Иногда нужно откатить без бросания exception:

```java
@Autowired
TransactionStatus status;    // не работает, статус привязан к текущей tx

// правильно — через TransactionAspectSupport
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
```

Или (лучше) — TransactionTemplate:
```java
tx.execute(status -> {
    if (badCondition) {
        status.setRollbackOnly();
        return null;
    }
    // work
});
```

### 1.6 Что если пометил rollbackOnly, но не бросил exception

Tx помечена. Внешняя транзакция → на выходе всё равно rollback (даже если она REQUIRED).

**Кавет UnexpectedRollbackException**: если внутренний REQUIRED-метод пометил rollbackOnly, а внешний не бросил exception → на commit внешней Spring увидит помеченный rollback → бросит `UnexpectedRollbackException`. Приложение может не ожидать этого.

---

## 2. readOnly

```java
@Transactional(readOnly = true)
public List<Order> list() { ... }
```

Что делает:
1. **JDBC уровень**: `conn.setReadOnly(true)` → PostgreSQL может оптимизировать read-only tx.
2. **JPA уровень**: Hibernate ставит `FlushMode.MANUAL` → не будет dirty checking + автоматический flush.
3. **Явно документирует** намерение.

Плюсы:
- Меньше памяти (без snapshot для сравнения).
- Быстрее (без flush).
- Некоторые БД оптимизируют.

**Правило**: на всех read-only методах → `readOnly = true`.

Кавет: если внутри случайно `setter` на managed-сущности → изменение НЕ сохранится (нет flush). Молча теряется.

---

## 3. Timeout

```java
@Transactional(timeout = 30)   // секунды
public void longOperation() { ... }
```

Через N секунд tx автоматически откатывается.

Реализация:
- JDBC: `conn.setQueryTimeout(30)` — на каждый statement.
- JPA: `em.createQuery(...).setHint("javax.persistence.query.timeout", 30000)`.

**Кавет**: timeout не всегда работает как ожидаешь. Часть операций (например, ожидание блокировки) может игнорировать. Полагайся на statement_timeout / lock_timeout на уровне PG:

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';
```

Или в yml:
```yaml
spring.datasource.hikari.data-source-properties:
  socketTimeout: 30       # мaximum на statement
```

---

## 4. Savepoints (NESTED propagation)

Когда одна tx хочет «попробовать и откатить кусок»:

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

### 4.1 Как работает

При вызове NESTED в существующей tx:
1. Spring создаёт savepoint (`SAVEPOINT sp1`).
2. Выполняется код.
3. Success → savepoint удаляется, изменения остаются в tx.
4. Exception → `ROLLBACK TO SAVEPOINT sp1` → внешняя tx продолжается.

### 4.2 Требования

- БД должна поддерживать savepoints (PostgreSQL — да, Oracle — да, MySQL InnoDB — да).
- В Spring: `DataSourceTransactionManager.setNestedTransactionAllowed(true)` (default true).
- JpaTransactionManager с Hibernate — работает.

### 4.3 Разница NESTED vs REQUIRES_NEW

- **NESTED** — один Connection, один tx с savepoint.
- **REQUIRES_NEW** — новый Connection, независимая tx.

Кавет — если внешняя откатывается, NESTED тоже откатывается (внутри одной tx). REQUIRES_NEW — независимая, коммитится отдельно.

---

## 5. TransactionSynchronization — хуки

Spring позволяет подписаться на события транзакции.

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

### 5.1 События

- **beforeCommit** — до commit. Можно бросить exception → tx откатится.
- **beforeCompletion** — до close (commit или rollback).
- **afterCommit** — после успешного commit. **Ошибки логируются, но не влияют на tx**.
- **afterCompletion(status)** — после close. status = STATUS_COMMITTED / STATUS_ROLLED_BACK / STATUS_UNKNOWN.

### 5.2 Use cases

- **afterCommit** — публикация события в Kafka/Rabbit **только если tx успешно закоммитилась**. Логика outbox.
- **afterCompletion** — очистка ресурсов, метрики.

```java
@Transactional
public void save(Order o) {
    repo.save(o);
    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronizationAdapter() {
            @Override
            public void afterCommit() {
                publisher.publish(o);   // публикуем только после commit
            }
        });
}
```

### 5.3 @TransactionalEventListener — правильный способ

Более удобно через события Spring:

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

Фазы:
- `BEFORE_COMMIT` — до commit.
- `AFTER_COMMIT` — после success.
- `AFTER_ROLLBACK` — после rollback.
- `AFTER_COMPLETION` — после (commit или rollback).

Обычный `@EventListener` (без Transactional) — срабатывает **синхронно** при publishEvent, до commit.

### 5.4 Кавет: событие потеряется если нет tx

`@TransactionalEventListener` **работает только в активной tx**. Если событие опубликовано вне @Transactional — listener НЕ вызовется.

Можно разрешить fallback:
```java
@TransactionalEventListener(fallbackExecution = true)
```

Тогда без tx выполняется как обычный @EventListener.

---

## 6. Async — обрабатывать вне tx

Опубликовать событие, дальше обработать асинхронно:

```java
@Component
class OrderEventListener {
    @Async                       // + @EnableAsync
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void handle(OrderCreatedEvent event) {
        // выполнится в отдельном потоке, вне tx
    }
}
```

Порядок:
1. Original method → save + publishEvent → commit.
2. Spring вызывает listener на другом потоке.
3. Даже если listener упадёт → не откатит уже commit'нутую tx.

---

## 7. Программные транзакции — TransactionTemplate

Уже упоминал в предыдущем файле. Детально:

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

Плюсы vs `@Transactional`:
- Динамический контроль.
- `setRollbackOnly` без бросания exception.
- Гибкая настройка per-invocation.
- Легче unit-тестить (можно замокать).

Минусы:
- Многословнее.
- Нельзя декларативно (аннотации виднее).

---

## 8. Тестирование транзакций

### 8.1 @Transactional в тестах = автоматический rollback

Spring Test автоматически:
1. Начинает tx перед каждым @Test.
2. Откатывает после.

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
        // после теста → rollback → БД чистая
    }
}
```

Плюсы: не нужен @BeforeEach cleanup.

**Кавет 1 — REQUIRES_NEW в тесте**: если код внутри создаёт новую tx (REQUIRES_NEW) — она НЕ откатится с тестовой tx. Тестовая откатит только свою.

**Кавет 2 — тест видит state внутри tx**: `assertThat(repo.findAll())` работает потому что тестовая tx та же что и `svc.createOrder`. Если тест делает `@Async`/новый поток — тестовая tx не пробросится → изменения не увидит.

### 8.2 @Commit — не откатывать

```java
@Test
@Commit
void keepDataForDebug() { ... }
```

БД сохранит изменения после теста. Для debug.

### 8.3 @Sql для setup

```java
@Test
@Sql("/setup-orders.sql")
void test() { ... }
```

### 8.4 Тестировать без Spring контекста (unit)

Мокать репозиторий:
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

Быстро (нет Spring), но нет проверки самой tx.

---

## 9. Distributed транзакции (краткое напоминание)

Из `32-transactions-acid-isolation-propagation.md`: **не используй XA в микросервисах**. Правильно:

- **Outbox pattern** для atomic БД + Kafka/Rabbit.
- **Saga** для multi-service tx.

Spring поддержки Saga из коробки нет — есть внешние библиотеки (Camunda, Axon Framework).

---

## 10. JTA (Java Transaction API) — краткий обзор

Стандарт для distributed tx через 2PC.

- `UserTransaction` — управление вручную.
- `TransactionManager` — уровень контейнера.
- `XAResource` — интерфейс участника.

В Spring Boot: `JtaTransactionManager` + XA-provider (Atomikos, Bitronix, Narayana).

**В микросервисах не используется**. В legacy monolith на JEE app-server — было.

---

## 11. Специфичные вещи для JPA

### 11.1 Flush time

`@Transactional` метод:
1. Изменения managed-объектов накапливаются.
2. При commit — `em.flush()` → SQL.
3. `conn.commit()`.

Ошибка на flush (constraint violation, staleObject) — throw exception на границе tx-метода, не в момент set-а.

**Кавет debug**: `setter` вроде работает, но на выходе метода — `ConstraintViolationException`. Причина — flush.

### 11.2 Optimistic lock exception

`@Version` — при UPDATE Hibernate проверяет version. Не совпадает → `OptimisticLockException`.

В @Transactional-методе → rollback + throw. Приложение должно handle (retry / show conflict user).

### 11.3 Multi-tenancy

Иногда одна tx работает с несколькими БД (например, каждый tenant — своя БД). Стандарт JPA — schema-based или database-based multi-tenancy через `MultiTenantConnectionProvider`. Spring поддерживает.

---

## 12. Диагностика проблем

### 12.1 «Изменения не сохраняются»

Проверить:
- Метод public?
- Класс — Spring bean (не new)?
- `@Transactional` есть?
- Не self-invocation?
- Не бросается checked exception без `rollbackFor`?
- Не `readOnly = true` случайно?
- Не `@Async` (session в другом потоке)?

### 12.2 «UnexpectedRollbackException»

Внутренний REQUIRED пометил rollback-only. Внешний не знает, продолжает — на commit ошибка.

Fix:
- Использовать REQUIRES_NEW для «независимых» операций.
- Или явно проверять / бросать exception.

### 12.3 «Connection pool exhausted»

Причины:
- REQUIRES_NEW внутри REQUIRED — держит 2 connection.
- Внешние API внутри tx — connection занят долго.
- Утечка connection (не вернулся в пул).

`leak-detection-threshold` в HikariCP.

### 12.4 «Deadlock»

Две tx ждут блокировки друг друга.

Fix:
- Всегда брать locks в одном порядке.
- Уменьшить длину tx.
- Использовать optimistic lock вместо pessimistic.

---

## 13. Полная best-practice тx-настройка

```java
@Configuration
@EnableTransactionManagement
public class TxConfig {
    // Spring Boot всё автоконфигурит для JPA;
    // это только если нужен custom
}

@Service
class OrderService {

    @Autowired OrderRepository repo;
    @Autowired ApplicationEventPublisher events;

    // read
    @Transactional(readOnly = true, timeout = 5)
    public Order findById(Long id) {
        return repo.findById(id).orElseThrow();
    }

    // write
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

---

## 14. Собесные вопросы

1. **Что откатывается по умолчанию?** — Только RuntimeException + Error; checked exceptions → commit.
2. **Как откатывать на checked exception?** — `@Transactional(rollbackFor = Exception.class)`.
3. **`readOnly = true` — что даёт?** — JDBC read-only, Hibernate MANUAL flush, БД оптимизации.
4. **Как откатить без exception?** — `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` или TransactionTemplate + status.setRollbackOnly.
5. **Что такое UnexpectedRollbackException?** — Внутренняя tx пометила rollback-only, внешняя пыталась commit → exception.
6. **NESTED vs REQUIRES_NEW?** — NESTED = savepoint в одной tx; REQUIRES_NEW = отдельная tx (новый connection).
7. **Как выполнить логику после commit?** — `@TransactionalEventListener(phase = AFTER_COMMIT)` или TransactionSynchronization.afterCommit.
8. **@EventListener vs @TransactionalEventListener?** — Первый sync до commit; второй с фазой (обычно AFTER_COMMIT).
9. **TransactionTemplate — когда?** — Программный контроль, динамические настройки, `setRollbackOnly` без exception.
10. **@Transactional в тестах?** — Spring Test автоматически откатывает после теста.
11. **@Async @TransactionalEventListener — как работают вместе?** — Listener выполняется в другом потоке, вне исходной tx.
12. **Что такое JTA?** — Стандарт distributed tx через 2PC; в микросервисах избегай.
13. **Timeout в @Transactional — что делает?** — Ограничивает время tx; JDBC statement timeout под капотом.
14. **Проблемы с REQUIRES_NEW?** — Требует два connection одновременно → истощение пула + возможные deadlock.
15. **Как правильно отправить событие в Kafka после save?** — TransactionalEventListener(AFTER_COMMIT) + outbox pattern.

---

## Итог

- **rollbackFor = Exception.class** — универсально безопасно (иначе checked не откатываются).
- **readOnly = true** на всех read-методах.
- **@TransactionalEventListener(AFTER_COMMIT)** для sending events / notifications.
- **NESTED** для «попытки с откатом» без потери tx.
- **REQUIRES_NEW** осторожно — 2 connections одновременно.
- **TransactionTemplate** — гибкий программный контроль.
- **@Transactional в тестах** — auto rollback.
- **Async listeners** для не-критичной пост-обработки.

Следующий — `35-transactional-jpa-persistence-context.md`.
