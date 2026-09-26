# 68. JPA Stream, Pageable, Cursor, JDBC Connection

Как читать большие наборы данных из БД. Разница подходов, когда что.

---

## 1. Проблема

Классический код:
```java
List<Order> all = repo.findAll();
for (Order o : all) {
    process(o);
}
```

Если в БД **10 миллионов** строк:
- **Все** в память → **OutOfMemoryError**.
- Одна строка ~1 KB → 10 GB heap.
- Приложение падает.

Даже если поместится — Hibernate держит все managed → PersistenceContext раздут → медленно.

Нужны **стратегии для больших данных**. Разберём 4 подхода.

---

## 2. Что такое JDBC Connection

Базовый уровень. Всё JPA работает через JDBC.

### 2.1 Определение

**JDBC Connection** — открытое **TCP-соединение** к БД + активная сессия.

```java
Connection conn = ds.getConnection();
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM orders");

while (rs.next()) {
    long id = rs.getLong("id");
    // ...
}

rs.close();
stmt.close();
conn.close();
```

### 2.2 Компоненты

- **Connection** — TCP + auth + session state.
- **Statement / PreparedStatement** — SQL для выполнения.
- **ResultSet** — итератор по результату.

### 2.3 Стоимость

Открытие Connection:
1. TCP handshake (~1-10 ms).
2. SSL handshake (~10-50 ms если SSL).
3. Auth (~10-30 ms).

**Итого 30-100 ms** на открытие → **connection pool** обязателен (HikariCP, см. `29-postgresql-spring-hikaricp.md`).

### 2.4 Что важно для больших данных

**JDBC не грузит весь результат в память сразу**. `ResultSet` — **курсор** на сервере БД.

По default:
- **`fetchSize=0`** — драйвер решает (для PG обычно = ВСЁ в память клиента!).
- **`fetchSize=N`** — драйвер тянет пачками по N строк.

```java
stmt.setFetchSize(100);   // тянуть по 100 строк
ResultSet rs = stmt.executeQuery("SELECT * FROM orders");
while (rs.next()) { ... }   // прозрачно, драйвер сам подгружает
```

Для PostgreSQL кавет: **fetchSize работает только при `autoCommit=false`**. Иначе PG отдаёт всё сразу.

### 2.5 ResultSet types

- **TYPE_FORWARD_ONLY** — только `next()` (default, быстро).
- **TYPE_SCROLL_INSENSITIVE** — можно `previous()`, `first()`, `last()`. Требует memory.
- **TYPE_SCROLL_SENSITIVE** — то же + видит изменения.

Для стриминга — `FORWARD_ONLY`.

---

## 3. Что такое Pageable

**Pageable** — Spring абстракция для **пагинации** (offset/limit).

### 3.1 Использование

```java
interface OrderRepository extends JpaRepository<Order, Long> {
}

Page<Order> page = repo.findAll(PageRequest.of(0, 20, Sort.by("createdAt").descending()));
List<Order> content = page.getContent();
long total = page.getTotalElements();
int pages = page.getTotalPages();
boolean hasNext = page.hasNext();
```

Под капотом:
```sql
-- запрос данных
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 0;

-- запрос total (extra!)
SELECT count(*) FROM orders;
```

**Два запроса** на каждый вызов `findAll(Pageable)`.

### 3.2 В контроллере

```java
@GetMapping("/orders")
public Page<Order> list(Pageable pageable) {
    return repo.findAll(pageable);
}
```

URL: `/orders?page=0&size=20&sort=createdAt,desc` — Spring парсит автоматом.

### 3.3 Slice — без COUNT

```java
Slice<Order> slice = repo.findAllByStatus(NEW, PageRequest.of(0, 20));
slice.getContent();
slice.hasNext();   // но нет totalPages
```

Один SQL вместо двух — быстрее.

Использовать когда total не нужен (infinite scroll).

### 3.4 Как работает под капотом (LIMIT OFFSET)

