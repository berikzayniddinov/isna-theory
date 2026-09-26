# 28. PostgreSQL: устройство, MVCC, WAL, индексы

## Процесс-модель PostgreSQL

PostgreSQL использует многопроцессную архитектуру принципиально отличающуюся от многопоточных СУБД вроде MySQL. Каждое клиентское соединение обрабатывается отдельным OS процессом (не потоком). Такая архитектура даёт хорошую изоляцию — сбой в одном backend не влияет на другие, но требует внимания при работе с большим количеством соединений.

```
                    ┌─────────────────────┐
                    │    postmaster       │  ← главный процесс, порт 5432
                    │  слушает accept()   │
                    └──────┬──────────────┘
                           │  fork() при новом connection
              ┌────────────┼────────────┬────────────┐
              ▼            ▼            ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
        │backend 1│  │backend 2│  │backend 3│  │backend N│  ← клиентские процессы
        │ (client)│  │ (client)│  │ (client)│  │ (client)│
        └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
             │            │            │            │
             └────────────┴────────────┴────────────┘
                          │  через shared memory (IPC)
                          ▼
              ┌───────────────────────────┐
              │      Shared Memory        │
              │  ┌──────────────────────┐ │
              │  │   shared_buffers     │ │  ← кэш страниц данных
              │  ├──────────────────────┤ │
              │  │   WAL buffers        │ │  ← буфер WAL до fsync
              │  ├──────────────────────┤ │
              │  │   locks table        │ │  ← блокировки строк/таблиц
              │  └──────────────────────┘ │
              └────────────┬──────────────┘
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

Postmaster это главный процесс слушающий port 5432 и accepting новые соединения. При приходе нового connection postmaster делает fork создавая отдельный backend процесс который обрабатывает всё дальнейшее взаимодействие с этим клиентом.

Backend процесс живёт весь срок соединения. Занимает 5-10 MB памяти на процесс минимум. Держит TCP connection с клиентом. Обрабатывает SQL запросы. При закрытии соединения процесс завершается и его ресурсы освобождаются.

Из процессной модели следует важное последствие для приложений — много одновременных соединений это много процессов и много памяти. 1000 open connections потребует 5-10 GB памяти только на процессы, плюс каждый из них создаёт нагрузку на CPU при переключении контекста. Отсюда стандартная схема с двухуровневым pooling — HikariCP внутри JVM плюс PgBouncer перед PostgreSQL. Приложение работает с JDBC pool внутри JVM. PgBouncer мультиплексирует соединения от многих приложений на small pool real PostgreSQL connections.

Shared buffers это ключевая структура shared memory — cache страниц данных используемый всеми backend процессами. При запросе PG сначала ищет нужную страницу в shared_buffers. Если найдена — cache hit, никакого disk I/O. Не найдена — читает с диска, помещает в shared_buffers для будущего использования, отдаёт клиенту. Правильно настроенный shared_buffers обычно устанавливается в 25 процентов available RAM. Default 128 MB слишком мало для production.

Background процессы выполняют maintenance функции параллельно основной работе backends. bgwriter периодически сбрасывает dirty pages из shared_buffers на диск чтобы освобождать место для новых страниц. checkpointer выполняет checkpoints — полный сброс всех dirty pages на диск, точка от которой можно начать recovery при crash. walwriter пишет WAL buffer на диск не блокируя transaction commit. autovacuum это фоновая уборка мёртвых версий tuples создаваемых MVCC. archiver копирует completed WAL segments для point-in-time recovery если настроено. logical replication процессы отправляют changes на logical replicas если настроено.

## MVCC Multi-Version Concurrency Control

MVCC это фундаментальная архитектурная особенность PostgreSQL определяющая как обрабатываются concurrent transactions. Понимание MVCC критически важно для правильной работы с базой.

Проблема которую решает MVCC — classical database используют locks для управления concurrent access. Читатель берёт shared lock, писатель берёт exclusive lock. Читатель блокирует писателя, писатель блокирует читателя. Это создаёт serialization concurrent operations что ограничивает throughput особенно для read-heavy workloads. Deadlocks становятся частыми при complex access patterns.

Идея MVCC радикально другая. Каждое изменение создаёт новую версию строки а не изменяет старую. Транзакция видит consistent snapshot данных на момент своего начала независимо от concurrent modifications. Читатели не блокируют писателей потому что читатели работают со snapshot. Писатели не блокируют читателей по той же причине. Только множественные писатели одной строки могут блокировать друг друга.

Реализация в PostgreSQL основана на скрытых системных колонках каждой tuple. xmin содержит transaction ID транзакции создавшей эту версию — когда tuple стал видимым. xmax содержит transaction ID транзакции удалившей или обновившей эту версию — когда tuple перестал быть актуальным. ctid это физический адрес в таблице для быстрого доступа.

Пример UPDATE операции. Оригинальная строка имеет xmin равный transaction создавшей её, xmax равный zero (не удалена). Транзакция T5 делает UPDATE. Старая версия помечается xmax равным T5 (удалена в T5). Создаётся новая версия с xmin равным T5. Обе версии остаются в таблице. Транзакции с snapshot времени когда T5 ещё не committed видят старую версию. Транзакции с snapshot после commit T5 видят новую версию.

Snapshot определяет видимость версий для конкретной транзакции. При начале транзакции создаётся snapshot — список transaction ID которые committed на момент начала. Каждая tuple version проверяется — если xmin меньше snapshot xmin и в snapshot list committed transactions, а xmax либо zero либо не committed на момент snapshot — версия видима. Иначе — не видима.

Уровни изоляции определяют когда snapshot обновляется. READ_COMMITTED (default) обновляет snapshot на каждый statement — каждый SELECT видит последние committed данные. REPEATABLE_READ использует один snapshot на всю транзакцию — все statements видят те же данные независимо от concurrent commits. SERIALIZABLE добавляет predicate locks для полной serialization — обнаруживает случаи когда результат зависит от порядка concurrent transactions и откатывает одну из них.

## VACUUM и bloat

MVCC создаёт побочный эффект — мёртвые версии tuples накапливаются в таблице. Когда UPDATE или DELETE — старая версия не удаляется сразу, только помечается как невидимая через xmax. Со временем накопление мёртвых версий приводит к разбуханию таблицы что называется bloat.

Bloat имеет несколько негативных последствий. Таблица занимает больше disk space чем нужно для актуальных данных. Sequential scans читают больше страниц включая dead tuples замедляя queries. Индексы также bloating потому что старые версии tuples остаются проиндексированными. Buffer cache менее эффективен потому что хранит мёртвые данные вместе с живыми.

VACUUM это фоновый процесс cleanup мёртвых версий. Ordinary VACUUM освобождает space внутри страниц для повторного использования новыми tuples но не возвращает space операционной системе. VACUUM FULL полностью перестраивает таблицу возвращая space OS, но требует exclusive lock на таблицу что делает недоступной для queries. VACUUM ANALYZE комбинирует cleanup с обновлением статистики для query planner.

Autovacuum это automatic background process запускающий VACUUM когда количество dead tuples превышает threshold. Настройка через параметры autovacuum_vacuum_threshold и autovacuum_vacuum_scale_factor. По default запускается когда dead tuples превышают 20 процентов от live tuples плюс 50 rows constant. Настраивается per-table для критических таблиц где нужны tighter параметры.

Проблемы связанные с VACUUM. Долгие транзакции блокируют VACUUM — если transaction началась час назад, VACUUM не может очистить версии которые могут быть нужны этой транзакции (по её snapshot). Длинные analytical queries могут привести к массивному bloat в write-heavy tables. Autovacuum может не поспевать за rate updates — необходимо тюнинг параметров или manual VACUUM.

Мониторинг через pg_stat_all_tables — колонки n_dead_tup для мёртвых tuples, n_live_tup для живых, last_vacuum для последнего запуска. Alert на растущий n_dead_tup или устаревший last_vacuum индикатор проблем с cleanup.

## WAL Write-Ahead Log

WAL это механизм durability в PostgreSQL. Прежде чем изменение попадёт в actual data files оно записывается в log. При crash log позволяет восстановить незакоммиченные изменения читая из последнего checkpoint.

Зачем нужен WAL. Durability гарантия для ACID — committed transactions не теряются даже при crash. Быстрая запись — WAL это append-only file, sequential write гораздо быстрее random write в data files. Replication — реплики читают WAL master и применяют те же изменения. PITR (Point-in-Time Recovery) — можно восстановить database на любой момент в прошлом replay WAL с backup point.

Механизм работы. Transaction выполняет UPDATE foo SET x=1 WHERE id=5. PostgreSQL создаёт WAL запись описывающую это изменение — какая tuple, старые и новые значения. Изменение применяется в shared_buffers — теперь страница помечена dirty. При COMMIT — WAL buffer flushes на диск через fsync. Только после успешного fsync commit считается completed и client получает подтверждение. Данные в actual data files могут ещё не быть — они на диске в shared_buffers плюс на диске в WAL. Периодически checkpoint сбрасывает все dirty pages из shared_buffers в actual data files.

Checkpoint это ключевая операция баланса между latency и recovery time. При checkpoint все dirty pages в shared_buffers записываются на диск. После этого WAL до этой точки становится ненужным для recovery — данные уже в actual files. Настройка через checkpoint_timeout (интервал между checkpoints) и max_wal_size (максимальный размер WAL до forced checkpoint).

Приложения с интенсивными writes создают frequent checkpoints что может замедлять систему из-за disk I/O bursts. Настройка checkpoint_completion_target позволяет распределить write load во времени вместо burst — например 0.9 означает checkpoint должен завершиться за 90 процентов от checkpoint_timeout, spreading writes.

fsync параметр критически важен для durability. При fsync on (default) commit блокируется до успешного flush WAL на диск. Медленнее но гарантирует durability. При fsync off — commit возвращается сразу, WAL flushes eventually. Быстро но при crash можно потерять secondscount committed transactions. Использовать только для тестовых или non-critical системах где данные легко восстановить из другого источника.

## Индексы

PostgreSQL поддерживает несколько типов индексов для разных use cases. Выбор правильного типа критически влияет на производительность.

B-Tree это default тип индекса подходящий для большинства сценариев. Основан на balanced tree структуре. Хорошо для equality queries (WHERE x = 5), range queries (WHERE x BETWEEN 10 AND 100), sorting (ORDER BY x), prefix matching (WHERE x LIKE 'abc%' без leading wildcard). Плохо для полнотекстового поиска, functional queries без functional index, non-equality operators like different.

Hash index предназначен только для equality checks. Быстрее B-Tree для точного совпадения. Не поддерживает range queries или sorting. До PostgreSQL 10 не был crash-safe что ограничивало использование. Сейчас crash-safe но всё ещё редко используется потому что B-Tree универсальнее с сопоставимой производительностью для equality.

GiST (Generalized Search Tree) это универсальная индексная структура для complex types. Используется для геометрических данных через PostGIS, full-text search, range types (tsrange для timestamp ranges). Balanced tree но с настраиваемым способом сравнения.

GIN (Generalized Inverted Index) это инвертированный индекс. Особенно эффективен для columns содержащих множественные значения. Full-text search через tsvector — GIN indexes text tokens позволяя fast lookup. Arrays — WHERE array_column contains ARRAY[1,2] efficient через GIN. JSONB — GIN indexes keys и values позволяя fast queries по nested структурам. Медленнее B-Tree на insert потому что каждый token/value inserted отдельно, но search быстрее.

BRIN (Block Range Index) это compact index для больших таблиц с naturally ordered данными. Индекс хранит min/max values для each block range обычно 128 pages. При query PG проверяет ranges — если query condition вне min/max блока, skip блок. Крошечный размер (мегабайты для терабайтной таблицы) но менее precise чем B-Tree. Хорош для timestamps в append-only tables, log data.

Composite index на несколько колонок:
```sql
CREATE INDEX ON fno(status, created_at);
```

Порядок колонок критически важен. Индекс работает для queries использующих leftmost prefix — WHERE status = 'NEW' (использует индекс), WHERE status = 'NEW' AND created_at > '2025-01-01' (использует индекс полностью), WHERE created_at > '2025-01-01' (НЕ использует индекс потому что status не первая колонка запроса).

Partial index покрывает только subset строк по WHERE clause:
```sql
CREATE INDEX ON fno(created_at) WHERE status = 'NEW';
```

Полезен когда queries обычно filter по specific condition. Меньший размер индекса и быстрее lookup потому что содержит только relevant данные. Использование для «горячих» partition — active orders, unprocessed messages.

Unique index гарантирует уникальность и одновременно быстрый поиск:
```sql
CREATE UNIQUE INDEX ON fno(reg_num);
```

Automatic invocation при вставке — DB проверяет отсутствие дубликата используя индекс. Возвращает constraint violation при попытке дубля.

Functional index на выражение columns:
```sql
CREATE INDEX ON users(lower(email));
```

Работает для queries использующих то же выражение — WHERE lower(email) = 'x@y.com'. Полезно для case-insensitive поисков.

## Планировщик и EXPLAIN

Query execution в PostgreSQL проходит четыре стадии. Parse превращает SQL текст в AST. Rewrite применяет правила — expanding views, применение rewrite rules. Plan выбирает оптимальный план execution из возможных вариантов. Execute выполняет выбранный план.

Планировщик генерирует множество possible plans и оценивает их cost. Cost основан на статистике таблицы — количество строк, distribution values в колонках, correlation with physical order. Статистика собирается через ANALYZE и хранится в pg_statistic. Устаревшая статистика приводит к плохим планам — планировщик оценивает queries неправильно, выбирает subptimal стратегию.

EXPLAIN показывает выбранный план без исполнения:
```sql
EXPLAIN SELECT * FROM fno WHERE status = 'NEW';

