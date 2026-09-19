# 35. @Transactional + JPA: PersistenceContext, flush, LazyInit

Как `@Transactional` работает с JPA конкретно. Что происходит с EntityManager внутри tx.

---

## 1. Ключевое связывание: tx = PersistenceContext

При старте `@Transactional`-метода Spring:

1. Открывает JPA-транзакцию через `JpaTransactionManager`.
2. Создаёт (или берёт из пула) **EntityManager**.
3. Привязывает его к текущему потоку через `TransactionSynchronizationManager`.
4. Все `@PersistenceContext EntityManager em;` в коде получают этот же EM (proxy'ed).

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

### 1.1 SharedEntityManagerCreator

Когда Spring инжектит:
```java
@PersistenceContext
private EntityManager em;
```

Ты получаешь **не настоящий EntityManager**, а **прокси** (`SharedEntityManagerCreator`). При каждом вызове он:
- Смотрит thread-local — есть ли связанный EM?
- Да → возвращает связанный (тот же для всей tx).
- Нет → создаёт временный на один вызов (не рекомендуется, т.к. вне tx).

Отсюда: **вне @Transactional операции с EM могут не работать** (или работать только на одну операцию).

---

## 2. Три сценария вызова

### 2.1 Внутри @Transactional

```java
@Transactional
public void save(Order o) {
    em.persist(o);              // managed
    o.setStatus(NEW);           // dirty
    // выход → flush → INSERT + UPDATE (single tx)
}
```

Работает как ожидается.

### 2.2 Вне @Transactional (в web-контроллере с open-in-view)

Spring Boot по умолчанию включает **Open-Session-In-View (OSIV)** — session открыта на весь HTTP-запрос.

```java
// application.yml
spring.jpa.open-in-view: true    # default!
```

Что делает: `OpenEntityManagerInViewInterceptor` открывает EM в начале HTTP-запроса, закрывает в конце. Внутри — есть session, LazyInit работает.

**Проблемы**:
- Скрывает N+1 (SELECT'ы летят из контроллера).
- Держит DB connection на весь запрос (если сервис делает внешние вызовы — connection зря простаивает).
- Скрывает архитектурные ошибки (Entity утекает в контроллер).

**Правильно**: `spring.jpa.open-in-view: false` + аккуратный дизайн (сервис возвращает DTO).

### 2.3 Вне @Transactional и без OSIV

`em.find(...)` вернёт detached-объект. `em.persist(...)` бросит `TransactionRequiredException`.

Это правильно — операции с БД должны быть в tx.

---

## 3. Flush — когда именно

Внутри @Transactional-метода Hibernate накапливает изменения (dirty checking + persist/remove). Реальные SQL летят в БД на **flush**.

### 3.1 Автоматический flush

Hibernate flushMode = AUTO (по умолчанию):
1. **Перед commit** — обязательно.
2. **Перед выполнением query** — чтобы query увидел изменения текущей tx.
3. **Иногда** перед `find()` (если сущность изменена).

Пример:
```java
@Transactional
public void updateAndCount() {
    Order o = em.find(Order.class, 1L);
    o.setStatus(NEW);              // dirty, не в БД

    long count = em.createQuery("SELECT count(o) FROM Order o WHERE o.status = :s", Long.class)
        .setParameter("s", NEW)
        .getSingleResult();
    // ← перед этим query — flush → UPDATE + SELECT

    // на выходе — flush (уже сделан), commit
}
```

### 3.2 Manual flush

Явно:
```java
em.flush();       // отправить SQL сейчас (для проверки, для получения ID)
```

Полезно:
- Проверить что INSERT прошёл валидацию БД до конца метода.
- Получить `@GeneratedValue(IDENTITY)` id немедленно.

### 3.3 FlushMode

- `AUTO` — auto flush (default).
- `COMMIT` — только на commit. Query может вернуть устаревшие данные.
- `MANUAL` — только явный `em.flush()`. Настраивается для read-only оптимизации.

### 3.4 readOnly и flush

`@Transactional(readOnly = true)` в Hibernate:
```
Session.setDefaultReadOnly(true)
Session.setFlushMode(FlushMode.MANUAL)
```

Отсюда:
- Нет dirty checking → нет UPDATE даже если ты изменил поле.
- Изменение молча теряется.

**Кавет**: если в read-only методе случайно `entity.setX(...)` — ничего не будет. Не паникуй когда «изменение не сохранилось» — проверь readOnly.

---

## 4. LazyInitializationException — глубже

Уже обсуждали в `15-jpa-performance.md`. Здесь связь с tx.

```java
@Transactional(readOnly = true)
public Order load(Long id) {
    return repo.findById(id).orElseThrow();
}

// controller:
Order o = svc.load(1L);
o.getItems().forEach(...);         // ← LazyInit!
```

### 4.1 Причина

`orderItems` — LAZY коллекция. Прокси (`PersistentBag`). При обращении → пытается SELECT.

Но session закрыта (tx закончилась) → нет откуда SELECT → **LazyInitializationException**.

### 4.2 Решения

**A) DTO в сервисе** (правильно):
```java
@Transactional(readOnly = true)
public OrderDto load(Long id) {
    Order o = repo.findById(id).orElseThrow();
    return new OrderDto(
        o.getId(),
        o.getItems().stream().map(...).toList()   // загружаем внутри tx
    );
}
```

**B) JOIN FETCH** / **EntityGraph** — форсированно загрузить перед закрытием session.