```sql
LIMIT 20 OFFSET 0     -- первая страница
LIMIT 20 OFFSET 20    -- вторая
LIMIT 20 OFFSET 40    -- третья
...
LIMIT 20 OFFSET 100000   -- 5000-я
```

**Проблема на больших OFFSET**: PG вынужден **прочитать 100020 строк** → отбросить 100000 → вернуть 20.

Latency растёт **линейно** с OFFSET.

### 3.5 Keyset pagination — правильно для больших

Вместо OFFSET — **курсор по значению**:
```sql
-- вместо OFFSET
WHERE created_at < :last_seen_at OR (created_at = :last_seen_at AND id < :last_seen_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Клиент шлёт `last_seen_at + id` следующей страницы. PG использует индекс → **O(log N)** независимо от глубины.

Минус: нельзя прыгнуть на страницу 500 сразу.

Spring Data JPA не имеет встроенного keyset — приходится ручной SQL или **Spring Data JDBC** / **Blaze-Persistence**.

### 3.6 Кавет N+1 с Pageable

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAll(Pageable p);
```

Hibernate предупредит:
```
HHH000104: firstResult/maxResults specified with collection fetch;
applying in memory!
```

**Не масштабируется**. Не делай JOIN FETCH коллекций с Pageable.

Fix:
1. Загрузить IDs через Pageable.
2. Второй запрос с JOIN FETCH по этим IDs.

---

## 4. Что такое JPA Stream

**Stream** — способ **лениво итерировать** большой набор без загрузки всего в память.

### 4.1 Использование

```java
interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o WHERE o.status = :status")
    Stream<Order> streamByStatus(@Param("status") OrderStatus status);
}
```

Использование:
```java
@Transactional(readOnly = true)
public void processAll() {
    try (Stream<Order> stream = repo.streamByStatus(NEW)) {
        stream.forEach(order -> process(order));
    }
    // try-with-resources — обязательно! закроет ResultSet
}
```

### 4.2 Как работает

Hibernate:
1. Открывает **JDBC ResultSet** cursor на сервере БД.
2. Возвращает `Stream` с ленивым чтением.
3. `.forEach()` тянет по одной строке.
4. Строка → managed Entity.
5. **PersistenceContext накапливается** (см. §4.4).
6. `stream.close()` (try-with-resources) → закрывает ResultSet + Statement + Connection.

### 4.3 Требования

- **`@Transactional(readOnly = true)`** — иначе LazyInit при итерации.
- **`try-with-resources`** — иначе утечка ResultSet.
- **`fetchSize`** настроен на driver (для PG — не забыть).
- Обычно **`hibernate.query.result_stream = true`** для Hibernate 6+.

### 4.4 Кавет OutOfMemory даже со Stream

Stream итерирует по одной строке, но **Hibernate держит все в PersistenceContext**. 10M строк → 10M managed entities → OOM.

**Fix**: периодически чистить:
```java
@PersistenceContext EntityManager em;

@Transactional(readOnly = true)
public void processAll() {
    try (Stream<Order> stream = repo.streamByStatus(NEW)) {
        AtomicInteger counter = new AtomicInteger();
        stream.forEach(order -> {
            process(order);
            if (counter.incrementAndGet() % 100 == 0) {
                em.clear();   // ← освободить PersistenceContext
            }
        });
    }
}
```

### 4.5 Настройки для PG streaming

```yaml
spring:
  jpa:
    properties:
      hibernate:
        query:
          hint:
            fetch_size: 100
  datasource:
    hikari:
      auto-commit: false          # ОБЯЗАТЕЛЬНО для PG streaming
```

Плюс:
- `@Query(hints = @QueryHint(name = "org.hibernate.fetchSize", value = "100"))`.

---

## 5. Что такое курсор JPA

**Cursor** — указатель на текущую позицию в результатах на сервере БД.

### 5.1 На уровне БД

PostgreSQL `DECLARE CURSOR`:
```sql
BEGIN;
DECLARE my_cursor CURSOR FOR SELECT * FROM orders WHERE status = 'NEW';
FETCH 100 FROM my_cursor;    -- получить 100 строк
FETCH 100 FROM my_cursor;    -- ещё 100
CLOSE my_cursor;
COMMIT;
```