Seq Scan on fno  (cost=0.00..1000.00 rows=100 width=200)
  Filter: (status = 'NEW')
```

Cost это оценка планировщика в abstract units (не миллисекунды). Rows — estimate количества строк на выходе. Width — average byte size per row. Оценки — не реальные значения, based on статистике.

EXPLAIN ANALYZE actually executes query с замерами:
```sql
EXPLAIN ANALYZE SELECT * FROM fno WHERE status = 'NEW';

Index Scan using idx_fno_status on fno  
    (cost=... rows=100) (actual time=0.5..2.3 rows=95 loops=1)
Planning Time: 0.2 ms
Execution Time: 2.5 ms
```

Actual time и rows дают реальные метрики. Планирование time — сколько заняло построение плана. Execution time — actual execution. Полезное сравнение — estimated vs actual rows. Большое расхождение указывает на устаревшую статистику.

Основные типы операций видимые в plan. Seq Scan это чтение всей таблицы sequentially. Плохо для больших таблиц с selective queries — нужен index. Хорошо для queries требующих значительную часть таблицы (более 10-20 процентов) — index scan создаст random I/O overhead. Index Scan использует индекс для нахождения матчащих rows и берёт актуальные значения из heap. Index Only Scan использует только индекс — все требуемые колонки в индексе, не нужно обращаться к heap. Bitmap Index Scan комбинирован с Bitmap Heap Scan — сначала строится bitmap строк матчащих index conditions, потом heap читается в физическом порядке.

## JOIN алгоритмы

PostgreSQL поддерживает три основных JOIN алгоритма. Планировщик выбирает на основе стоимости и характеристик data.

Nested Loop это простейший алгоритм — для каждой row внешней таблицы поиск в внутренней:
```
for row1 in table1:
    for row2 in table2:
        if row1.key == row2.key:
            output(row1, row2)
