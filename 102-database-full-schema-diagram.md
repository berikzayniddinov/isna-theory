# 102. Точечная схема БД: полный путь от SQL-запроса до диска (PostgreSQL)

Полная визуальная карта работы БД в проекте КНП. Как приложение общается с PostgreSQL, что происходит внутри сервера, где живут данные физически, как транзакции, MVCC, индексы, WAL, репликация, backup работают на самом деле. Точечно, слой за слоем.

Такая же структура, как в 101-м файле для build/deploy, но для БД.

---

## 0. Общая карта — от JDBC до NVMe SSD

```
┌────────────────────────────────────────────────────────────────────┐
│                    JAVA APPLICATION (Spring Boot)                  │
│                                                                    │
│   @Service                                                         │
│   UserService                                                      │
│      │                                                             │
│      ▼                                                             │
│   UserRepository extends JpaRepository                             │
│      │                                                             │
│      ▼                                                             │
│   Hibernate ORM (JPA implementation)                               │
│      │                                                             │
│      ▼                                                             │
│   JDBC API (java.sql.Connection)                                   │
│      │                                                             │
│      ▼                                                             │
│   HikariCP connection pool                                         │
│      │                                                             │
│      ▼                                                             │
│   PostgreSQL JDBC driver (pgjdbc)                                  │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ TCP/IP (порт 5432)
                       │ PostgreSQL wire protocol v3
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│                     PGBOUNCER (опционально)                        │
│   - Connection pooler                                              │
│   - Мультиплексирует thousands клиентов в десятки соединений       │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│                    POSTGRESQL SERVER                               │
│                                                                    │
│   Postmaster (main process)                                        │
│      │ fork() на каждое соединение                                 │
│      ▼                                                             │
│   Backend process (один на клиента)                                │
│      │                                                             │
│      ├──► Parser (SQL → AST)                                       │
│      ├──► Rewriter (views, rules)                                  │
│      ├──► Planner/Optimizer (best execution plan)                  │
│      └──► Executor (выполнение)                                    │
│               │                                                    │
│               ├──► Buffer Pool (shared_buffers, ~25% RAM)          │
│               │       │                                            │
│               │       └──► Page cache (Linux kernel)               │
│               │             │                                      │
│               │             └──► Disk (data files, WAL)            │
│               │                                                    │
│               ├──► WAL Writer                                      │
│               ├──► MVCC (xmin/xmax visibility)                     │
│               └──► Locks                                           │
│                                                                    │
│   Background processes:                                            │
│      - Autovacuum workers                                          │
│      - Checkpointer                                                │
│      - Background Writer                                           │
│      - WAL Writer                                                  │
│      - WAL Sender (для replication)                                │
│      - Stats Collector                                             │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│                    FILESYSTEM / DISK                               │
│                                                                    │
│   /var/lib/postgresql/data/                                        │
│   ├── base/               ← таблицы, индексы (heap files)          │
│   ├── pg_wal/             ← Write-Ahead Log                        │
│   ├── pg_xact/            ← transaction commit log                 │
│   ├── pg_multixact/       ← multi-transaction state                │
│   ├── pg_tblspc/          ← дополнительные tablespaces             │
│   ├── postgresql.conf                                              │
│   └── pg_hba.conf         ← access control                         │
│                                                                    │
│   Физически: NVMe SSD (data), возможно separate SSD для WAL        │
└────────────────────────────────────────────────────────────────────┘
```

---

## 1. КАК ПРИЛОЖЕНИЕ ОБЩАЕТСЯ С БД — от @Service до TCP

```
┌────────────────────────────────────────────────────────────────────┐
│  СЛОЙ 1: Business Logic                                            │
│                                                                    │
│  @Service                                                          │
│  public class UserService {                                        │
│      @Autowired UserRepository userRepo;                           │
│                                                                    │
│      @Transactional                                                │
│      public User findById(Long id) {                               │
│          return userRepo.findById(id).orElseThrow();               │
│      }                                                             │
│  }                                                                 │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ Spring создаёт proxy для @Transactional
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  СЛОЙ 2: Transaction Interceptor (Spring AOP)                      │
│                                                                    │
│  1. transactionManager.getTransaction()                            │
│  2. Получаем Connection из HikariCP через DataSource               │
│  3. setAutoCommit(false)                                           │
│  4. Устанавливаем isolation level (например READ_COMMITTED)        │
│  5. Bind Connection к текущему thread (ThreadLocal)                │
│  6. Вызываем оригинальный метод                                    │
│  7. Если exception → rollback, иначе commit                        │
│  8. Return Connection в pool                                       │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  СЛОЙ 3: Spring Data JPA                                           │
│                                                                    │
│  UserRepository extends JpaRepository<User, Long>                  │
│                                                                    │
│  findById(1L) → SimpleJpaRepository → EntityManager.find(...)      │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  СЛОЙ 4: Hibernate (JPA implementation)                            │
│                                                                    │
│  1. Проверить Persistence Context (L1 cache):                      │
│     - Уже загружен User с id=1? → вернуть без SQL                  │
│     - Нет → SQL запрос                                             │
│  2. Проверить Second-Level cache (Ehcache, если включён)           │
│  3. Генерирует SQL:                                                │
│     SELECT u.id, u.name, u.email, u.created_at                     │
│     FROM users u WHERE u.id = ?                                    │
│  4. Bind параметров: ? = 1                                         │
│  5. Prepared statement (кэшируется)                                │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  СЛОЙ 5: JDBC API                                                  │
│                                                                    │
│  Connection conn = dataSource.getConnection();                     │
│  PreparedStatement ps = conn.prepareStatement(sql);                │
│  ps.setLong(1, 1L);                                                │
│  ResultSet rs = ps.executeQuery();                                 │
│  while (rs.next()) { ... }                                         │
│  rs.close(); ps.close(); conn.close();                             │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  СЛОЙ 6: HikariCP (connection pool)                                │
│                                                                    │
│  Pool configuration:                                               │
│    maximumPoolSize: 20                                             │
│    minimumIdle: 5                                                  │
│    connectionTimeout: 30s                                          │
│    idleTimeout: 600s                                               │
│    maxLifetime: 1800s                                              │
│                                                                    │
│  getConnection():                                                  │
│    1. Ищет idle connection в pool → возвращает                     │
│    2. Если нет и current < max → создаёт новый                     │
│    3. Если max reached → ждёт connectionTimeout → exception        │
│                                                                    │
│  close() на connection:                                            │
│    - НЕ закрывает физически                                        │
│    - Reset connection state (rollback pending tx)                  │
│    - Возвращает в pool                                             │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  СЛОЙ 7: PostgreSQL JDBC Driver (pgjdbc)                           │
│                                                                    │
│  1. Формирует PostgreSQL wire protocol messages:                   │
│                                                                    │
│     Parse message:                                                 │
│       'P' | length | statement_name | query | num_params | ...    │
│                                                                    │
│     Bind message:                                                  │
│       'B' | length | portal | statement | params...                │
│                                                                    │
│     Execute message:                                               │
│       'E' | length | portal | max_rows                             │
│                                                                    │
│     Sync message:                                                  │
│       'S' | length                                                 │
│                                                                    │
│  2. Отправляет через Socket                                        │
│  3. Читает ответ (RowDescription, DataRow, CommandComplete, ...)   │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ TCP socket
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  СЛОЙ 8: TCP/IP → PostgreSQL server (port 5432)                    │
│                                                                    │
│  В K8s:                                                            │
│    Service postgres-svc.knp.svc.cluster.local:5432                 │
│      → EndpointSlice → 10.244.5.10:5432 (Postgres Pod)             │
│                                                                    │
│  Wire protocol v3:                                                 │
│    - Messages имеют формат: byte1 (type) + int32 (length) + body   │
│    - Startup message для соединения                                │
│    - Authentication (MD5, SCRAM-SHA-256, GSS, LDAP, ...)           │
│    - После auth: Simple Query или Extended Query protocol          │
└────────────────────────────────────────────────────────────────────┘
```

---

## 2. POSTGRESQL SERVER — общая архитектура