Сервер держит "снимок" данных на моменте `DECLARE`.

### 5.2 В JDBC

`ResultSet` — фактически cursor:
- `next()` перемещает cursor.
- Данные подгружаются по `fetchSize`.

### 5.3 В JPA

Stream (§4) внутри использует JDBC ResultSet cursor.

Более "низкоуровневый" cursor — через `ScrollableResults` (Hibernate-specific):

```java
Session session = em.unwrap(Session.class);

try (ScrollableResults<Order> results = session
        .createQuery("FROM Order WHERE status = :s", Order.class)
        .setParameter("s", NEW)
        .setFetchSize(100)
        .scroll(ScrollMode.FORWARD_ONLY)) {

    while (results.next()) {
        Order o = results.get();
        process(o);
        if (results.getRowNumber() % 100 == 0) {
            em.flush();
            em.clear();
        }
    }
}
```

Прямой контроль. Ниже уровнем чем Stream.

### 5.4 JdbcTemplate + RowCallbackHandler

Ещё один вариант — обойти JPA целиком:
```java
@Autowired JdbcTemplate jdbc;

jdbc.query(
    "SELECT * FROM orders WHERE status = ?",
    ps -> {
        ps.setFetchSize(100);
        ps.setString(1, "NEW");
    },
    (rs) -> {
        Order o = mapRow(rs);
        process(o);
    });
```

`RowCallbackHandler` — вызывается для каждой строки. Никакой PersistenceContext, никакого Hibernate → **низкая память + быстро**.

Минус: mapping ручной.

---

## 6. Разница подходов

### 6.1 Сравнительная таблица

| | Pageable | Stream | Cursor (ScrollableResults) | JdbcTemplate + RowCallback |
|---|---|---|---|---|
| Загружает всё? | По страницам | Ленивая итерация | Ленивая итерация | Ленивая итерация |
| Использование памяти (при 1M rows) | ~20 объектов + context | 1M managed! | 1M managed! | 0 (просто callback) |
| SQL кол-во | 2 на страницу (data + count) | 1 | 1 | 1 |
| Total count | Есть | Нет | Нет | Нет |
| Может прыгнуть на страницу N | Да (медленно на больших) | Нет | Нет | Нет |
| Требует tx | Не для Slice | Да | Да | Да |
| Простота | Простая | Средняя (нужно закрывать) | Сложная | Средняя |
| Производительность на 1M | Медленно (OFFSET) | Средне (context bloat) | Средне | Быстро |
| Best for | UI pagination | Batch processing | Batch processing | Reporting / export |

### 6.2 Когда что

**Pageable (Slice)**:
- ✅ UI пагинация (< 1000 страниц).
- ✅ REST API endpoints для клиентов.
- ❌ Batch processing большими наборами.
- ❌ Report / export.

**Stream**:
- ✅ Batch processing (обработать все NEW orders).
- ✅ ETL.
- ✅ Migration.
- ❌ UI (клиент не может ждать всё).

**ScrollableResults**:
- Аналог Stream, более контроль.
- Редко используется — Stream проще.

**JdbcTemplate + RowCallbackHandler**:
- ✅ Reports / exports (миллионы строк).
- ✅ ETL heavy processing.
- ✅ Когда JPA overhead не нужен.
- ❌ Когда нужны managed entities.

**Keyset pagination**:
- ✅ Infinite scroll (много "next").
- ✅ Deep pages в UI.
- ✅ Замена OFFSET на больших наборах.
- ❌ Прыжок на конкретную страницу.

---

## 7. Различие Pageable и Stream

Ключевое.

### 7.1 Pageable — offset-based

```
Page 1: SELECT ... LIMIT 20 OFFSET 0
Page 2: SELECT ... LIMIT 20 OFFSET 20
Page 3: SELECT ... LIMIT 20 OFFSET 40
...
```