```

Хорош когда внутренняя таблица маленькая или есть индекс на join key внутренней таблицы. Плох для больших таблиц без индекса — quadratic complexity делает его непрактичным.

Hash Join строит hash table по join key одной таблицы, потом проходит другую:
```
hash_table = build hash on table1.key
for row2 in table2:
    if row2.key in hash_table:
        output(...)
```

Хорош для equi-joins больших таблиц. Требует память для hash table но обеспечивает O(N+M) complexity. Работает только для equality joins.

Merge Join требует чтобы обе таблицы были sorted по join key:
```
i=0, j=0
while i < len(t1) and j < len(t2):
    if t1[i].key == t2[j].key: output; move both
    elif t1[i].key < t2[j].key: i++
    else: j++
```

Хорош когда данные уже sorted через индекс или предыдущий ORDER BY. Efficient для больших таблиц если sorting не нужен дополнительно.

Планировщик выбирает алгоритм на основе оценок стоимости для каждого варианта. Обычно правильный выбор автоматический — но иногда через hints или переписывание query можно повлиять на планировщик если очевидно неправильный выбор.

## Транзакции и блокировки

Базовая транзакция:
```sql
BEGIN;
UPDATE fno SET status='X' WHERE id=1;
COMMIT;
-- или ROLLBACK для отмены
```

MVCC даёт concurrent readers без блокировок но writers на одной строке блокируются. PostgreSQL автоматически берёт row-level locks при UPDATE/DELETE. Explicit locks через SELECT FOR UPDATE берут exclusive lock на матчащие rows не изменяя их — полезно для «read modify write» patterns:
```sql
SELECT * FROM fno WHERE id=1 FOR UPDATE;
-- работа с данными
UPDATE fno SET status='X' WHERE id=1;
COMMIT;
```

Другая транзакция ждёт пока текущая completes. Опции NOWAIT (сразу error если lock unavailable) и SKIP LOCKED (пропустить locked rows) полезны для job queues где несколько workers обрабатывают одну таблицу.

Advisory locks позволяют application-level distributed locking через database:
```sql
SELECT pg_advisory_lock(12345);
-- exclusive section
SELECT pg_advisory_unlock(12345);
```

Использование для scheduled jobs которые должны работать singleton в кластере (аналог ShedLock). Обеспечивают distributed coordination без external системы.

Deadlock происходит когда две транзакции ждут друг друга. Классический пример:
```
T1: UPDATE row WHERE id=1;    (lock 1)
T2: UPDATE row WHERE id=2;    (lock 2)
T1: UPDATE row WHERE id=2;    (ждёт T2)
T2: UPDATE row WHERE id=1;    (ждёт T1)
```

PostgreSQL обнаруживает deadlocks через периодические checks — если transaction blocked более deadlock_timeout (default 1 second), checker строит wait-for graph. При detection одна transaction получает deadlock_detected error, forced к ROLLBACK. Application должен retry операцию.

Профилактика deadlocks. Всегда брать locks в consistent порядке (например по id ascending) — если все следуют same order, cycle невозможен. Держать transactions короткими — меньше time для conflict. Избегать user interaction внутри transaction — время работы predictable.

## Statistic tables для мониторинга

PostgreSQL предоставляет множество системных views для мониторинга:
```sql
-- активные connections и их состояние
SELECT pid, state, query FROM pg_stat_activity;