**C) OSIV** (плохо) — session до конца HTTP-запроса.

**D) FetchType.EAGER** (ужасно) — всегда загружать.

### 4.3 Hibernate 6 стало строже

С Hibernate 6 (Spring Boot 3) LazyInit ловится чаще. Раньше некоторые случаи молчали.

Реальный кейс ИСНА `knp-fo-sync-notification-bugs`:
- `@Transactional` был мёртв из-за self-invocation.
- Значит session никогда не открывалась.
- Hib6 стал жёстче → LazyInit проявился (Hib5 молчал).

Урок: **не полагаться на «работает в Hib5»**. Правильные tx + DTO.

### 4.4 `getById` vs `findById`

Уже разбирали. Напоминание:
- `findById(id)` → `Optional<T>` → SELECT сразу.
- `getReferenceById(id)` → `T` → **прокси без SELECT** → LazyInit если использовать вне tx.

Практика: `getReferenceById` только для «мне нужна ссылка не читая» (например, для FK).

---

## 5. Cascade и tx

```java
@Entity
class Order {
    @OneToMany(mappedBy="order", cascade = ALL, orphanRemoval = true)
    List<OrderItem> items;
}
```

Cascade расширяет операции persist/remove на связанные объекты — **всё в одной tx**.

```java
@Transactional
public void createOrder(Order o) {
    o.setItems(List.of(item1, item2, item3));
    repo.save(o);
    // на flush → INSERT Order + 3 × INSERT OrderItem — одна tx
}
```

Если что-то не пройдёт валидацию БД → rollback всей tx (никаких «половинных» состояний).

---

## 6. Bulk operations

```java
@Modifying
@Query("UPDATE Order o SET o.status = :new WHERE o.status = :old")
int bulkUpdate(...);
```

Bulk UPDATE **не проходит через PersistenceContext**. Managed-объекты в текущей сессии остаются со старым значением.

Правило после bulk:
```java
em.flush();   // отправить всё в БД до bulk
em.clear();   // очистить кэш
repo.bulkUpdate(OLD, NEW);
// managed-объекты теперь detached
```

Или изолировать в отдельном сервисе / tx.

---

## 7. Transaction propagation в JPA

Практика для типовых кейсов.

### 7.1 REQUIRED (default)

Обычный сервис:
```java
@Service
class OrderService {
    @Transactional
    public void save(Order o) { ... }

    @Transactional
    public void update(Order o) { ... }
}
```

Всё в своих tx.