- **Клиент решает** какую страницу.
- **Отдельные запросы** на каждую страницу.
- **Позиция сохраняется на клиенте** (page number).
- Хорошо для UI: «показать страницу 5».

### 7.2 Stream — single query, lazy fetch

```
SELECT ... FROM orders WHERE status = 'NEW'
[cursor] → fetch 100 → fetch 100 → ... → close
```

- **Один SQL** запрос.
- **Cursor держится открытым** на весь процесс.
- **Позиция на сервере БД**.
- Один сеанс обрабатывает **всё**.

### 7.3 Пример

Обработать 1M orders:

**С Pageable** (плохо):
```java
int page = 0;
while (true) {
    Page<Order> p = repo.findAll(PageRequest.of(page, 100));
    if (p.isEmpty()) break;
    for (Order o : p.getContent()) process(o);
    page++;
}
// 10 000 SQL запросов + count'ы!
// OFFSET растёт → каждая страница медленнее.
```

**Со Stream** (лучше):
```java
try (Stream<Order> s = repo.streamByStatus(NEW)) {
    s.forEach(o -> {
        process(o);
        counter++;
        if (counter % 100 == 0) em.clear();
    });
}
// 1 SQL, 1M строк лениво.
```

**С JdbcTemplate** (самое быстрое):
```java
jdbc.query("SELECT * FROM orders WHERE status = 'NEW'",
    ps -> ps.setFetchSize(100),
    rs -> process(mapRow(rs)));
// 1 SQL, минимум памяти.
```

---

## 8. Pageable в БД (SQL уровень)

На SQL — это **`LIMIT` + `OFFSET`**.

### 8.1 LIMIT / OFFSET