```
┌────────────────────────────────────────────────────────────────────┐
│                    POSTGRESQL INSTANCE                             │
│                                                                    │
│   Один "PostgreSQL server" = один Unix процесс дерево:             │
│                                                                    │
│   postmaster (parent, PID 1)                                       │
│   ├─ Client backend 1 (per client connection)                      │
│   ├─ Client backend 2                                              │
│   ├─ Client backend N (max_connections=100 by default)             │
│   │                                                                │
│   ├─ Background workers:                                           │
│   │   ├─ Autovacuum launcher                                       │
│   │   │  └─ Autovacuum workers (autovacuum_max_workers=3)          │
│   │   ├─ Background Writer (bgwriter)                              │
│   │   ├─ WAL Writer (walwriter)                                    │
│   │   ├─ Checkpointer                                              │
│   │   ├─ Stats Collector (в PG 15+: cumulative statistics system)  │
│   │   ├─ Logger process                                            │
│   │   ├─ Archiver process                                          │
│   │   └─ WAL Sender processes (для replication, per replica)       │
│   │                                                                │
│   └─ Shared memory area:                                           │
│       ├─ shared_buffers (~25% RAM, обычно 4-8 GB)                  │
│       ├─ WAL buffers                                               │
│       ├─ CLOG buffers (commit log)                                 │
│       ├─ Lock table                                                │
│       └─ Statistics arrays                                         │
└────────────────────────────────────────────────────────────────────┘
```

### 2.1 Postmaster — главный процесс

```
┌────────────────────────────────────────────────────────────────────┐
│  POSTMASTER lifecycle                                              │
│                                                                    │
│  Startup:                                                          │
│    1. Читает postgresql.conf                                       │
│    2. Читает pg_hba.conf (access control rules)                    │
│    3. Аллоцирует shared memory (mmap)                              │
│    4. Инициализирует lock manager                                  │
│    5. Стартует background workers                                  │
│    6. Recovery если нужно (crash recovery через WAL)               │
│    7. Начинает listen на порту 5432                                │
│                                                                    │
│  Основная работа:                                                  │
│    - accept() на listening socket                                  │
│    - На каждый новый connection: fork() → backend process          │
│    - Не обрабатывает сам запросы!                                  │
│                                                                    │
│  На signals:                                                       │
│    SIGHUP → reload config                                          │
│    SIGTERM/SIGINT → graceful shutdown (fast/smart)                 │
│    SIGQUIT → immediate shutdown (без recovery)                     │
└────────────────────────────────────────────────────────────────────┘
```

### 2.2 Backend process — на каждого клиента

```
┌────────────────────────────────────────────────────────────────────┐
│  BACKEND process (один на TCP connection)                          │
│                                                                    │
│  При fork() от postmaster'а:                                       │
│    1. Аутентификация клиента (pg_hba.conf → md5/scram/ldap/...)    │
│    2. Инициализация backend memory (heap, work_mem, ...)           │
│    3. Прикрепление к shared memory                                 │
│    4. Loop:                                                        │
│         wait for message from client                               │
│         parse → plan → execute                                     │
│         send result                                                │
│                                                                    │
│  Memory per backend:                                               │
│    - work_mem (default 4MB) — для сортировок, hash                 │
│    - temp_buffers (default 8MB) — для temp tables                  │
│    - maintenance_work_mem (default 64MB) — для VACUUM, CREATE INDEX│
│    - процесс-специфичные структуры                                 │
│                                                                    │
│  Итог: один backend = 5-50 MB RAM.                                 │
│  100 backends = 500 MB - 5 GB.                                     │
│  Отсюда рекомендация: не более max_connections=200-500 на сервере. │
└────────────────────────────────────────────────────────────────────┘
```

### 2.3 Background processes — что каждый делает

```
┌────────────────────────────────────────────────────────────────────┐
│  BACKGROUND WRITER (bgwriter)                                      │
│                                                                    │
│  Задача: сбрасывать грязные страницы из buffer pool на диск        │
│  в промежутках между checkpoint'ами.                               │
│                                                                    │
│  Настройки:                                                        │
│    bgwriter_delay = 200ms       — период проверки                  │
│    bgwriter_lru_maxpages = 100  — макс страниц за раз              │
│                                                                    │
│  Помогает: избежать stall'ов backend'ов, которым нужен clean buffer│
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  WAL WRITER                                                        │
│                                                                    │
│  Задача: сбрасывать WAL buffers на диск (pg_wal/*).                │
│                                                                    │
│  Настройки:                                                        │
│    wal_writer_delay = 200ms                                        │
│    wal_writer_flush_after = 1MB                                    │
│                                                                    │
│  Работает вместе с backend'ами:                                    │
│    - Backend при commit делает fsync WAL (synchronous_commit=on)   │
│    - Между commits — WAL writer помогает выталкивать               │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  CHECKPOINTER                                                      │
│                                                                    │
│  Задача: периодически делать checkpoint — гарантировать что все    │
│  данные из shared_buffers сохранены на диск.                       │
│                                                                    │
│  Настройки:                                                        │
│    checkpoint_timeout = 5min      — макс интервал                  │
│    max_wal_size = 1GB             — trigger если WAL слишком боль. │
│    checkpoint_completion_target = 0.9  — растянуть на 90% времени  │
│                                                                    │
│  Checkpoint процесс:                                               │
│    1. Найти все dirty buffers                                      │
│    2. Write их на диск (постепенно, чтобы не создавать I/O storm)  │
│    3. fsync data files                                             │
│    4. Записать checkpoint record в WAL                             │
│    5. Обновить pg_control                                          │
│    6. Освободить старые WAL сегменты                               │
│                                                                    │
│  При recovery — начинает replay WAL с последнего checkpoint.       │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  AUTOVACUUM                                                        │
│                                                                    │
│  Launcher + Workers (autovacuum_max_workers=3 default).            │
│                                                                    │
│  Задачи:                                                           │
│    - VACUUM: пометить dead tuples как reusable                     │
│    - ANALYZE: обновить статистику для planner                      │
│    - Prevent transaction ID wraparound                             │
│                                                                    │
│  Настройки:                                                        │
│    autovacuum_naptime = 1min          — период проверки            │
│    autovacuum_vacuum_threshold = 50                                │
│    autovacuum_vacuum_scale_factor = 0.2                            │
│      → VACUUM когда изменилось 20% + 50 tuples                     │
│                                                                    │
│  Работает так:                                                     │
│    - Launcher каждую минуту читает pg_stat_all_tables              │
│    - Находит таблицы требующие VACUUM/ANALYZE                      │
│    - Стартует worker (до 3 параллельно)                            │
│    - Worker: SELECT + UPDATE в pg_class, pg_class visibility map   │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  WAL SENDER (для replication)                                      │
│                                                                    │
│  Один процесс на каждый replica.                                   │
│  Читает pg_wal/ файлы, стримит их через TCP на standby.            │
│                                                                    │
│  Standby side: WAL Receiver process, applies WAL locally.          │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. ПУТЬ SQL-ЗАПРОСА — от Parse до Execute

Что делает backend, когда получает `SELECT * FROM users WHERE id = 1`:

```
┌────────────────────────────────────────────────────────────────────┐
│  ФАЗА 1: PARSER                                                    │
│                                                                    │
│  Input: "SELECT * FROM users WHERE id = 1"                         │
│                                                                    │
│  Lexer: tokens                                                     │
│    SELECT, *, FROM, users, WHERE, id, =, 1                         │
│                                                                    │
│  Parser: AST (Abstract Syntax Tree)                                │
│    SelectStmt                                                      │
│    ├── targetList: [*]                                             │
│    ├── fromClause: [RangeVar("users")]                             │
│    └── whereClause: A_Expr(=, ColumnRef("id"), IntConst(1))        │
│                                                                    │
│  Проверка синтаксиса. Если ошибка → syntax error.                  │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  ФАЗА 2: ANALYZER                                                  │
│                                                                    │
│  1. Разрешает имена (users → pg_class oid).                        │
│  2. Проверяет permissions (SELECT на таблицу?).                    │
│  3. Проверяет типы (id — integer, 1 — integer, OK).                │
│  4. Разворачивает *: users.id, users.name, users.email, ...        │
│                                                                    │
│  Output: Query tree (типизированный AST).                          │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  ФАЗА 3: REWRITER                                                  │
│                                                                    │
│  Применяет правила (rules):                                        │
│    - Views: раскрывает запрос view в исходный                      │
│    - Row-level security policies                                   │
│    - Custom rules (CREATE RULE)                                    │
│                                                                    │
│  Для простого запроса — nothing to rewrite.                        │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  ФАЗА 4: PLANNER / OPTIMIZER                                       │
│                                                                    │
│  Задача: выбрать самый дешёвый способ выполнить запрос.            │
│                                                                    │
│  Входы:                                                            │
│    - Query tree                                                    │
│    - Statistics (pg_statistic, pg_class.reltuples, ...)            │
│    - Config (random_page_cost, work_mem, ...)                      │
│    - Индексы (pg_index)                                            │
│                                                                    │
│  Что делает:                                                       │
│    1. Генерирует candidate plans:                                  │
│       - Seq Scan on users, Filter: id=1                            │
│       - Index Scan on users_pkey                                   │
│    2. Для каждого plan вычисляет cost:                             │
│       cost = startup_cost + total_cost                             │
│       (в abstract units, привязка через seq_page_cost=1.0)         │
│    3. Выбирает cheapest plan.                                      │
│                                                                    │
│  Пример plan:                                                      │
│    Index Scan using users_pkey on users                            │
│      Index Cond: (id = 1)                                          │
│      Cost: 0.29..8.30 rows=1 width=64                              │
│                                                                    │
│  Cost estimation формула:                                          │
│    Index Scan cost = idx_random_page_cost * pages_read             │
│                    + cpu_index_tuple_cost * rows                   │
│                    + cpu_operator_cost * conditions                │
│                    + cpu_tuple_cost * rows                         │
│                                                                    │
│  Для JOIN'ов — оценивает Nested Loop / Hash Join / Merge Join      │
│  выбирает лучший, учитывая selectivity.                            │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  ФАЗА 5: EXECUTOR                                                  │
│                                                                    │
│  Выполняет plan tree сверху вниз (pipeline).                       │
│                                                                    │
│  Для Index Scan:                                                   │
│    1. Открыть индекс users_pkey (B-tree).                          │
│    2. Спуститься по B-tree к листу с ключом 1.                     │
│    3. Получить TID (Tuple Identifier: block_number, offset).       │
│    4. Прочитать страницу heap по этому TID.                        │
│    5. Извлечь tuple.                                               │
│    6. Проверить MVCC visibility (xmin, xmax vs snapshot).          │
│    7. Вернуть tuple клиенту через socket.                          │
│                                                                    │
│  Каждый шаг может задействовать buffer pool (см. слой 4).          │
└────────────────────────────────────────────────────────────────────┘
```

---

## 4. BUFFER POOL — сердце PostgreSQL

Никакие данные не читаются с диска напрямую. Всё проходит через **shared_buffers**.

```
┌────────────────────────────────────────────────────────────────────┐
│  SHARED_BUFFERS (in-memory cache of pages)                         │
│                                                                    │
│  Настройка: shared_buffers = 4GB (обычно 25% RAM)                  │
│                                                                    │
│  Организация: массив buffers по 8 KB each.                         │
│    4 GB / 8 KB = 524288 buffers                                    │
│                                                                    │
│  Каждый buffer:                                                    │
│    ┌────────────────────────────────────────┐                      │
│    │  Header:                               │                      │
│    │    - tag: relation OID + block number  │                      │
│    │    - state: pin count, dirty, valid    │                      │
│    │    - lock: content lock (shared/excl.) │                      │
│    ├────────────────────────────────────────┤                      │
│    │  Page data (8 KB):                     │                      │
│    │    ...                                 │                      │
│    └────────────────────────────────────────┘                      │
│                                                                    │
│  Hash table:                                                       │
│    (relation, block) → buffer_id                                   │
│  Позволяет быстро найти "уже ли эта страница в памяти".            │
└────────────────────────────────────────────────────────────────────┘

