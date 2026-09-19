# 28. PostgreSQL: устройство, MVCC, WAL, индексы

Как PG работает внутри. Что видит Java-приложение через JDBC.

---

## 1. Процесс-модель

PostgreSQL — не многопоточный (как MySQL), а **многопроцессный**. Каждое соединение = отдельный OS-процесс.

```
                    ┌─────────────────────┐
                    │    postmaster       │  ← главный процесс (порт 5432)
                    │  слушает accept     │
                    └──────┬──────────────┘
                           │  fork при новом connection
              ┌────────────┼────────────┬────────────┐
              ▼            ▼            ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
        │backend 1│  │backend 2│  │backend 3│  │backend N│  ← клиентские процессы
        │ (client)│  │ (client)│  │ (client)│  │ (client)│
        └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
             │            │            │            │
             └────────────┴────────────┴────────────┘
                          │  через shared memory
                          ▼
              ┌───────────────────────────┐
              │      Shared Memory         │
              │  ┌──────────────────────┐  │
              │  │   shared_buffers     │  │  ← кэш страниц данных
              │  ├──────────────────────┤  │
              │  │   WAL buffers        │  │
              │  ├──────────────────────┤  │
              │  │   locks              │  │
              │  └──────────────────────┘  │
              └────────────┬───────────────┘
                           │
              ┌────────────┴──────────────┐
              ▼                           ▼
      ┌──────────────┐          ┌────────────────┐
      │  Background  │          │  Data files    │
      │  processes   │          │  (на диске)    │
      │              │          │                │
      │ bgwriter     │          │ base/          │
      │ checkpointer │          │ pg_wal/        │
      │ walwriter    │          │ pg_stat/       │
      │ autovacuum   │          └────────────────┘
      │ archiver     │
      └──────────────┘
```

### 1.1 Backend процесс

- Создаётся на каждый connection.
- Занимает 5-10 MB памяти.
- Держит соединение TCP.
- Обрабатывает запросы клиента.

Отсюда: **много соединений = много процессов = много памяти**. 1000 connections = 5-10 GB на процессы.