-- статистика per таблица
SELECT relname, n_dead_tup, n_live_tup, last_vacuum, last_analyze
FROM pg_stat_user_tables;

-- медленные queries (требует pg_stat_statements extension)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 20;

-- блокировки
SELECT * FROM pg_locks WHERE NOT granted;

-- размеры
SELECT pg_size_pretty(pg_total_relation_size('fno'));
SELECT pg_size_pretty(pg_relation_size('idx_fno_status'));

-- использование индексов
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes;
```

pg_stat_statements это extension крайне важный для production. Собирает статистику каждого выполненного query — count executions, total и mean execution time, rows returned. Позволяет identifying самые problematic queries через query performance analysis. Включение через:
```
shared_preload_libraries = 'pg_stat_statements'
```

Обязательный элемент production PostgreSQL setup. Без него нет visibility в query performance patterns.

## Репликация

Streaming replication (physical) это стандартный механизм HA в PostgreSQL. WAL пишется на master и streams на реплики в реальном времени. Реплики применяют те же changes к своим копиям database.

```
[master] ──WAL stream──► [replica 1]
                    ╲───► [replica 2]
                    ╲───► [replica 3]
```

Synchronous replication требует что replica подтвердила receipt WAL перед commit successful на master. Медленнее (extra round-trip) но zero data loss при master failure — все committed changes точно на replica. Asynchronous replication commits возвращаются сразу без ожидания replica. Быстрее но при master failure можно потерять недавние transactions ещё не отправленные на replica.

Logical replication работает через change data capture. Публикуются decoded logical changes (INSERT/UPDATE/DELETE операции) вместо physical WAL. Гибче — можно реплицировать specific tables, между разными versions PostgreSQL, применять transformations. Медленнее physical но более flexible.

Read replicas часто используются для scaling read workload. Приложение делает writes к master, reads к replicas. Балансировщик распределяет read queries между replicas.
```yaml
spring:
  datasource:
    write:
      url: jdbc:postgresql://master:5432/knp
    read:
      url: jdbc:postgresql://replica:5432/knp