Что происходит при чтении tuple:
        │
        ▼
┌────────────────────────────────────────────────────────────────────┐
│  1. Backend хочет прочитать блок 42 таблицы users (oid=16384)     │
│  2. Ищет в hash table: (16384, 42)                                 │
│  3. HIT → buffer_id найден:                                        │
│     - pin buffer (increment pin count)                             │
│     - read data                                                    │
│     - unpin                                                        │
│                                                                    │
│  4. MISS → нужно загрузить:                                        │
│     - Ищет free buffer (или eviction candidate)                    │
│     - Eviction: clock-sweep algorithm                              │
│       (аналог LRU, но проще для concurrent access)                 │
│     - Если candidate dirty → write to disk сначала                 │
│     - read() из disk (или из OS page cache) → в buffer             │
│     - Обновляет hash table                                         │
│                                                                    │
│  Замер: pg_stat_database.blks_hit / blks_read = cache hit ratio    │
│  Норма: 95-99% для OLTP workload.                                  │
└────────────────────────────────────────────────────────────────────┘
```

### 4.1 Двойное кэширование — PostgreSQL vs OS page cache

```
┌────────────────────────────────────────────────────────────────────┐
│  Приложение: SELECT * FROM users WHERE id=1                        │
│      │                                                             │
│      ▼                                                             │
│  1. shared_buffers (PostgreSQL memory)                             │
│      │  hit? → return                                              │
│      │  miss ↓                                                     │
│      ▼                                                             │
│  2. OS Page Cache (Linux kernel memory)                            │
│      │  hit? → copy в shared_buffers → return                      │
│      │  miss ↓                                                     │
│      ▼                                                             │
│  3. Disk (SSD)                                                     │
│      │  read → OS page cache → shared_buffers → return             │
│      ▼                                                             │
│      ~100 μs (NVMe SSD)                                            │
└────────────────────────────────────────────────────────────────────┘

Данные хранятся дважды в RAM: в shared_buffers PG и в OS page cache.
Не проблема — PG использует small fraction, page cache динамически.

Совет: shared_buffers = 25% RAM. Больше не всегда лучше:
  - PG lock contention растёт с числом buffers
  - Page cache тоже нужен
  - На больших shared_buffers eviction медленнее

Для heavy write workload — можно увеличить до 40%.
Для read-heavy — 25% + доверять OS page cache.
```

---

## 5. ФИЗИЧЕСКОЕ ХРАНЕНИЕ — pages, tuples, TOAST

```
┌────────────────────────────────────────────────────────────────────┐
│  СТРУКТУРА PAGE (8 KB default)                                     │
│                                                                    │
│  ┌────────────────────────────────────────────────┐                │
│  │  Page Header (24 bytes)                        │                │
│  │    - pd_lsn: last WAL LSN                      │                │
│  │    - pd_checksum: page checksum                │                │
│  │    - pd_flags: various flags                   │                │
│  │    - pd_lower: end of item pointers            │                │
│  │    - pd_upper: start of tuples                 │                │
│  │    - pd_special: reserved                      │                │
│  ├────────────────────────────────────────────────┤                │
│  │  Item Pointers (ItemIdData, по 4 bytes each)   │  ↓ растут вниз │
│  │    - offset, length, flags                     │                │
│  │    - указывают на tuples ниже                  │                │
│  ├─── Free space (может быть 0-90% страницы) ─────┤                │
│  │                                                │                │
│  ├────────────────────────────────────────────────┤                │
│  │  Tuple N                                       │  ↑ растут вверх│
│  │  ...                                           │                │
│  │  Tuple 2                                       │                │
│  │  Tuple 1                                       │                │
│  ├────────────────────────────────────────────────┤                │
│  │  Special space (index-specific data)           │                │
│  └────────────────────────────────────────────────┘                │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  СТРУКТУРА TUPLE (row в таблице)                                   │
│                                                                    │
│  ┌────────────────────────────────────────────────┐                │
│  │  Tuple Header (23 bytes)                       │                │
│  │    - xmin: transaction ID that inserted        │  ← MVCC        │
│  │    - xmax: transaction ID that deleted/updated │  ← MVCC        │
│  │    - cmin/cmax: command IDs                    │                │
│  │    - ctid: self-pointer (block, offset)        │                │
│  │    - infomask: flags                           │                │
│  │    - null bitmap (если nullable columns)       │                │
│  ├────────────────────────────────────────────────┤                │
│  │  User data:                                    │                │
│  │    id: int4   (4 bytes)                        │                │
│  │    name: varchar → text pointer или inline     │                │
│  │    email: varchar                              │                │
│  │    created_at: timestamptz (8 bytes)           │                │
│  └────────────────────────────────────────────────┘                │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  TOAST (The Oversized-Attribute Storage Technique)                 │
│                                                                    │
│  Если tuple > 8 KB / 4 = 2 KB — не влезает в page.                 │
│  Values более 2 KB сохраняются в TOAST-таблице:                    │
│                                                                    │
│  ┌────────────────┐          ┌────────────────────┐                │
│  │  users         │          │  pg_toast_16384    │                │
│  │  id  name  ... │          │  chunk_id chunk_no │                │
│  │  1   TOAST────►│──────────►  16385    0        │                │
│  │      pointer   │          │  16385    1        │                │
│  │      (18 bytes)│          │  16385    2   ...  │                │
│  └────────────────┘          └────────────────────┘                │
│                                                                    │
│  4 стратегии для каждого column:                                   │
│    - PLAIN: inline, без сжатия (только для fixed-width)            │
│    - EXTENDED: сжать и TOAST если нужно (default для varlen)       │
│    - EXTERNAL: TOAST, не сжимать                                   │
│    - MAIN: сжать inline, TOAST только если не влезает              │
│                                                                    │
│  Компрессия: PGLZ или LZ4 (PG 14+, lz4 быстрее).                   │
│  Chunking: значения разбиваются на chunks по 2000 bytes.           │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  ФАЙЛОВАЯ СТРУКТУРА                                                │
│                                                                    │
│  /var/lib/postgresql/data/base/<database_oid>/                     │
│                                                                    │
│  Каждая таблица = набор файлов:                                    │
│    16384         ← main fork (heap tuples)                         │
│    16384.1       ← ещё один segment (файлы до 1 GB)                │
│    16384.2                                                         │
│    ...                                                             │
│    16384_fsm     ← Free Space Map (где есть место для новых)       │
│    16384_vm      ← Visibility Map (для index-only scans)           │
│    16384_init    ← template для unlogged tables                    │
│                                                                    │
│  Индексы — свои файлы:                                             │
│    16385         ← index main fork                                 │
│    16385_fsm                                                       │
│                                                                    │
│  OID таблицы можно найти:                                          │
│    SELECT oid FROM pg_class WHERE relname = 'users';               │
│    Или: SELECT pg_relation_filepath('users');                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## 6. WAL — Write-Ahead Log