### 7.2 REQUIRES_NEW для independent audit

```java
@Service
class OrderService {
    @Autowired AuditService audit;

    @Transactional
    public void createOrder(Order o) {
        repo.save(o);
        audit.log("Order created", o);   // отдельная tx
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

Даже если Order сохранился, а audit упал — audit-запись сохранится независимо.

**Кавет**: 2 connections в пуле одновременно. При нагрузке пул истощается.

### 7.3 NESTED для «попыток»

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

### 7.4 SUPPORTS для «может быть в tx, может не быть»

```java
@Transactional(propagation = SUPPORTS)
public List<Order> list() {
    // если снаружи tx — участвуем;
    // если нет — работаем без tx (одиночные SELECT)
    return repo.findAll();
}
```

Редко полезно. Обычно `readOnly = true` + REQUIRED достаточно.

---

## 8. Session per request (что не путать)

**OSIV** = session per HTTP-request.
**@Transactional** = session per tx.

С OSIV **включённым**: session открыта весь запрос, tx открывается внутри при @Transactional-методах. По окончании tx — session НЕ закрывается (продолжается до конца HTTP-запроса).

С OSIV **выключённым**: session открыта ТОЛЬКО в @Transactional. Вне — нет.

**Правильно**: OSIV **OFF** + все взаимодействия с БД внутри @Transactional-сервисов + возвращать DTO.

---

## 9. Долгие транзакции — почему плохо

```java
@Transactional
public void process(Order o) {
    repo.save(o);
    externalApi.call(o);        // ждём 30 сек
    audit.log(o);
}
```

30 сек tx open:
- Connection в пуле занят → пул истощается.
- Row locks удерживаются → deadlock растёт.
- Long-running tx блокирует VACUUM → bloat.
- В PgBouncer transaction-mode → connection не возвращается в pool.

**Правило**: **внешние вызовы вне @Transactional**.

```java
public void process(Order o) {
    Order saved = doSave(o);              // короткая tx
    externalApi.call(saved);              // снаружи tx
    doAudit(saved);                       // отдельная tx
}

@Transactional
Order doSave(Order o) { return repo.save(o); }
@Transactional
void doAudit(Order o) { ... }
```

Или **outbox pattern**: сохранить в БД + запись в outbox, отдельный job сделает external call асинхронно.

---

## 10. Read-only в SUPPORTS + repo

Common pattern:
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

Правильно: read = readOnly + DTO; write = обычная tx + Entity.

---

## 11. Session per test

`@SpringBootTest @Transactional`:

```java
@SpringBootTest
@Transactional      // auto rollback после теста
class OrderServiceTest {

    @Autowired OrderService svc;
    @Autowired OrderRepository repo;