```

Caveat replication lag. Async replicas отстают от master на некоторое время (обычно миллисекунды или секунды). Read сразу после write может показать old data (read-your-writes проблема). Для критичных read-after-write сценариев необходимо читать с master или использовать synchronous replication.

## Партиционирование

Для очень больших таблиц (десятки миллионов и больше строк) партиционирование даёт значительные benefits. Таблица разбивается на partitions по ключу, каждый partition physically separate table но logically объединены.

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

Плюсы включают быстрый DELETE старых данных через DROP partition (мгновенно vs DELETE каждой row). Partition pruning в queries — WHERE created_at > '2026-01-01' автоматически skips partition fno_2025. Меньшие индексы per partition быстрее bulk operations. VACUUM работает on partition level что ограничивает impact на другие partitions.

Минусы. Более сложные миграции — schema changes применяются к каждому partition. UNIQUE constraints через все partitions сложны — обычно возможны только если partition key часть unique constraint. Foreign keys к partitioned table имеют ограничения.

Стратегии партиционирования. Range для timestamps (по датам). List для discrete values (по статусу, регионам). Hash для равномерного распределения без natural ordering. Выбор зависит от access patterns и характеристик data.

## Типы данных правильные choices

bigserial или bigint для ID — 64-bit integer поддерживает вплоть до 9 квинтиллионов. Не надо экономить на int (32-bit) — легко превысить при growth.

text это стандартный тип для строк без предопределённой длины. varchar(N) практически идентичен text но с constraint на length. varchar(N) для строк с business reason иметь max length (email, phone), text для arbitrary text без length constraint.

timestamp против timestamptz. timestamp без timezone info хранит just точку во времени как naive datetime. timestamptz включает timezone information и автоматически конвертирует между timezones. Всегда timestamptz в production — избежание timezone bugs.

jsonb для JSON данных — binary формат позволяющий indexing через GIN. Значительно быстрее json (текстовый формат) при querying. json оставляют только когда нужна exact preservation входного формата включая whitespace и key ordering что редко практически.

uuid для universally unique identifiers — 128-bit random values. Полезно для distributed систем где невозможно централизованное generation IDs. Trade-off с bigserial — random UUIDs имеют плохую locality что снижает cache efficiency, bigserial более cache-friendly но требует централизованного generation.

numeric(P, S) для точных десятичных значений — precision и scale. Обязательно для money, финансовых расчётов где floating-point rounding errors неприемлемы. Slower than double precision но precise.

bytea для binary data — arbitrary bytes. Полезно для small binaries stored inline (например certificates, small images). Для larger binaries обычно external storage через file system или object store с references в database.

## Настройки под нагрузку

Ключевые параметры для production tuning:
```
shared_buffers = 4GB              # 25% RAM
effective_cache_size = 12GB       # 75% RAM (подсказка планировщику про OS cache)
work_mem = 32MB                    # для sorting в query, per операция
maintenance_work_mem = 1GB         # для VACUUM, CREATE INDEX
max_connections = 200              # ограничение параллелизма
wal_level = replica                # для replication
max_wal_size = 4GB                 # между checkpoints
checkpoint_timeout = 15min         # частота checkpoints
random_page_cost = 1.1             # для SSD (default 4.0 для HDD)
```

shared_buffers 25 процентов RAM обычно рекомендуется. Меньше — недостаточно cache. Больше — конкурирует с OS page cache без большого benefit.

work_mem умножается на каждую sorting/hashing операцию в query. Слишком большое значение может привести к OOM когда много параллельных queries. Balance — обычно 32-128 MB для типичных workloads.

random_page_cost для SSD должен быть уменьшен с default 4.0 до 1.1-2.0. На SSD random access не намного медленнее sequential. Правильная настройка помогает планировщику предпочитать index scans когда appropriate.

pgtune (https://pgtune.leopard.in.ua) даёт reasonable starting configuration на основе hardware характеристик. Отправная точка для дальнейшего tuning.

## Итоги

PostgreSQL использует многопроцессную архитектуру. Каждое соединение отдельный OS процесс. Много соединений много памяти — HikariCP плюс PgBouncer стандартная схема.

MVCC создаёт новую версию строки при каждом изменении. Читатели не блокируют писателей. xmin/xmax скрытые колонки определяют видимость версий для транзакций. Snapshot определяет что видит транзакция.

VACUUM обязателен потому что MVCC накапливает мёртвые версии. Autovacuum automatic но может требовать tuning. VACUUM FULL блокирует таблицу.

WAL обеспечивает durability. Write-ahead log записывается перед изменениями в data files. Checkpoint периодически сбрасывает dirty pages. Основа replication и PITR.

Индексы разных типов для разных use cases. B-Tree default. GIN для JSONB, arrays, full-text. BRIN для больших sorted таблиц. Composite index leftmost prefix rule.

EXPLAIN ANALYZE обязательный инструмент. Показывает actual execution с временами. Ищи Seq Scan на больших таблицах, огромные расхождения estimated vs actual rows (стale статистика).

JOIN algorithms — Nested Loop, Hash Join, Merge Join. Планировщик выбирает автоматически на основе стоимости.

Транзакции с row-level locks. Advisory locks для distributed coordination. Deadlocks detected и одна transaction откатывается. Профилактика через consistent lock ordering.

Мониторинг через системные views. pg_stat_statements обязательный extension для production. pg_stat_activity для current activity. pg_stat_user_tables для table health.

Replication physical (streaming) или logical (CDC). Read replicas для scaling. Replication lag caveat для read-after-write.

Партиционирование для очень больших таблиц. Range/List/Hash strategies. Fast DROP старых данных, partition pruning в queries.

Типы данных critically choose right. timestamptz always. jsonb over json. numeric(P,S) для money. UUID vs bigserial trade-offs.

Настройки под нагрузку. shared_buffers 25%, work_mem 32MB, random_page_cost 1.1 для SSD, pg_stat_statements включён.

Дальше — Spring Boot integration с PostgreSQL через HikariCP, connection pool, timeouts и tuning.