Ключевая структура для durability. Любое изменение сначала записывается в WAL, потом (позже) применяется к data files.

```
┌────────────────────────────────────────────────────────────────────┐
│  ПРИНЦИП WAL                                                       │
│                                                                    │
│  Правило: **сначала лог, потом данные**.                           │
│                                                                    │
│  При UPDATE:                                                       │
│    1. Модифицируется страница в shared_buffers (не на диске!)      │
│    2. Записывается WAL record: "was old, now new"                  │
│    3. WAL fsync'ится (при commit, если synchronous_commit=on)      │
│    4. Только позже (checkpoint) грязная страница пишется в data    │
│                                                                    │
│  Зачем: если сервер упадёт между шагами 3 и 4, при recovery        │
│  прочитаем WAL и применим все изменения. Durability гарантирована. │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  WAL FILES                                                         │
│                                                                    │
│  /var/lib/postgresql/data/pg_wal/                                  │
│  ├── 000000010000000000000001    ← 16 MB segment                   │
│  ├── 000000010000000000000002                                      │
│  ├── 000000010000000000000003                                      │
│  ├── ...                                                           │
│  └── archive_status/                                               │
│                                                                    │
│  Имя файла: <timeline><logical_id><segment>                        │
│                                                                    │
│  LSN (Log Sequence Number) — позиция в WAL:                        │
│    0/1234ABCD                                                      │
│    ↑ log id  ↑ offset в log id                                     │
│                                                                    │
│  pg_current_wal_lsn() — текущая позиция.                           │
│  pg_wal_lsn_diff(lsn1, lsn2) — разница в bytes.                    │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  WAL RECORD                                                        │
│                                                                    │
│  Каждая запись содержит:                                           │
│    - LSN                                                           │
│    - Transaction ID (xid)                                          │
│    - Type: INSERT / UPDATE / DELETE / COMMIT / CHECKPOINT / ...    │
│    - Relation info                                                 │
│    - Data (before/after image или delta)                           │
│    - CRC checksum                                                  │
│                                                                    │
│  Full-page writes: первое изменение страницы после checkpoint —    │
│  пишется вся страница целиком (защита от torn writes).             │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  FLOW: COMMIT транзакции                                           │
│                                                                    │
│  BEGIN;                                                            │
│  UPDATE users SET name='Berik' WHERE id=1;                         │
│    → Modified page в shared_buffers                                │
│    → WAL record в WAL buffer                                       │
│  COMMIT;                                                           │
│    → WAL commit record                                             │
│    → fsync WAL до текущего LSN     ← EXPENSIVE (~1 ms NVMe)        │
│    → return OK клиенту                                             │
│                                                                    │
│  Настройка synchronous_commit:                                     │
│    on (default) — fsync перед return                               │
│    off — return без fsync (риск потери 200 ms данных на crash)     │
│    remote_apply — ждём apply на synchronous replica                │
│    remote_write — ждём write (без fsync) на replica                │
│    local — как on, но игнорируем replicas                          │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  CRASH RECOVERY                                                    │
│                                                                    │
│  Сервер упал. Что происходит при следующем старте:                 │
│                                                                    │
│  1. postmaster читает pg_control:                                  │
│     - last checkpoint LSN                                          │
│     - last redo LSN                                                │
│                                                                    │
│  2. Recovery mode: replays WAL от checkpoint LSN до конца.         │
│     Для каждой WAL record:                                         │
│       - Читает страницу с диска                                    │
│       - Применяет изменение из WAL                                 │
│       - Не пишет обратно (сделает следующий checkpoint)            │
│                                                                    │
│  3. Reached end of WAL — recovery complete.                        │
│                                                                    │
│  4. Открывает для connections.                                     │
│                                                                    │
│  Время recovery = f(размер WAL с checkpoint'а).                    │
│  checkpoint_timeout=5min → recovery обычно < 5 минут.              │
└────────────────────────────────────────────────────────────────────┘
```

---

## 7. MVCC — Multi-Version Concurrency Control

Как PostgreSQL достигает высокого concurrency без блокировок читателей.

```
┌────────────────────────────────────────────────────────────────────┐
│  ПРИНЦИП MVCC                                                      │
│                                                                    │
│  Каждая строка имеет несколько "версий" (tuples).                  │
│  Каждая версия помечена xmin (создатель) и xmax (удалятель).       │
│                                                                    │
│  Транзакция видит только те tuples, которые:                       │
│    xmin.committed && xmin < snapshot.xmin && xmax либо null,       │
│    либо not committed / > snapshot                                 │
│                                                                    │
│  Читатели не блокируют писателей, писатели не блокируют читателей. │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  ПРИМЕР                                                            │
│                                                                    │
│  T=0: users table                                                  │
│    ctid: (0,1)  xmin=100  xmax=0    id=1  name='Alex'              │
│                                                                    │
│  T=1: TX 101 обновляет: UPDATE users SET name='Berik' WHERE id=1  │
│                                                                    │
│  Что происходит:                                                   │
│    1. Старая версия: xmax=101 (deleted by 101)                     │
│    2. Новая версия создаётся:                                      │
│       ctid: (0,2)  xmin=101  xmax=0    id=1  name='Berik'          │
│    3. Обе версии на диске одновременно!                            │
│                                                                    │
│  T=2: TX 102 (открыта до commit 101):                              │
│    SELECT * FROM users WHERE id=1                                  │
│    → видит старую версию (xmin=100 committed до 102)               │
│    → name='Alex'                                                   │
│                                                                    │
│  T=3: TX 101 commits.                                              │
│                                                                    │
│  T=4: TX 103 (новая):                                              │
│    SELECT * FROM users WHERE id=1                                  │
│    → xmin=100, xmax=101 committed → tuple deleted, skip            │
│    → xmin=101 committed, xmax=0 → visible                          │
│    → name='Berik'                                                  │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  ПРОБЛЕМЫ MVCC                                                     │
│                                                                    │
│  1. **Dead tuples**: старые версии, которые никто уже не видит.    │
│     Занимают место, замедляют scan.                                │
│     Решение: VACUUM (помечает как reusable, но не даёт места OS).  │
│                                                                    │
│  2. **Bloat**: если UPDATE'ов много, а VACUUM не поспевает —       │
│     таблица разрастается. `pg_stat_user_tables.n_dead_tup`.        │
│     Решение: настроить autovacuum aggressively, pg_repack.         │
│                                                                    │
│  3. **XID wraparound**: xid — 32-bit. При 2^32 транзакций          │
│     wraparound → old tuples become "future" → visible incorrectly. │
│     Решение: autovacuum freezes old xid before wraparound.         │
│                                                                    │
│  4. **HOT (Heap-Only Tuple) updates**: если UPDATE не меняет       │
│     indexed columns, новая версия на той же странице, без index    │
│     update. Оптимизация.                                           │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  ISOLATION LEVELS в PostgreSQL                                     │
│                                                                    │
│  READ UNCOMMITTED — не поддерживается (эквивалент READ COMMITTED). │
│                                                                    │
│  READ COMMITTED (default):                                         │
│    - Каждый statement видит committed data на его старте.          │
│    - Разные statements в одной транзакции могут видеть разные      │
│      snapshots.                                                    │
│    - Anomalies: non-repeatable read, phantom read.                 │
│                                                                    │
│  REPEATABLE READ:                                                  │
│    - Один snapshot на всю транзакцию (первый statement).           │
│    - Anomalies: только serialization anomaly.                      │
│                                                                    │
│  SERIALIZABLE:                                                     │
│    - Гарантирует что результат = как будто транзакции шли по одной.│
│    - Реализация: SSI (Serializable Snapshot Isolation).            │
│    - Если конфликт обнаружен → serialization failure → retry.      │
└────────────────────────────────────────────────────────────────────┘
```