```sql
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

PG (и большинство БД):
1. Сканирует таблицу или индекс.
2. Сортирует.
3. Пропускает первые 40 строк (`OFFSET`).
4. Возвращает следующие 20 (`LIMIT`).

**Проблема**: чтобы пропустить 40 — надо **прочитать 40**.

- `OFFSET 100` → прочитано 120 строк, вернулось 20.
- `OFFSET 10000` → прочитано 10020 строк.
- `OFFSET 1000000` → прочитано 1M строк!

Отсюда — **медленно на больших OFFSET**.

### 8.2 Индексы помогают

Если ORDER BY по индексируемой колонке — PG может использовать index scan → не читает всю таблицу, но всё равно должен пропустить N.

### 8.3 Keyset pagination

Заменяет OFFSET на **условие**:
```sql
-- вместо OFFSET 100000
SELECT * FROM orders
WHERE (created_at, id) < ('2026-09-01 12:00:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Использует индекс на `(created_at, id)` → **O(log N)** независимо от глубины.

Клиент запоминает last `created_at + id` → шлёт в следующем запросе как cursor.

### 8.4 SQL Server / Oracle синтаксис

- **SQL Server**: `OFFSET 100 ROWS FETCH NEXT 20 ROWS ONLY`.
- **Oracle**: `OFFSET 100 ROWS FETCH FIRST 20 ROWS ONLY` (Oracle 12c+) или `ROWNUM`.

Spring Data JPA абстрагирует — через Hibernate dialect.

---

## 9. Реализация в Spring — примеры

### 9.1 Simple UI pagination

```java
@RestController
class OrderController {
    @GetMapping("/orders")
    public Page<Order> list(Pageable pageable) {
        return repo.findAll(pageable);
    }
}

// URL: /orders?page=0&size=20&sort=createdAt,desc
```

### 9.2 Infinite scroll (Slice)

```java
@GetMapping("/orders")
public Slice<Order> list(Pageable pageable) {
    return repo.findAllByStatus(NEW, pageable);   // без count
}
```

### 9.3 Batch processing (Stream)

```java
@Service
class OrderProcessor {

    @PersistenceContext EntityManager em;
    @Autowired OrderRepository repo;

    @Transactional(readOnly = true)
    public void processAllNew() {
        try (Stream<Order> stream = repo.streamByStatus(OrderStatus.NEW)) {
            AtomicInteger count = new AtomicInteger();
            stream.forEach(order -> {
                try {
                    process(order);
                    if (count.incrementAndGet() % 100 == 0) {
                        em.clear();
                    }
                } catch (Exception e) {
                    log.error("Failed to process {}", order.getId(), e);
                }
            });
        }
    }
}

interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o FROM Order o WHERE o.status = :status")
    @QueryHints(@QueryHint(name = "org.hibernate.fetchSize", value = "100"))
    Stream<Order> streamByStatus(@Param("status") OrderStatus status);
}
```

### 9.4 Export (JdbcTemplate)

Экспорт 1M ordersов в CSV:

```java
@Service
class OrderExporter {

    @Autowired JdbcTemplate jdbc;

    public void exportToCsv(OutputStream out) throws IOException {
        try (Writer w = new OutputStreamWriter(out)) {
            w.write("id,customerId,amount,status,createdAt\n");
            jdbc.query(conn -> {
                conn.setAutoCommit(false);   // для PG cursor
                PreparedStatement ps = conn.prepareStatement(
                    "SELECT * FROM orders ORDER BY created_at",
                    ResultSet.TYPE_FORWARD_ONLY,
                    ResultSet.CONCUR_READ_ONLY);
                ps.setFetchSize(500);
                return ps;
            }, rs -> {
                try {
                    w.write(rs.getLong("id") + ",");
                    w.write(rs.getString("customer_id") + ",");
                    w.write(rs.getBigDecimal("amount").toString() + ",");
                    w.write(rs.getString("status") + ",");
                    w.write(rs.getTimestamp("created_at").toString() + "\n");
                } catch (IOException e) {
                    throw new UncheckedIOException(e);
                }
            });
        }
    }
}
```

Минимум памяти. Быстро.

### 9.5 Keyset pagination

```java
interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("""
        SELECT o FROM Order o
        WHERE (o.createdAt < :lastCreatedAt)
           OR (o.createdAt = :lastCreatedAt AND o.id < :lastId)
        ORDER BY o.createdAt DESC, o.id DESC
        """)
    List<Order> findNext(
        @Param("lastCreatedAt") LocalDateTime lastCreatedAt,
        @Param("lastId") Long lastId,
        Pageable limit);   // только size, без offset
}

// использование
List<Order> firstPage = repo.findNext(now(), Long.MAX_VALUE, PageRequest.of(0, 20));
Order last = firstPage.get(firstPage.size() - 1);
List<Order> secondPage = repo.findNext(last.getCreatedAt(), last.getId(), PageRequest.of(0, 20));
```

Быстро независимо от "глубины".

---

## 10. Типовые проблемы

### 10.1 `findAll()` без Pageable в проде

```java
List<Order> all = repo.findAll();   // 10M строк!
```

OOM. **Никогда** без `Pageable` / Stream / limit.

### 10.2 Stream не в @Transactional

```java
Stream<Order> stream = repo.streamByStatus(NEW);   // ← не в tx
stream.forEach(...);   // → LazyInit / stream closed
```

Fix: `@Transactional`.

### 10.3 Stream не закрыт

```java
Stream<Order> stream = repo.streamByStatus(NEW);
stream.forEach(...);
// ← забыли close → утечка ResultSet + connection
```

Fix: `try-with-resources`.

### 10.4 Stream + большой context

Stream итерирует ленивую, но Hibernate держит managed entities. Периодически `em.clear()`.

### 10.5 fetchSize=0 в PG

```java
stmt.executeQuery(...);   // без setFetchSize → PG отдаёт всё сразу → OOM
```

Fix: `setFetchSize(N)` + `autoCommit=false`.

### 10.6 Pageable + JOIN FETCH коллекций

`HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!`.

Hibernate загружает **всё** и режет в памяти. Не масштабируется.

Fix: 2-фазный запрос.

### 10.7 OFFSET на глубоких страницах медленный

Смотри keyset pagination.

### 10.8 Sliding pagination

При изменении данных страница «плавает»:
```
Time T1: страница 1 = [O1, O2, ..., O20]
Time T2: INSERT O0
Time T2: страница 2 = [O20, O21, ...O39]
                       ↑ O20 повторяется!
```

Keyset решает.

---

## 11. Реальные кейсы ИСНА

Из memory:

- **`knp-filter-sent-documents-perf`** — sync RestTemplate на АРМ `/api/documents/fno` через `DocumentService`. Не Stream, но проблема аналогичная — медленный proxy-запрос блокирует downstream.
- Прогон `knp-e2e` через flow-runner — по сути batch processing большого набора tasks.
- Fno328 регенерация — `Fno328PdfRegenerationJob` в report — типичный batch, где Stream / Cursor уместнее чем Page.

Правило: **batch job = Stream**, **UI = Pageable/Slice**.

---

## 12. Best practices

1. **UI**: `Slice` (без count) или `Page` (с count если нужно).
2. **Deep pages**: keyset pagination.
3. **Batch processing**: `Stream` с `@Transactional(readOnly=true)` + `em.clear()` каждые N.
4. **Export/report**: `JdbcTemplate` + `RowCallbackHandler` + `fetchSize`.
5. **PG streaming**: `autoCommit=false`.
6. **`fetchSize`** явно устанавливать.
7. **`try-with-resources`** для Stream / ScrollableResults.
8. **Никогда `findAll()`** без limit в проде.
9. **`JOIN FETCH коллекций` + Pageable** — не масштабируется.
10. **Мониторинг**: длинные транзакции (streaming может держать долго).

---

## 13. Собесные вопросы

1. **Что такое Pageable?** — Spring абстракция для offset-based пагинации (LIMIT/OFFSET).
2. **Разница Page и Slice?** — Page с total count (extra COUNT SQL); Slice без.
3. **Что такое Stream в JPA?** — Ленивая итерация ResultSet cursor'а; для batch processing.
4. **Разница Pageable и Stream?** — Pageable: страницы через отдельные SQL; Stream: один SQL, cursor.
5. **Что такое JDBC Connection?** — TCP + auth + session к БД; создание дорого — использовать pool.
6. **Что такое ResultSet?** — Cursor по результатам, ленивая подгрузка через fetchSize.
7. **Что такое fetchSize?** — Сколько строк JDBC-драйвер тянет за раз; 0 = всё сразу (плохо).
8. **PostgreSQL и streaming — что важно?** — `autoCommit=false` обязательно, иначе PG всё сразу отдаст.
9. **OFFSET на больших страницах — проблема?** — PG вынужден прочитать N+size строк; медленно.
10. **Keyset pagination — что даёт?** — Условие вместо OFFSET; использует индекс; O(log N).
11. **Курсор БД что такое?** — Указатель на позицию в результате; серверная память.
12. **Как не получить OOM со Stream?** — Периодически `em.clear()`.
13. **Требования для JPA Stream?** — @Transactional, try-with-resources, fetchSize.
14. **Когда JdbcTemplate вместо JPA?** — Reports/exports миллионов строк; минимум overhead.
15. **JOIN FETCH коллекций + Pageable?** — Hibernate warning; загружает всё в память; не масштабируется.

---

## Итог

- **JDBC Connection** = базовый уровень; всё через pool (HikariCP).
- **Pageable** (LIMIT/OFFSET) — для **UI пагинации**; отдельные запросы на страницу.
- **Stream** (cursor) — для **batch processing**; один SQL, ленивая итерация.
- **JdbcTemplate + RowCallback** — для **export/report** миллионов строк; минимум памяти.
- **Keyset pagination** — замена OFFSET на глубоких страницах.
- **PG streaming**: `autoCommit=false` + `fetchSize`.
- **Никогда `findAll()`** на больших таблицах.
- Правило: **UI = Pageable, Batch = Stream, Export = JdbcTemplate**.

Итого файл 68 добавлен. **68 файлов в `isna-theory\`**.