    @Test
    void createOrder_savesEntity() {
        svc.create(new OrderRequest(...));
        assertThat(repo.findAll()).hasSize(1);
        // на конец теста → rollback → БД чистая
    }
}
```

Тест и сервис в **одной** tx → repo видит изменение сервиса.

Кавет: если сервис делает REQUIRES_NEW → это отдельная tx, откатится сама, но тестовая tx её не увидит после rollback.

---

## 12. Кейсы из ИСНА

### 12.1 `knp-fo-sync-notification-bugs`

6 багов sync-сервиса, часть связана с tx + JPA:
- `@Transactional` мёртв (self-invocation) → session не открывается.
- Hib6 LazyInit ловится там где раньше молчал.
- `printStackTrace` вместо log.error → ELK-слепая зона (не видно tx-ошибок).
- Тихий скип на ошибке (MAX offset) → потеря данных.

Урок: правильные tx + structured logging + отсутствие self-invocation.

### 12.2 `taxreport21-java21-runtime-regressions`

- `getById` + `toDto` вне tx → LazyInit.
- Fix: `findById` (SELECT сразу) + правильные @Transactional-методы.

### 12.3 `knp-e2e-runner-hikari-isolation-poisoning`

Не JPA, но связано: `isolation=-1` (opt-in) отравлял пул на pp-pgbouncer → gate краснел ~18 мин. Явно указывать isolation.

---

## 13. Best practices итог

1. **@Transactional на сервис**, не на репозиторий, не на контроллер.
2. **`readOnly = true`** на все чтения.
3. **`rollbackFor = Exception.class`** (или иметь только Runtime exceptions).
4. **Сервис возвращает DTO**, не Entity.
5. **Внешние API — вне tx** (outbox / отдельные методы).
6. **OSIV = false** (`spring.jpa.open-in-view: false`).
7. **`@TransactionalEventListener(AFTER_COMMIT)`** для sending events.
8. **`findById` не `getReferenceById`** (если только не для FK-связи).
9. **Bulk UPDATE / DELETE** → `em.clear()` после.
10. **Timeout** на каждую tx (`@Transactional(timeout = 30)`).
11. **Проверить** что нет self-invocation.
12. **Логировать** `org.springframework.transaction: DEBUG` при отладке.

---

## 14. Собесные вопросы

1. **Как @Transactional работает с EntityManager?** — Открывает EM, привязывает к потоку через TransactionSynchronizationManager; все `@PersistenceContext` получают тот же.
2. **Что такое OSIV?** — Open-Session-In-View: session открыта весь HTTP-запрос; default в Spring Boot; лучше выключить.
3. **Почему OSIV плохо?** — Скрывает N+1, держит DB connection долго, скрывает архитектурные проблемы.
4. **Когда происходит flush?** — Перед commit, перед query (auto), явно `em.flush()`.
5. **readOnly = true — что делает?** — JDBC read-only + Hibernate MANUAL flush + БД оптимизации.
6. **Что произойдёт если изменить поле в readOnly tx?** — Молча ничего (нет flush).
7. **Почему LazyInit важно связано с @Transactional?** — Session открывается только в tx; после выхода — закрыта; lazy fields пытаются load → exception.
8. **REQUIRES_NEW в JPA — риски?** — Требует 2 connection одновременно → пул истощается + возможные deadlock.
9. **Как сохранить audit-запись независимо от главной tx?** — REQUIRES_NEW в отдельном бине.
10. **Почему `@TransactionalEventListener(AFTER_COMMIT)`?** — Публиковать в Kafka/Rabbit только после успешного commit; иначе можно опубликовать, а tx откатится.
11. **Почему нельзя внешний API внутри @Transactional?** — Держит connection и row locks долго; истощение пула, deadlock, длинная tx.
12. **Как правильно возвращать данные из @Transactional?** — DTO. Entity не должна утекать за границы tx.
13. **Разница `findById` и `getReferenceById` в контексте tx?** — findById SELECT сразу; getReferenceById прокси без SELECT — LazyInit если использовать вне tx.
14. **Bulk UPDATE и managed объекты?** — Не проходит через PersistenceContext; managed остаются со старым значением; `em.clear()` после.
15. **Тесты с @Transactional?** — Auto rollback после теста; тест и сервис в одной tx (кроме REQUIRES_NEW внутри).

---

## Итог

- **@Transactional** = **session** = **PersistenceContext** = **connection**, всё привязано к потоку.
- **OSIV = false** и работать с DTO.
- **`readOnly = true`** для чтения.
- **Внешние вызовы вне tx**.
- **AFTER_COMMIT** для events.
- **Self-invocation** = самая частая причина «tx не работает».
- **Hibernate 6** строже к LazyInit — правильные tx обязательны.

---

## Итог блока @Transactional

- 32 — Транзакции: ACID, isolation, propagation, XA, Saga.
- 33 — @Transactional изнутри: прокси, PlatformTransactionManager, self-invocation.
- 34 — Продвинутое: rollback, readOnly, listeners, savepoints, TransactionTemplate, testing.
- 35 — @Transactional + JPA: PersistenceContext, flush, LazyInit, OSIV, best practices.

Следующий — `36-spring-cloud.md`.