---

## 8. ИНДЕКСЫ — как работают

PostgreSQL поддерживает несколько типов индексов. Каждый — для своих задач.

```
┌────────────────────────────────────────────────────────────────────┐
│  B-TREE (default) — сбалансированное дерево                        │
│                                                                    │
│                          Root                                      │
│                     ┌─────────────┐                                │
│                     │  50  |  100 │                                │
│                     └──┬───┴──┬───┘                                │
│                        │      │                                    │
│                ┌───────┘      └───────┐                            │
│                ▼                      ▼                            │
│         Internal              Internal                             │
│      ┌──────────────┐     ┌──────────────┐                         │
│      │ 20 | 35 | 45 │     │ 70 | 85 | 95 │                         │
│      └───┬──┬──┬──┬─┘     └──────────────┘                         │
│          │  │  │  │                                                │
│          ▼  ▼  ▼  ▼                                                │
│         Leaf pages (linked list)                                   │
│         ┌───────────────────────────────────────┐                  │
│         │ [key1→ctid][key2→ctid][...] → next    │                  │
│         └───────────────────────────────────────┘                  │
│                                                                    │
│  Использования:                                                    │
│    - Equality (=), range (<, >, BETWEEN)                           │
│    - ORDER BY по индексированной колонке                           │
│    - JOIN на equality                                              │
│                                                                    │
│  Depth обычно 3-5 уровней даже для миллиардов строк.               │
│  Поиск: O(log n).                                                  │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  HASH — только equality                                            │
│                                                                    │
│  hash(key) → bucket → chained list                                 │
│                                                                    │
│  Быстрее B-tree для equality на 10-20%.                            │
│  Не поддерживает range queries.                                    │
│  Редко используется.                                               │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  GIN (Generalized Inverted Index) — inverted index                 │
│                                                                    │
│  Для composite types: arrays, jsonb, tsvector.                     │
│                                                                    │
│  Пример: тексты                                                    │
│    row 1: "hello world"                                            │
│    row 2: "world peace"                                            │
│                                                                    │
│  Inverted index:                                                   │
│    "hello" → [row 1]                                               │
│    "peace" → [row 2]                                               │
│    "world" → [row 1, row 2]                                        │
│                                                                    │
│  Query "WHERE text @@ 'world'" — быстро находит матчинг rows.      │
│                                                                    │
│  Использования: full-text search, jsonb containment (@>), arrays.  │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  GiST (Generalized Search Tree) — extensible tree                  │
│                                                                    │
│  Для geometric types, ranges, distance queries.                    │
│                                                                    │
│  Использования: PostGIS (spatial), pg_trgm (similarity), tsrange.  │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  BRIN (Block Range Index) — very small                             │
│                                                                    │
│  Для очень больших таблиц с naturally sorted data                  │
│  (например, time-series, sensor data).                             │
│                                                                    │
│  Хранит min/max для каждого блока страниц.                         │
│  Query проверяет: какие блоки могут содержать value?               │
│  Читает только эти блоки.                                          │
│                                                                    │
│  Размер: KB для GB таблицы.                                        │
│  Точность ниже, чем B-tree, но disk savings огромные.              │
└────────────────────────────────────────────────────────────────────┘
```

### 8.1 Как индекс используется в query

```
Query: SELECT * FROM users WHERE id = 42
        │
        ▼
Planner: cost estimate → Index Scan on users_pkey (cheapest)
        │
        ▼
Executor:
  1. Открыть users_pkey (B-tree)
  2. Спуститься от root к leaf:
     - Root: 50 | 100 → идём налево (42 < 50)
     - Internal: 20 | 35 | 45 → идём к leaf с 42
     - Leaf: находим entry [42 → ctid(0,17)]
  3. Читаем страницу heap (block 0)
  4. Извлекаем tuple по offset 17
  5. MVCC check (xmin/xmax)
  6. Return tuple
        │
        ▼
Через buffer pool:
  - Читаем index page (обычно в памяти)
  - Читаем heap page (может быть miss → OS page cache → disk)
```

---

## 9. TRANSACTIONS и LOCKS

```
┌────────────────────────────────────────────────────────────────────┐
│  ЖИЗНЕННЫЙ ЦИКЛ ТРАНЗАКЦИИ                                         │
│                                                                    │
│  BEGIN;                                                            │
│    → xid не выделяется сразу                                       │
│    → snapshot запоминается при первом statement                    │
│                                                                    │
│  UPDATE users SET name='X' WHERE id=1;                             │
│    → xid выделяется (например 100)                                 │
│    → берётся row-level lock на tuple                               │
│    → создаётся новая версия tuple с xmin=100                       │
│    → пишется WAL record                                            │
│                                                                    │
│  COMMIT;                                                           │
│    → WAL commit record                                             │
│    → fsync                                                         │
│    → pg_xact: mark xid 100 as committed                            │
│    → release row-level locks                                       │
│    → notification в pg_stat_activity                               │
│                                                                    │
│  ROLLBACK;                                                         │
│    → WAL abort record                                              │
│    → pg_xact: mark xid 100 as aborted                              │
│    → new tuples остаются на диске, но dead (VACUUM уберёт)         │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  8 LOCK MODES (table-level)                                        │
│                                                                    │
│  От слабого к сильному:                                            │
│                                                                    │
│  1. ACCESS SHARE       — SELECT                                    │
│  2. ROW SHARE          — SELECT FOR UPDATE, FOR SHARE              │
│  3. ROW EXCLUSIVE      — INSERT/UPDATE/DELETE                      │
│  4. SHARE UPDATE EXCL. — VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY│
│  5. SHARE              — CREATE INDEX (без CONCURRENTLY)           │
│  6. SHARE ROW EXCL.    — CREATE TRIGGER                            │
│  7. EXCLUSIVE          — REFRESH MAT VIEW CONCURRENTLY             │
│  8. ACCESS EXCLUSIVE   — DROP, TRUNCATE, ALTER TABLE, REINDEX      │
│                                                                    │
│  Совместимость (упрощённо):                                        │
│    - 1-3 совместимы между собой (OLTP параллелит)                  │
│    - 4-8 блокируют DML или SELECT                                  │
│    - 8 блокирует всё                                               │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  ROW-LEVEL LOCKS                                                   │
│                                                                    │
│  Хранятся в самом tuple (xmax + infomask), не в отдельной table.   │
│                                                                    │
│  4 режима:                                                         │
│    FOR KEY SHARE      — можно всё кроме изменения key columns      │
│    FOR SHARE          — можно SELECT, нельзя UPDATE/DELETE         │
│    FOR NO KEY UPDATE  — UPDATE non-key columns                     │
│    FOR UPDATE         — эксклюзивный, для writers                  │
│                                                                    │
│  SELECT ... FOR UPDATE SKIP LOCKED — job queue pattern             │
│  SELECT ... FOR UPDATE NOWAIT — fail-fast                          │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  ADVISORY LOCKS — application-level                                │
│                                                                    │
│  SELECT pg_advisory_lock(12345);                                   │
│    -- do work                                                      │
│  SELECT pg_advisory_unlock(12345);                                 │
│                                                                    │
│  Использования: distributed locks (ShedLock),                      │
│  migration coordination (Liquibase, Flyway).                       │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  ДИАГНОСТИКА BLOCKING                                              │
│                                                                    │
│  SELECT pid, state, wait_event_type, wait_event,                   │
│         NOW() - xact_start AS duration, query                      │
│  FROM pg_stat_activity WHERE state != 'idle';                      │
│                                                                    │
│  SELECT blocked.pid, blocking.pid,                                 │
│         blocked.query, blocking.query                              │
│  FROM pg_stat_activity blocked                                     │
│  JOIN pg_stat_activity blocking                                    │
│    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));           │
│                                                                    │
│  Убить сессию:                                                     │
│    SELECT pg_cancel_backend(pid);   -- soft (SIGINT)               │
│    SELECT pg_terminate_backend(pid); -- hard (SIGTERM)             │
└────────────────────────────────────────────────────────────────────┘
```