Именно поэтому **HikariCP + PgBouncer** — обычная схема. Приложение → пул JDBC (внутри JVM) → PgBouncer (мультиплексирует) → PostgreSQL (мало настоящих backend'ов).

### 1.2 Shared buffers

Кэш страниц данных в RAM. Настройка:
```
shared_buffers = 25% RAM      # default мало (128MB), надо увеличить
```

При запросе:
1. PG ищет страницу в shared_buffers.
2. Нет → читает с диска, кладёт в shared_buffers.
3. Отдаёт клиенту.

### 1.3 Фоновые процессы

- **bgwriter** — сбрасывает грязные страницы на диск.
- **checkpointer** — периодические checkpoint'ы (см. §3).
- **walwriter** — пишет WAL.
- **autovacuum** — чистит мусор (см. §2.5).
- **archiver** — архивирует WAL (для PITR).
- **logical replication** — если настроена.

---

## 2. MVCC — Multi-Version Concurrency Control

Ключевая архитектурная штука PostgreSQL.

### 2.1 Проблема

Классические БД используют locks: если А читает строку, Б не может писать (или наоборот). Медленно, deadlock'и.

### 2.2 Идея MVCC

**Каждая версия строки — отдельная копия**. Транзакция видит ту версию, которая была валидна на момент старта транзакции.

- Читатели не блокируют писателей.
- Писатели не блокируют читателей.
- Много писателей могут блокировать друг друга (изменяют одну строку).

### 2.3 Как реализовано в PG

Каждая строка (tuple) имеет скрытые колонки:
- **xmin** — id транзакции, создавшей эту версию.
- **xmax** — id транзакции, «удалившей» эту версию (или следующая версия).
- **ctid** — физический адрес в таблице.

При UPDATE:
1. Старая версия помечается `xmax = current_tx`.
2. Создаётся новая версия с `xmin = current_tx`.
3. Оба tuple в таблице!

Транзакция T2 читает: если её ID выше xmin И не выше xmax (или xmax=0) → видит эту версию.

### 2.4 Snapshot

Каждая транзакция получает **snapshot** — «список видимых транзакций» на момент начала. Определяет что она видит.

Isolation levels:
- **READ_COMMITTED** (default) — snapshot обновляется на каждый statement.
- **REPEATABLE_READ** — snapshot один на всю tx.
- **SERIALIZABLE** — то же + предикатные блокировки для serialization.

### 2.5 Bloat и VACUUM

**Bloat** — мёртвые версии tuple. Когда UPDATE / DELETE — старая версия не удаляется сразу, только помечается.

**VACUUM** — фоновый процесс, чистит мёртвые версии.

```sql
VACUUM my_table;              -- обычный
VACUUM FULL my_table;          -- полный (перестраивает таблицу, требует lock!)
VACUUM ANALYZE my_table;       -- + обновить статистику
```

**Autovacuum** — работает автоматически, запускается когда мёртвых версий много.

**Проблемы**:
- Долгие транзакции блокируют VACUUM (мёртвая версия «нужна» долгоживущей tx).
- Bloat → таблица растёт, индексы разъезжаются, seq scan медленный.

Мониторить: `pg_stat_all_tables` — `n_dead_tup`, `last_vacuum`.

---

## 3. WAL — Write-Ahead Log

Прежде чем изменение попадёт в таблицу, оно записывается в WAL.

### 3.1 Зачем

- **Durability** (D в ACID) — при crash можно восстановить незакоммиченные изменения.
- **Быстрая запись** — WAL это append-only file, быстрее чем random-write в таблицу.
- **Replication** — реплики читают WAL и применяют изменения.
- **PITR** (Point-in-Time Recovery) — восстановление на любую точку в прошлом.

### 3.2 Как работает

1. Transaction: UPDATE foo SET x=1.
2. PG пишет в WAL buffer запись «UPDATE tuple с ctid=(1,5): xmax=T5».
3. Изменение применяется в shared_buffers.
4. При commit — WAL buffer flush на диск (`fsync`).
5. **После fsync — commit считается успешным**. Данные могут ещё быть только в shared_buffers, не в основных файлах.
6. Периодически **checkpoint** — сброс всех грязных страниц на диск.

### 3.3 Checkpoint

Настройка:
```
checkpoint_timeout = 5min
max_wal_size = 1GB
```

При checkpoint:
- Все грязные страницы shared_buffers → на диск.
- WAL до этой точки можно удалить (уже применено в файлы).

Если приложение много пишет → частые checkpoint → замедление.

### 3.4 fsync и durability

`fsync = on` (default) — синхронный флаш на диск. Медленно, но надёжно.

`fsync = off` — быстро, но при crash — потеря данных. Только для тестов.

---

## 4. Индексы

### 4.1 B-Tree (default)

Балансированное дерево. Хорошо для:
- Equality: `WHERE x = 5`.
- Range: `WHERE x > 5 AND x < 100`.
- Sorting: `ORDER BY x`.
- Prefix: `WHERE x LIKE 'abc%'`.

Не подходит:
- `LIKE '%abc%'` (без leading `%`).
- Функции на колонке: `WHERE lower(name) = 'x'` (нужен functional index).

### 4.2 Hash

Только equality. Быстрее B-Tree для точного совпадения, но:
- Не range.
- Не sort.
- Раньше не был crash-safe (до PG 10).

Использовать редко.

### 4.3 GiST (Generalized Search Tree)

Для сложных типов:
- Геометрия (PostGIS).
- Full-text search.
- Range types (`tsrange`).

### 4.4 GIN (Generalized Inverted Index)

Инвертированный индекс. Для:
- Full-text search (`tsvector`).
- Массивы (`array_column @> ARRAY[1,2]`).
- JSONB.

Медленный на insert, быстрый на search.

### 4.5 BRIN (Block Range Index)

Индекс по диапазонам блоков. Только для больших таблиц с натурально упорядоченными данными (например, `created_at`).

Крошечный (мегабайты для терабайтной таблицы), но менее точный.

### 4.6 Composite index

Многоколоночный:
```sql
CREATE INDEX ON fno(status, created_at);
```

**Порядок важен!** Индекс работает для:
- `WHERE status = 'NEW'` (leftmost).
- `WHERE status = 'NEW' AND created_at > ...`.
- НЕ работает для `WHERE created_at > ...` (без status).

### 4.7 Partial index

Индекс только по части строк:
```sql
CREATE INDEX ON fno(created_at) WHERE status = 'NEW';
```

Меньше индекс + быстрее поиск по «горячим» данным.

### 4.8 Unique index

```sql
CREATE UNIQUE INDEX ON fno(reg_num);
```

Гарантирует уникальность + быстрый поиск.

### 4.9 Функциональный индекс

```sql
CREATE INDEX ON users(lower(email));
```

Работает для `WHERE lower(email) = 'x@y.com'`.

---

## 5. Планировщик и EXPLAIN

### 5.1 Как PG выполняет запрос

1. **Parse** — SQL → AST.
2. **Rewrite** — применение правил (views).
3. **Plan** — планировщик выбирает лучший план (nested loop / hash / merge, index / seq scan).
4. **Execute** — выполнение плана.

Планировщик использует **статистику** (`pg_statistic`) — гистограммы значений колонок. Обновляется через `ANALYZE`.

Устаревшая статистика → плохой план → тормоза.

### 5.2 EXPLAIN

```sql
EXPLAIN SELECT * FROM fno WHERE status = 'NEW';
```

Показывает план:
```
Seq Scan on fno  (cost=0.00..1000.00 rows=100 width=200)
  Filter: (status = 'NEW')
```

`cost` — оценка планировщика (не время в мс!).
`rows` — оценка количества строк.

### 5.3 EXPLAIN ANALYZE

Реальное выполнение с замерами:
```sql
EXPLAIN ANALYZE SELECT * FROM fno WHERE status = 'NEW';
```

```
Index Scan using idx_fno_status on fno  (cost=... rows=100)
                                        (actual time=0.5..2.3 rows=95 loops=1)
Planning Time: 0.2 ms
Execution Time: 2.5 ms
```

Ищи:
- `Seq Scan` на большой таблице → нужен индекс.
- `rows` estimate != actual → устаревшая статистика (`ANALYZE`).
- Много loops в nested loop → плохой join.

### 5.4 Основные операции

- **Seq Scan** — читать всю таблицу.
- **Index Scan** — по индексу.
- **Index Only Scan** — только индекс, не в таблицу (если все нужные колонки в индексе).
- **Bitmap Index Scan** + **Bitmap Heap Scan** — сначала собрать bitmap строк по индексу, потом читать.
- **Hash Join** — построить хеш-таблицу меньшей стороны, пройти большую.
- **Merge Join** — обе стороны отсортированы, сливаем.
- **Nested Loop** — вложенный цикл (внешняя таблица × поиск для каждой строки).

---

## 6. JOIN алгоритмы

### 6.1 Nested Loop

```
for row1 in table1:
    for row2 in table2:
        if row1.key == row2.key:
            output(row1, row2)
```

Плохо для больших таблиц. Хорошо когда внутренняя маленькая или есть индекс.

### 6.2 Hash Join

```
hash_table = build hash on table1.key
for row2 in table2:
    if row2.key in hash_table:
        output(...)
```

Хорошо для equi-join большие таблицы. Требует память для hash.

### 6.3 Merge Join

Обе таблицы отсортированы по key → merge:
```
i=0, j=0
while i < len(t1) and j < len(t2):
    if t1[i].key == t2[j].key: output
    elif t1[i].key < t2[j].key: i++
    else: j++
```

Хорошо когда данные уже отсортированы (индекс).

Планировщик сам выбирает.

---

## 7. Транзакции и блокировки

### 7.1 Basic

```sql
BEGIN;
UPDATE fno SET status='X' WHERE id=1;
COMMIT;
-- или ROLLBACK
```

### 7.2 Row-level locks

```sql
SELECT * FROM fno WHERE id=1 FOR UPDATE;
```

Блокирует строку. Другая транзакция ждёт (или падает с `NOWAIT` / `SKIP LOCKED`).

### 7.3 Advisory locks

Приложение-уровень:
```sql
SELECT pg_advisory_lock(12345);
-- работа
SELECT pg_advisory_unlock(12345);
```

Использование: distributed lock для scheduled jobs (аналог ShedLock).

### 7.4 Deadlock

Две транзакции ждут друг друга:
```
T1: UPDATE a WHERE id=1;
T2: UPDATE a WHERE id=2;
T1: UPDATE a WHERE id=2;  ← ждёт T2
T2: UPDATE a WHERE id=1;  ← ждёт T1
```

PG обнаруживает через таймаут → одна tx получает `deadlock detected`.

Профилактика: всегда брать locks в одном порядке (по id).

---

## 8. Statistic tables (для мониторинга)

```sql
-- активные connections
SELECT * FROM pg_stat_activity;

-- статистика таблиц
SELECT * FROM pg_stat_user_tables;
  -- n_dead_tup, n_live_tup, last_vacuum, last_analyze

-- медленные запросы (нужен pg_stat_statements)
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 20;

-- блокировки
SELECT * FROM pg_locks WHERE NOT granted;

-- размер таблиц
SELECT pg_size_pretty(pg_total_relation_size('fno'));

-- размер индексов
SELECT pg_size_pretty(pg_relation_size('idx_fno_status'));

-- список индексов и их использование
SELECT * FROM pg_stat_user_indexes;
```

`pg_stat_statements` — расширение, надо включить:
```
shared_preload_libraries = 'pg_stat_statements'
```

Обязательно в проде.

---

## 9. Репликация

### 9.1 Streaming replication (physical)

WAL пишется на master, стримится на реплики. Реплики применяют.

```
[master] --WAL--> [replica 1]
              \--> [replica 2]
```

- **Synchronous** — commit только когда реплика подтвердила (медленнее, надёжнее).
- **Asynchronous** — быстрее, но при падении master'а может потерять последние транзакции.

### 9.2 Logical replication

Репликация по CDC (change data capture). Гибче — можно реплицировать конкретные таблицы, между разными версиями PG.

### 9.3 Read replicas

Реплики можно использовать для чтения (SELECT). Приложение делает write в master, read в replica.

```yaml
spring:
  datasource:
    write:
      url: jdbc:postgresql://master:5432/knp
    read:
      url: jdbc:postgresql://replica:5432/knp
```

Кавет: **lag** — replica может отстать → устаревшие данные. Не для критичного read-after-write.

---

## 10. Партиционирование

Разбивка большой таблицы на части (partitions) по ключу.

```sql
CREATE TABLE fno (
    id BIGSERIAL,
    created_at TIMESTAMP NOT NULL,
    ...
) PARTITION BY RANGE (created_at);

CREATE TABLE fno_2025 PARTITION OF fno
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
CREATE TABLE fno_2026 PARTITION OF fno
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

Плюсы:
- Быстрый DELETE старых данных (DROP partition).
- Партиция pruning в запросах.
- Меньшие индексы.

Минусы:
- Сложнее миграции.
- UNIQUE constraint по всем партициям — сложно.

---

## 11. Типы данных

- **`bigserial`** / **`bigint`** — 64-бит id.
- **`text`** — строка без лимита.
- **`varchar(N)`** — по сути `text` с check.
- **`timestamp`** vs **`timestamptz`** — с/без таймзоны. **Всегда `timestamptz`** в проде.
- **`jsonb`** — бинарный JSON, индексируется через GIN. Лучше `json`.
- **`uuid`** — 128-бит.
- **`numeric(P, S)`** — точный десятичный (для денег!).
- **`bytea`** — бинарные данные.

---

## 12. Настройки под нагрузку

```
shared_buffers = 4GB              # 25% RAM
effective_cache_size = 12GB       # 75% RAM (подсказка планировщику)
work_mem = 32MB                    # для сортировок в query
maintenance_work_mem = 1GB         # для VACUUM, CREATE INDEX
max_connections = 200              # ограничение
wal_level = replica
max_wal_size = 4GB
checkpoint_timeout = 15min
random_page_cost = 1.1             # для SSD
```

Тюнинг → `pgtune`, `pgtune.leopard.in.ua`.

---

## 13. Собесные вопросы

1. **Процесс-модель PG?** — postmaster + отдельный backend на каждый connection.
2. **Что такое MVCC?** — Много версий tuple; читатели не блокируют писателей.
3. **Что такое xmin/xmax?** — Скрытые колонки tuple: transaction ID создания / удаления.
4. **Что такое VACUUM?** — Фоновая чистка мёртвых версий tuple.
5. **Что такое WAL?** — Write-Ahead Log — сначала пишем изменение в WAL, потом в таблицу.
6. **Разница shared_buffers и OS-кэш?** — shared_buffers в РАМ PG; OS-кэш — файловый.
7. **Типы индексов?** — B-Tree (default), Hash, GiST, GIN (для jsonb/массивов), BRIN.
8. **Как работает composite index?** — Дерево, порядок колонок важен, leftmost prefix.
9. **Что такое EXPLAIN ANALYZE?** — Реальное выполнение с замерами.
10. **JOIN алгоритмы?** — Nested Loop, Hash Join, Merge Join.
11. **Что такое deadlock?** — Циклическое ожидание блокировок; PG detected → одна tx падает.
12. **Разница timestamp и timestamptz?** — timestamptz с таймзоной; всегда используй его.
13. **jsonb vs json?** — jsonb бинарный, индексируется, поддерживает операторы; json — просто текст.
14. **Read replicas — когда?** — Для масштабирования read-нагрузки; кавет — replication lag.
15. **Что такое партиционирование?** — Разбивка большой таблицы по ключу; ускоряет запросы + DROP старых.

---

## Итог

- **Процесс-модель**: 1 backend = 1 connection.
- **MVCC** через xmin/xmax + snapshot.
- **VACUUM** обязателен, иначе bloat.
- **WAL** = durability + replication.
- **shared_buffers** = кэш страниц (25% RAM).
- **Индексы**: B-Tree default; GIN для jsonb; composite → порядок важен.
- **EXPLAIN ANALYZE** — must для оптимизации.
- **timestamptz + jsonb + numeric(P,S)** — правильные типы.
- **pg_stat_statements** — must в проде.

Следующий — `29-postgresql-spring-hikaricp.md`.