---

## 10. CONNECTION POOLING — HikariCP + PgBouncer

```
┌────────────────────────────────────────────────────────────────────┐
│  БЕЗ POOLING                                                       │
│                                                                    │
│  Каждый HTTP request → new JDBC connection → new PostgreSQL fork() │
│    - fork() дорогой (~1-5 ms)                                      │
│    - Backend memory (~5-50 MB)                                     │
│    - Ограничение max_connections                                   │
│                                                                    │
│  При 1000 concurrent requests → 1000 backends → OOM.               │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  APPLICATION-LEVEL POOL: HikariCP                                  │
│                                                                    │
│  Внутри приложения:                                                │
│    minimumIdle: 5                                                  │
│    maximumPoolSize: 20                                             │
│    connectionTimeout: 30s                                          │
│    idleTimeout: 600s                                               │
│    maxLifetime: 1800s (30 min)                                     │
│                                                                    │
│  При старте: 5 connections (idle).                                 │
│  При нагрузке: до 20 concurrent.                                   │
│  Idle > 10 min → закрываются.                                      │
│  Lifetime > 30 min → закрываются и создаются новые                 │
│    (защита от stale connections).                                  │
│                                                                    │
│  На K8s: если 4 Pod'а по 20 connections = 80 к БД.                 │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  MIDDLE-LEVEL POOL: PgBouncer                                      │
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                          │
│  │ App Pod1 │  │ App Pod2 │  │ App Pod3 │                          │
│  │ 20 conns │  │ 20 conns │  │ 20 conns │                          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                          │
│       └─────────────┼─────────────┘                                │
│                     │                                              │
│                60 conns                                            │
│                     ▼                                              │
│              ┌──────────────┐                                      │
│              │  PgBouncer   │                                      │
│              │ pool_size=20 │                                      │
│              └──────┬───────┘                                      │
│                     │                                              │
│                20 conns                                            │
│                     ▼                                              │
│              ┌──────────────┐                                      │
│              │  PostgreSQL  │                                      │
│              │ 20 backends  │                                      │
│              └──────────────┘                                      │
│                                                                    │
│  Режимы PgBouncer:                                                 │
│    - session: connection assigned до disconnect (не мультипл.)     │
│    - transaction: connection per transaction (стандарт для веб)    │
│    - statement: connection per statement (агрессивно, ограничения) │
│                                                                    │
│  Transaction mode: 3000 клиентов → 20 backends к Postgres.         │
│  Ограничение: нельзя использовать session-level features           │
│    (prepared statements, SET, LISTEN/NOTIFY, temp tables).         │
└────────────────────────────────────────────────────────────────────┘

Рекомендация для КНП:
  App HikariCP maximumPoolSize = 10-20
  На 4 Pod'а = 40-80 connections
  Если больше сервисов — ставить PgBouncer между.
```

---

## 11. REPLICATION — streaming и logical

```
┌────────────────────────────────────────────────────────────────────┐
│  STREAMING REPLICATION (physical)                                  │
│                                                                    │
│  ┌─────────────┐                    ┌─────────────┐                │
│  │  PRIMARY    │                    │  STANDBY    │                │
│  │             │                    │             │                │
│  │  Backend    │                    │  Startup    │                │
│  │  writes WAL │                    │  process    │                │
│  │       │     │                    │       │     │                │
│  │       ▼     │                    │       ▼     │                │
│  │  WAL Sender │──── stream ───────►│ WAL Receiver│                │
│  │             │  (pg_wal records)  │       │     │                │
│  │             │                    │       ▼     │                │
│  │             │                    │  Apply WAL  │                │
│  │             │                    │  локально   │                │
│  └─────────────┘                    └─────────────┘                │
│                                                                    │
│  Standby можно использовать для read-only queries                  │
│  (hot standby).                                                    │
│                                                                    │
│  Задержка (lag): обычно миллисекунды в LAN, зависит от нагрузки.   │
│  Мониторинг: pg_stat_replication (на primary).                     │
│    SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn,      │
│           replay_lsn,                                              │
│           pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes       │
│    FROM pg_stat_replication;                                       │
│                                                                    │
│  Modes:                                                            │
│    asynchronous (default) — primary не ждёт, минимальная latency   │
│    synchronous — primary ждёт flush на synchronous replica         │
│      (durable, но latency растёт на RTT)                           │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  LOGICAL REPLICATION                                               │
│                                                                    │
│  Не физические WAL records, а логические изменения (row-level).    │
│                                                                    │
│  Настройка:                                                        │
│    Publisher:                                                      │
│      CREATE PUBLICATION my_pub FOR TABLE users, orders;            │
│    Subscriber:                                                     │
│      CREATE SUBSCRIPTION my_sub                                    │
│      CONNECTION 'host=primary port=5432 dbname=...'                │
│      PUBLICATION my_pub;                                           │
│                                                                    │
│  Плюсы:                                                            │
│    - Selective replication (только определённые tables)            │
│    - Между разными версиями PostgreSQL                             │
│    - Между разными schemas                                         │
│    - Multi-master возможен                                         │
│                                                                    │
│  Минусы:                                                           │
│    - Медленнее streaming                                           │
│    - Не replicate DDL                                              │
│    - Sequence values не replicate autoматически                    │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  FAILOVER                                                          │
│                                                                    │
│  Primary упал → нужно promote standby в primary.                   │
│                                                                    │
│  Manual: pg_ctl promote                                            │
│                                                                    │
│  Automated: Patroni / repmgr / cloud-native (RDS, Cloud SQL).      │
│    - Detect primary failure                                        │
│    - Elect best standby (наибольший LSN)                           │
│    - Promote to primary                                            │
│    - Update DNS / service endpoints                                │
│    - Reconfigure other standbys к новому primary                   │
│                                                                    │
│  Split brain защита: consensus через etcd / Consul / Zookeeper.    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 12. BACKUP и RECOVERY

```
┌────────────────────────────────────────────────────────────────────┐
│  ТИПЫ BACKUP                                                       │
│                                                                    │
│  1. LOGICAL BACKUP (pg_dump)                                       │
│     - Dump SQL statements                                          │
│     - Portable между versions                                      │
│     - Медленно для больших БД                                      │
│     - Восстановление тоже медленное                                │
│                                                                    │
│     $ pg_dump -Fc dbname > backup.dump                             │
│     $ pg_restore -d newdb backup.dump                              │
│                                                                    │
│  2. PHYSICAL BACKUP (pg_basebackup)                                │
│     - Копия data directory + WAL                                   │
│     - Быстро для больших БД                                        │
│     - Только для этой версии PostgreSQL                            │
│                                                                    │
│     $ pg_basebackup -D /backup -Ft -X stream                       │
│                                                                    │
│  3. CONTINUOUS ARCHIVING (WAL archiving)                           │
│     - pg_basebackup + все WAL от того момента                      │
│     - Позволяет Point-In-Time Recovery (PITR)                      │
│                                                                    │
│     archive_command = 'cp %p /backup/wal/%f'                       │
│                                                                    │
│  4. SNAPSHOT-BASED (LVM, ZFS, cloud snapshots)                     │
│     - Filesystem-level                                             │
│     - Мгновенный при использовании copy-on-write                   │
│     - Требует consistent state (pg_start_backup / pg_stop_backup)  │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  POINT-IN-TIME RECOVERY (PITR)                                     │
│                                                                    │
│  Восстановление БД до конкретной точки в прошлом.                  │
│                                                                    │
│  Требования:                                                       │
│    - Physical base backup                                          │
│    - Все WAL с того момента                                        │
│                                                                    │
│  Процесс:                                                          │
│    1. Restore base backup                                          │
│    2. Настроить restore_command в postgresql.conf:                 │
│       restore_command = 'cp /backup/wal/%f %p'                     │
│    3. recovery_target_time = '2026-09-22 14:30:00'                 │
│    4. Startup — apply WAL до этого timestamp                       │
│    5. Promote                                                      │
│                                                                    │
│  Позволяет откатить случайное DELETE FROM users;                   │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  BACKUP TOOLS                                                      │
│                                                                    │
│  - pgBackRest — enterprise-grade, incremental, parallel            │
│  - Barman — PostgreSQL admin friendly                              │
│  - WAL-G — cloud-native (S3, Azure, GCP)                           │
│  - Native pg_basebackup + archiving                                │
└────────────────────────────────────────────────────────────────────┘
```

---

## 13. МОНИТОРИНГ — что смотреть в проде

```
┌────────────────────────────────────────────────────────────────────┐
│  КЛЮЧЕВЫЕ МЕТРИКИ                                                  │
│                                                                    │
│  1. Cache hit ratio                                                │
│     SELECT sum(heap_blks_hit) / sum(heap_blks_hit + heap_blks_read)│
│     FROM pg_statio_user_tables;                                    │
│     Норма: > 0.95                                                  │
│                                                                    │
│  2. Slow queries                                                   │
│     Включить: shared_preload_libraries='pg_stat_statements'        │
│     SELECT query, calls, total_exec_time, mean_exec_time           │
│     FROM pg_stat_statements                                        │
│     ORDER BY total_exec_time DESC LIMIT 20;                        │
│                                                                    │
│  3. Bloat                                                          │
│     SELECT schemaname, tablename,                                  │
│            n_dead_tup, n_live_tup,                                 │
│            n_dead_tup::float / NULLIF(n_live_tup, 0) AS ratio      │
│     FROM pg_stat_user_tables                                       │
│     ORDER BY n_dead_tup DESC;                                      │
│                                                                    │
│  4. Long-running queries                                           │
│     SELECT pid, NOW() - xact_start AS duration, query              │
│     FROM pg_stat_activity                                          │
│     WHERE state != 'idle'                                          │
│     ORDER BY duration DESC;                                        │
│                                                                    │
│  5. Replication lag                                                │
│     SELECT client_addr, replay_lag                                 │
│     FROM pg_stat_replication;                                      │
│                                                                    │
│  6. Connection count                                               │
│     SELECT count(*) FROM pg_stat_activity;                         │
│                                                                    │
│  7. Locks / blocking                                               │
│     См. раздел 9.                                                  │
│                                                                    │
│  8. Autovacuum status                                              │
│     SELECT relname, last_autovacuum, last_autoanalyze,             │
│            autovacuum_count                                        │
│     FROM pg_stat_user_tables;                                      │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  МОНИТОРИНГ STACK                                                  │
│                                                                    │
│  Prometheus + Grafana:                                             │
│    - postgres_exporter → метрики PG                                │
│    - Grafana dashboard "PostgreSQL"                                │
│                                                                    │
│  Alerts на:                                                        │
│    - Cache hit ratio < 90%                                         │
│    - Replication lag > 10s                                         │
│    - Connections > 80% max_connections                             │
│    - Long transactions > 5 min                                     │
│    - Disk usage > 80%                                              │
│    - Locks waiting > 1 min                                         │
│                                                                    │
│  Logs:                                                             │
│    log_min_duration_statement = 1000ms                             │
│      → медленные queries в лог                                     │
│    Collect через Loki / ELK.                                       │
│                                                                    │
│  Traces:                                                           │
│    OpenTelemetry в приложении → JDBC spans → Jaeger/Tempo          │
└────────────────────────────────────────────────────────────────────┘
```

---

## 14. ФАЙЛОВАЯ СИСТЕМА PostgreSQL — что где лежит

```
┌────────────────────────────────────────────────────────────────────┐
│  /var/lib/postgresql/data/  (PGDATA)                               │
│                                                                    │
│  ├── postgresql.conf         ← конфиг сервера                      │
│  ├── postgresql.auto.conf    ← ALTER SYSTEM changes                │
│  ├── pg_hba.conf             ← client authentication rules         │
│  ├── pg_ident.conf           ← user name mapping                   │
│  ├── PG_VERSION              ← "16" (major version)                │
│  │                                                                 │
│  ├── base/                   ← databases                           │
│  │   ├── 1/                  ← template1 (OID 1)                   │
│  │   ├── 13396/              ← postgres db                         │
│  │   └── 16384/              ← ваша БД (OID пример)                │
│  │       ├── 16385           ← файл таблицы (heap fork)            │
│  │       ├── 16385.1         ← продолжение (файлы до 1 GB)         │
│  │       ├── 16385_fsm       ← Free Space Map                      │
│  │       ├── 16385_vm        ← Visibility Map                      │
│  │       ├── 16386           ← индекс                              │
│  │       └── ...                                                   │
│  │                                                                 │
│  ├── global/                 ← cluster-wide tables                 │
│  │   ├── pg_control          ← control file (LSN, состояние)       │
│  │   ├── pg_database         ← список БД                           │
│  │   ├── pg_authid           ← roles                               │
│  │   └── ...                                                       │
│  │                                                                 │
│  ├── pg_wal/                 ← WAL segments (16 MB каждый)         │
│  │   ├── 000000010000000000000001                                  │
│  │   ├── 000000010000000000000002                                  │
│  │   └── archive_status/                                           │
│  │                                                                 │
│  ├── pg_xact/                ← transaction status (CLOG)           │
│  ├── pg_multixact/                                                 │
│  ├── pg_subtrans/                                                  │
│  ├── pg_snapshots/                                                 │
│  ├── pg_tblspc/              ← tablespaces links                   │
│  ├── pg_stat/                                                      │
│  ├── pg_stat_tmp/            ← stats collector data                │
│  ├── pg_replslot/            ← replication slots                   │
│  ├── pg_logical/                                                   │
│  ├── pg_notify/                                                    │
│  ├── pg_serial/                                                    │
│  ├── pg_commit_ts/                                                 │
│  ├── pg_dynshmem/                                                  │
│  └── pg_twophase/                                                  │
└────────────────────────────────────────────────────────────────────┘

Размер типичной production БД:
  base/           — 100 GB - 10 TB (зависит от размера данных)
  pg_wal/         — 1-10 GB (max_wal_size, обычно 1-2 GB)
  pg_xact/        — 100 KB - 100 MB
  Остальные       — мегабайты
```

---

## 15. ДЕПЛОЙ POSTGRESQL В K8s

```
┌────────────────────────────────────────────────────────────────────┐
│  ВАРИАНТЫ                                                          │
│                                                                    │
│  1. Managed database (RDS, Cloud SQL, Aiven)                       │
│     Плюсы: backup/HA/upgrades сделаны за тебя                      │
│     Минусы: vendor lock-in, стоимость                              │
│                                                                    │
│  2. StatefulSet в K8s                                              │
│     Свой контроль, но нужно уметь managing                         │
│                                                                    │
│  3. Postgres Operator (Zalando, CloudNativePG, Crunchy)            │
│     StatefulSet + автоматика (failover, backup)                    │
│                                                                    │
│  4. External VM / bare metal                                       │
│     Приложение в K8s → БД на VM. Классика enterprise.              │
│     Может быть в том же дата-центре, но вне K8s.                   │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  StatefulSet в K8s                                                 │
│                                                                    │
│  apiVersion: apps/v1                                               │
│  kind: StatefulSet                                                 │
│  metadata: {name: postgres}                                        │
│  spec:                                                             │
│    serviceName: postgres-headless                                  │
│    replicas: 1                                                     │
│    template:                                                       │
│      spec:                                                         │
│        containers:                                                 │
│        - name: postgres                                            │
│          image: postgres:16                                        │
│          env:                                                      │
│          - name: POSTGRES_PASSWORD                                 │
│            valueFrom: {secretKeyRef: ...}                          │
│          volumeMounts:                                             │
│          - name: pgdata                                            │
│            mountPath: /var/lib/postgresql/data                     │
│          resources:                                                │
│            requests: {memory: 4Gi, cpu: 2}                         │
│            limits:   {memory: 8Gi, cpu: 4}                         │
│    volumeClaimTemplates:                                           │
│    - metadata: {name: pgdata}                                      │
│      spec:                                                         │
│        accessModes: [ReadWriteOnce]                                │
│        storageClassName: fast-ssd                                  │
│        resources: {requests: {storage: 100Gi}}                     │
│                                                                    │
│  Плюсы:                                                            │
│    - Stable pod names (postgres-0, postgres-1)                     │
│    - Persistent storage через PVC                                  │
│    - Ordered startup/shutdown                                      │
│                                                                    │
│  Минусы:                                                           │
│    - Ручной failover                                               │
│    - Backup нужно настроить самому                                 │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  POSTGRES OPERATOR (CloudNativePG)                                 │
│                                                                    │
│  apiVersion: postgresql.cnpg.io/v1                                 │
│  kind: Cluster                                                     │
│  metadata: {name: knp-db}                                          │
│  spec:                                                             │
│    instances: 3                    # primary + 2 standby           │
│    postgresql:                                                     │
│      parameters:                                                   │
│        max_connections: "200"                                      │
│        shared_buffers: "4GB"                                       │
│    bootstrap:                                                      │
│      initdb:                                                       │
│        database: knp                                               │
│        owner: knp                                                  │
│    storage:                                                        │
│      size: 100Gi                                                   │
│      storageClass: fast-ssd                                        │
│    backup:                                                         │
│      barmanObjectStore:                                            │
│        destinationPath: "s3://backups/knp"                         │
│      retentionPolicy: "30d"                                        │
│                                                                    │
│  Оператор автоматически:                                           │
│    - Создаёт primary + standbys                                    │
│    - Настраивает streaming replication                             │
│    - Failover при падении primary                                  │
│    - Backup в S3                                                   │
│    - PITR                                                          │
│    - Rolling upgrade PG versions                                   │
│                                                                    │
│  Endpoints:                                                        │
│    knp-db-rw  → всегда primary                                     │
│    knp-db-ro  → любой standby (read-only queries)                  │
│    knp-db-r   → любой (round-robin)                                │
└────────────────────────────────────────────────────────────────────┘
```

---

## 16. ПОЛНАЯ ЦЕПОЧКА — от Java до диска

Собираем всё вместе. Что происходит от `userRepository.findById(1L)` до чтения с NVMe SSD.

```
Time    Layer                    Что делает                     Latency
─────   ──────────────           ──────────────────             ──────
0 μs    UserService              вызывает repo.findById(1L)     0
1 μs    Spring @Transactional    proxy interceptor              1
2 μs    Hibernate                проверяет L1 cache (miss)      1
5 μs    Hibernate                генерирует SQL                 3
10 μs   HikariCP                 getConnection (idle→busy)      5
15 μs   pgjdbc driver            формирует wire protocol msg    5
                                 (Parse, Bind, Execute, Sync)
20 μs   TCP socket write         send через Unix / TCP          5-100

------ Now on PostgreSQL side ------

100 μs  Backend process          читает message из socket       80
200 μs  Parser                   parse SQL → AST                100
300 μs  Analyzer                 разрешение имён, permissions   100
500 μs  Planner                  cost estimation, план выбран   200
                                 (Index Scan on users_pkey)
550 μs  Executor начинает
600 μs  Buffer manager           ищет index root в buffer_pool
                                   HIT → 50 μs
                                   MISS → OS page cache HIT → 100 μs
                                   MISS → disk read → 100 μs (NVMe)
700 μs  B-tree traversal         спуск от root к leaf (3-4 уровня)
                                 каждый уровень — buffer_pool lookup
1 ms    Найдена запись [42→ctid(0,17)]
1.1 ms  Buffer manager           ищет heap block 0 таблицы users
                                   обычно cache hit
1.2 ms  Извлекается tuple по offset 17
1.3 ms  MVCC check               xmin/xmax vs snapshot
1.4 ms  Backend                  формирует RowDescription+DataRow msgs
1.5 ms  TCP socket write         send back to client

------ Back to Java ------

1.6 ms  pgjdbc driver            парсит response, создаёт ResultSet
1.7 ms  Hibernate                маппит row → User entity
                                 кэширует в L1 (Persistence Context)
1.8 ms  Spring @Transactional    commit (для read-only — noop)
1.9 ms  HikariCP                 return connection to pool
2.0 ms  UserService              возвращает User в контроллер

Total: 2 ms (в happy path со всем в кэшах).

Если cache miss на heap page + disk read: +100-200 μs.
Если connection pool exhausted: +30-second wait.
Если lock contention: +секунды-минуты.
Если replication lag на read replica: несколько ms задержки.
```

---

## 17. ГДЕ ЧТО ДИАГНОСТИРОВАТЬ ПРИ ИНЦИДЕНТЕ

```
Симптом                              →   Где смотреть
─────────────────────────────────────────────────────────────────
Приложение hang в getConnection()    →   HikariCP metrics
                                          - Pool utilization %
                                          - waitCount / activeCount
                                     →   pg_stat_activity
                                          Долгие транзакции?

Slow query                           →   pg_stat_statements
                                          Топ по total_exec_time
                                     →   EXPLAIN (ANALYZE, BUFFERS)
                                          Actual vs estimated rows

БД disk usage растёт                 →   pg_stat_user_tables.n_dead_tup
                                          Bloat из-за плохого VACUUM
                                     →   pg_wal/ размер
                                          WAL накапливается? Проверить
                                          replication slots, archiving

Backend процессов слишком много      →   pg_stat_activity count
                                     →   HikariCP pool size × N Pods
                                          / max_connections

Deadlock                             →   PG log (log_lock_waits=on)
                                     →   pg_stat_activity + pg_locks

Replication lag                      →   pg_stat_replication на primary
                                          replay_lag, flush_lag

Cache hit ratio низкий               →   pg_statio_user_tables
                                     →   Увеличить shared_buffers
                                          (или упростить queries)

Autovacuum не работает               →   pg_stat_user_tables
                                          last_autovacuum
                                     →   log_autovacuum_min_duration
                                          логи в PG log

Data corruption                      →   pg_amcheck / pg_verify_checksums
                                     →   Проверить дисковое здоровье
                                          (SMART)

БД не стартует                       →   PG log (postgresql.log)
                                     →   pg_control corruption?
                                     →   Recovery в single-user mode
```

---

## Итог

PostgreSQL — это ~7 слоёв абстракции между `userRepository.findById()` и физическим NVMe SSD:

**Application** — Spring @Transactional, JPA/Hibernate, JDBC, HikariCP, pgjdbc.

**Wire protocol** — TCP на порт 5432, PostgreSQL protocol v3 (Parse/Bind/Execute/Sync).

**Backend process** — форкается postmaster'ом на каждый connection. Parse → Analyze → Plan → Execute.

**Buffer pool** — shared_buffers (~25% RAM), кэш страниц. Двойное кэширование с OS page cache.

**Physical storage** — 8 KB pages с header + item pointers + tuples. TOAST для больших values. Файлы по 1 GB в base/<db_oid>/.

**WAL** — write-ahead log в pg_wal/, 16 MB segments. Гарантирует durability. fsync на commit.

**MVCC** — каждый tuple с xmin/xmax. Читатели не блокируют писателей. Требует VACUUM для очистки dead tuples.

**Indexes** — B-tree (default), GIN (jsonb/tsvector), GiST (geometry), BRIN (time-series).

**Background workers** — checkpointer, bgwriter, walwriter, autovacuum, WAL sender.

**Connection pooling** — HikariCP в приложении + опционально PgBouncer для мультиплексирования.

**Replication** — streaming (physical WAL) или logical (row-level events). Failover через Patroni / operator.

**Backup** — pg_basebackup + WAL archiving = PITR. pgBackRest / WAL-G в проде.

**Мониторинг** — pg_stat_statements, pg_stat_activity, pg_stat_replication + postgres_exporter → Prometheus.

**Deployment в K8s** — StatefulSet или Postgres Operator (CloudNativePG). Persistent volumes через CSI. Endpoints для rw / ro.

Полный запрос от Java до диска — ~2 ms в happy path (всё в кэше). При cache miss + disk read — 5-10 ms. При lock contention — секунды. При bad plan (seq scan вместо index) — минуты.

Практика: возьми свою БД в проде, посмотри `pg_stat_statements` топ-10 медленных queries. Каждый прогони через `EXPLAIN (ANALYZE, BUFFERS)`. Сравни estimated vs actual rows. Проверь cache hit ratio через `pg_statio_user_tables`. Посмотри `pg_stat_activity` — есть ли idle in transaction? Разберись с одной проблемой — и общая картина устройства PostgreSQL встанет на место.
