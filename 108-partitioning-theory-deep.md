# 108. Партиции в БД: механика PostgreSQL глубоко

## Зачем это знать

Партиционирование как концепция понятное: разбить большую таблицу на подтаблицы. Практический старт — в файле 91 (типы range/list/hash, миграция обычной таблицы, базовые команды). Здесь — уровень глубже. Что реально делает planner когда решает какие партиции пропустить и почему иногда pruning не срабатывает. Почему в partitioned table нет «глобального» primary key. Что происходит с FK когда обе таблицы partitioned. Как autovacuum работает per partition и почему это одновременно преимущество и головная боль. Row movement при UPDATE меняющем partition key — как оно устроено и почему дорого. Ошибки, которые не видны в документации: default partition как ловушка, too many partitions как проблема планировщика, wrong тип partition key ломающий pruning.

Разберём: механику partition pruning в двух режимах (plan-time и execution-time), тонкости с prepared statements и параметризованными запросами. Ограничения индексов на partitioned tables — почему unique нужно с partition key, отсутствие «глобального» индекса, cross-partition uniqueness через ухищрения. Constraint exclusion (legacy inheritance-based партиционирование) vs declarative (10+) — почему знать оба важно. Foreign keys — что можно с Postgres 11, 12, 13, 15, 16. Attach/detach партиции — как атомарно подключать существующую таблицу, ловушки с блокировками. Row movement и почему partition key лучше не менять. Statistics и планирование per partition. Autovacuum multipliers и тюнинг. Locking семантика: что блокирует что. Default partition — почему её лучше не иметь. Практические паттерны миграции без даунтайма (dual-write, INSERT-SELECT в batch, ATTACH готовой партиции). Реальная диагностика в проде: как понять что pruning не сработал, где смотреть per-partition метрики, чем ловить hotspot.

## Механика partition pruning: plan-time vs execution-time

Partition pruning — процесс исключения партиций из выполнения запроса. Ключевая оптимизация, ради которой партиционирование обычно и делают. Работает в двух режимах, и это разделение критично понимать.

**Plan-time pruning** (constraint exclusion в терминах старой inheritance-модели, planner-level pruning для declarative) — работает **на этапе планирования запроса**. Planner видит `WHERE created_at >= '2024-05-01' AND created_at < '2024-06-01'` и bounds каждой партиции. Может доказать что нужна только `orders_2024_05` — генерирует план только по этой партиции. Остальные партиции даже не появляются в плане.

Пример вывода EXPLAIN:

```
Seq Scan on orders_2024_05 orders  (cost=0.00..1234.00 rows=50000 width=64)
   Filter: (created_at >= '2024-05-01' AND created_at < '2024-06-01')
```

Только одна партиция в плане. Если бы pruning не сработал, увидели бы:

```
Append  (cost=0.00..15000.00 rows=600000 width=64)
  ->  Seq Scan on orders_2024_01 orders
  ->  Seq Scan on orders_2024_02 orders_1
  ->  Seq Scan on orders_2024_03 orders_2
  ...
  ->  Seq Scan on orders_2024_12 orders_11
```

Все 12 партиций в плане — pruning провалился. Обычно из-за одной из трёх причин: partition key не в WHERE; тип не совпадает (partition key `date`, а сравниваем с `timestamp` — planner не может доказать boundedness без cast); функция вокруг partition key (`WHERE date_trunc('month', created_at) = ...` — не работает pruning).

**Execution-time pruning** (Postgres 11+) — работает **во время выполнения**, когда значение partition key становится известно только в runtime. Классический пример — параметризованный prepared statement:

```sql
PREPARE q AS SELECT * FROM orders WHERE created_at = $1;
EXECUTE q('2024-05-15');
```

На этапе planning значение `$1` неизвестно. Planner генерирует план с **всеми партициями** в Append, но каждая партиция обёрнута в runtime-фильтр. На execution PostgreSQL смотрит на значение параметра, применяет тот же алгоритм boundedness — пропускает партиции без реального сканирования.

EXPLAIN для такого запроса:

```
Append  (cost=... loops=1)
  Subplans Removed: 11    ← вот execution-time pruning
  ->  Index Scan on orders_2024_05_pkey
```

`Subplans Removed: 11` — 11 партиций были в плане, но выкинуты на runtime. Только одна реально сканировалась. Это работает для параметризованных запросов, subplans, PL/pgSQL функций.

**Ключевая ловушка** — pruning с nested loop join'ами. При join'е partitioned таблицы с другой, planner может не знать заранее какие партиции нужны — pruning не сработает. Обход: `SET enable_partitionwise_join = on` (default off в старых версиях, on в 12+), тогда planner попробует partition-wise join — если обе таблицы партиционированы по одному ключу, join делается per-partition отдельно.

## Индексы на partitioned tables: тонкости

Один из самых частых сюрпризов для новичков — **индекс на parent таблице не создаёт индекса на партициях автоматически до Postgres 11**. С 11-й версии `CREATE INDEX ON parent_table` создаёт индекс на parent и **автоматически на всех существующих и будущих партициях**. Это удобно, но за этим стоит ограничение.

Индексы на partitioned table в реальности — набор **локальных** индексов, по одному на каждой партиции. Никакого «глобального» индекса поверх всех данных нет. Каждая партиция имеет свой B-tree, свой heap, свои страницы.

Отсюда следуют важные последствия:

**Unique constraint должен включать partition key.** PostgreSQL не может гарантировать уникальность значения через все партиции без сканирования всех — это было бы дороже чем uniqueness обычной таблицы. Поэтому:

```sql
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    ...
) PARTITION BY RANGE (created_at);

-- Работает: partition key входит в unique
ALTER TABLE orders ADD CONSTRAINT orders_pk PRIMARY KEY (id, created_at);

-- НЕ работает:
ALTER TABLE orders ADD CONSTRAINT orders_pk PRIMARY KEY (id);
-- ERROR: unique constraint on partitioned table must include all partitioning columns
```

Это ломает привычку «PK = только id». Приходится либо делать composite PK (`id, created_at`), либо жить без глобальной уникальности id и полагаться на sequence + application-level контроль.

**Foreign key на partitioned table** появился в Postgres 11: partitioned table может быть **целью** FK (`REFERENCES orders(id, created_at)`). До 12-й версии — не могла быть **источником** (FK **from** partitioned to another). С Postgres 12 работает в обе стороны, но требует те же условия — FK-колонки должны включать partition key целевой таблицы, если она partitioned.

**Concurrent index creation** на partitioned table (`CREATE INDEX CONCURRENTLY`) — до Postgres 11 не работал вовсе, с 11 работает на parent (создаёт concurrently на каждой партиции последовательно). Долго — если 100 партиций по 5 минут каждая, час на всю операцию. Обход: создавать индексы на каждой партиции параллельно вручную (`CREATE INDEX CONCURRENTLY ON orders_2024_01`, ..., в разных сессиях), потом создать index на parent (без CONCURRENTLY, но быстро — он только «attach» уже существующие).

## Constraint exclusion vs declarative

Исторически до Postgres 10 партиционирование делалось через **inheritance + check constraints + constraint exclusion**. Родительская таблица, дочерние наследуют её, у каждой child своё `CHECK (created_at >= '2024-01-01' AND < '2024-02-01')`. Planner при запросе смотрит на constraints каждой child, решает какие сканировать. Ручной роутинг через триггер на parent, ручное создание партиций. Много кода, много ловушек (забыли обновить constraint — пропущенные данные), медленный planning на большом числе партиций.

Postgres 10 добавил **declarative partitioning**: `PARTITION BY RANGE`, `PARTITION OF ... FOR VALUES FROM ... TO`. Роутинг INSERT'ов автоматический, constraints выводятся из partition boundaries. Это тот tooling, который надо использовать в новых проектах.

Знать legacy inheritance подход всё ещё полезно потому что: (1) старые системы работают на нём и мигрировать не тривиально; (2) inheritance даёт flexibility, которой нет в declarative — можно партиционировать по expression, использовать разные типы констрейнтов; (3) partition pruning в старом режиме управляется параметром `constraint_exclusion` (`on`/`off`/`partition`) — по умолчанию `partition` (работает только для inheritance-based partitioned tables). Для declarative pruning всегда on, отдельного параметра нет.

Разница в производительности: declarative planning значительно быстрее, особенно при большом числе партиций. Inheritance-based с 100+ партициями начинает тормозить на этапе planning, потому что constraint exclusion проверяет каждую child таблицу по её constraints последовательно.

## Foreign keys и partitioned tables

Матрица по версиям — стоит помнить, потому что грабли:

| Postgres | Partitioned as target of FK | Partitioned as source of FK |
|----------|------------------------------|------------------------------|
| ≤ 10     | Нет (declarative вообще нет для partitions) | Нет |
| 11       | Да, но с ограничениями | Нет |
| 12+      | Да | Да |
| 15+      | + FK от partitioned на partitioned работает нормально | |

Основное ограничение до 12-й версии — нельзя было делать FK **из** partitioned таблицы. То есть если Orders партиционирована, а Users нет — FK `orders.user_id → users.id` не создавалась. Приходилось либо не делать FK (полагаться на app-level контроль), либо не партиционировать Orders.

С Postgres 12 FK работают, но remember: если целевая таблица partitioned, FK-колонки должны включать её partition key. Классическая цепочка:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    ...
);

CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT NOT NULL REFERENCES users(id),  -- работает, users не partitioned
    created_at TIMESTAMP NOT NULL,
    ...
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE order_items (
    id BIGSERIAL,
    order_id BIGINT NOT NULL,
    order_created_at TIMESTAMP NOT NULL,
    -- FK на partitioned table — должен включать её partition key
    FOREIGN KEY (order_id, order_created_at) REFERENCES orders(id, created_at),
    ...
);
```

Приходится тащить `order_created_at` в дочернюю таблицу — не эстетично, но необходимо для FK. Многие в таких случаях отказываются от FK и делают проверки на уровне приложения.

## Attach и detach партиций

`ATTACH PARTITION` — подключить существующую таблицу как партицию. Полезно для миграции: подготавливаешь новую таблицу отдельно (загружаешь данные, строишь индексы, вакуумишь) — потом атомарно attach'ишь.

```sql
CREATE TABLE orders_2024_06 (LIKE orders INCLUDING ALL);
-- загружаешь данные, строишь индексы
COPY orders_2024_06 FROM '/tmp/2024_06.csv';
CREATE INDEX ON orders_2024_06 (user_id);
VACUUM ANALYZE orders_2024_06;

-- атомарный attach
ALTER TABLE orders ATTACH PARTITION orders_2024_06 
    FOR VALUES FROM ('2024-06-01') TO ('2024-07-01');
```

**Ключевая ловушка ATTACH** — блокировка и валидация. PostgreSQL должен проверить что все существующие строки таблицы попадают в объявленный range. Без специальных мер он берёт `ACCESS EXCLUSIVE` lock на **parent таблицу** — блокирует все SELECT'ы и DML на всей partitioned table на время валидации.

Обход: перед ATTACH добавить check constraint, который «matches» partition boundary:

```sql
ALTER TABLE orders_2024_06 
    ADD CONSTRAINT orders_2024_06_check 
    CHECK (created_at >= '2024-06-01' AND created_at < '2024-07-01') NOT VALID;

ALTER TABLE orders_2024_06 
    VALIDATE CONSTRAINT orders_2024_06_check;
    
ALTER TABLE orders ATTACH PARTITION orders_2024_06 
    FOR VALUES FROM ('2024-06-01') TO ('2024-07-01');
```

При наличии эквивалентного констрейнта ATTACH пропускает валидацию — берёт лёгкий lock, работает мгновенно. `NOT VALID` + `VALIDATE` — валидация констрейнта на существующих строках без ACCESS EXCLUSIVE (SHARE UPDATE EXCLUSIVE, DML не блокирует).

**DETACH** — обратная операция, отсоединить партицию не удаляя её:

```sql
ALTER TABLE orders DETACH PARTITION orders_2024_01;
-- orders_2024_01 теперь обычная standalone таблица
```

С Postgres 14 доступен `DETACH PARTITION CONCURRENTLY` — не берёт ACCESS EXCLUSIVE, DML продолжает работать. Полезно для safe drop старых данных: DETACH → потом отдельно DROP TABLE, между этими шагами таблица уже не в partitioned set, но живёт своей жизнью и может быть переиспользована/архивирована.

## Row movement: UPDATE меняющий partition key

Что происходит когда `UPDATE orders SET created_at = '2024-07-15' WHERE id = 42` затрагивает строку в партиции `orders_2024_06`? Новое значение `created_at` больше не попадает в диапазон `2024_06`, оно должно быть в `2024_07`.

До Postgres 11 — ошибка: `ERROR: new row for relation "orders_2024_06" violates partition constraint`. Строку нельзя было переместить между партициями через UPDATE.

С Postgres 11 — **автоматический row movement**: PostgreSQL делает DELETE из старой партиции и INSERT в новую внутри одного UPDATE. Атомарно с точки зрения транзакции.

Но операция дорогая. DELETE+INSERT вместо простого UPDATE = 2x работа с индексами (удалить из старых, добавить в новые), TID изменился (важно для конкурентных запросов, триггеров), возможные проблемы с FK (старая строка удаляется, новая появляется — cascade срабатывает).

Практическое правило: **не меняй partition key**. Если возможно — сделай partition key immutable в бизнес-смысле (created_at, user_id — не меняются). Если приходится менять — знай про стоимость и делай пакетно, не по одной строке.

Row movement также обязателен для corner cases: `INSERT ... ON CONFLICT DO UPDATE`, где UPDATE меняет partition key. Тоже работает, но с той же стоимостью.

## Statistics и planning per partition

PostgreSQL хранит статистику per partition — каждая партиция имеет свой `pg_statistic`. Это правильно для partition pruning (planner видит реальное распределение внутри каждой партиции), но создаёт нюансы.

**ANALYZE parent** запускает ANALYZE на всех child партициях последовательно. Долго при много партиций. С Postgres 10+ также обновляется **partition-level статистика в parent** (`inhrelid IS NOT NULL AND stainherit = true`) — combined статистика для запросов без partition pruning.

Autovacuum триггерит analyze по threshold независимо на каждой партиции: `autovacuum_analyze_threshold` (default 50) + `autovacuum_analyze_scale_factor` (default 0.1) применяются к каждой child. Свежая партиция получает первый ANALYZE после 50 insert'ов + 10% от 0 (то есть первые 50) — быстро. Большая старая партиция с 10 млн строк — только после 1 млн изменений (10%). Возможна проблема stale статистики на больших редко меняющихся партициях.

**Extended statistics** — статистика по correlated колонкам (`CREATE STATISTICS ...`) — работает на уровне отдельных партиций, не автоматически на parent. Приходится создавать вручную на каждой child, что неудобно.

## Autovacuum и партиции

Autovacuum обрабатывает каждую партицию как отдельную таблицу. Триггерится по своим порогам per partition. Идёт в один worker (обычно), максимум `autovacuum_max_workers` параллельно на весь кластер.

**Преимущество**: горячая партиция (текущий месяц) вакуумится часто, cold партиции (старые месяцы) — редко или никогда (если не меняются). VACUUM на 10-миллионной партиции быстрее чем на 100-миллионной non-partitioned таблице.

**Проблема — параллелизм autovacuum ограничен**. Если у вас 100 партиций и все активно меняются, а `autovacuum_max_workers = 3` — три партиции вакуумятся одновременно, остальные ждут. Backlog vacuum'а растёт. Autovacuum может отставать.

Тюнинг для partitioned tables:

```sql
-- Per-table override для активных партиций
ALTER TABLE orders_2024_06 SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- вакуум чаще
    autovacuum_analyze_scale_factor = 0.02  -- analyze чаще
);
```

Cold партиции — увеличить `autovacuum_freeze_max_age` чтобы reject лишних anti-wraparound vacuum'ов (см. файл 89 про VACUUM).

**Freezing**. TXID wraparound защита требует периодического freeze. На большой cold партиции freeze полной таблицы = долгий read + write всех страниц. Партиционированность помогает — freeze делается по одной партиции за раз, не всей таблицы сразу.

## Locking семантика на партициях

Понимание что блокирует что критично для конкурентности.

**SELECT на partitioned table** — берёт `ACCESS SHARE` на parent + `ACCESS SHARE` на каждую партицию, которая **не была исключена pruning**. При хорошем pruning — только на нужные. При плохом — на все.

**INSERT** — `ROW EXCLUSIVE` на parent + `ROW EXCLUSIVE` на конкретную target партицию.

**UPDATE / DELETE без row movement** — то же самое.

**UPDATE с row movement** — `ROW EXCLUSIVE` на parent + на **обе** партиции (старая, новая).

**ATTACH PARTITION** — `SHARE UPDATE EXCLUSIVE` на parent (не блокирует SELECT / DML), `ACCESS EXCLUSIVE` на подключаемую таблицу. Валидация может занять время если нет соответствующего constraint (см. выше).

**DETACH PARTITION** — `ACCESS EXCLUSIVE` на parent (блокирует всё!) плюс на детачируемую партицию. С Postgres 14 `DETACH ... CONCURRENTLY` — `SHARE UPDATE EXCLUSIVE`, безопасно для DML.

**DROP PARTITION** через `DROP TABLE partition_name` — быстро (метаданные), но берёт `ACCESS EXCLUSIVE` на parent на короткий момент.

**CREATE INDEX на parent** — `SHARE` на parent и каждую партицию, блокирует DML на время создания на каждой. `CONCURRENTLY` — работает per partition последовательно с более лёгкими локами.

Правило: partitioned tables обычно **лучше для параллельности** чем non-partitioned (локи локальные), но операции над parent часто требуют lock на всех партициях — могут быть неожиданно тяжёлыми.

## Default partition — почему её лучше не иметь

`DEFAULT` партиция принимает записи которые не попадают ни в одну другую партицию:

```sql
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

Кажется безопасным (не потеряем данные с датой в 2050 если забыли создать партицию). На деле — источник проблем:

1. **Ломает partition pruning для будущих партиций**. Когда потом добавляешь `orders_2025_01`, PostgreSQL должен проверить что в default партиции нет строк, которые должны туда переехать. При `ATTACH PARTITION` на существующий диапазон, если default не пустая — ATTACH сканирует default целиком под ACCESS EXCLUSIVE. Медленно, блокирующе.

2. **Скрывает ошибки**. Строки с «неправильной» датой (баг в приложении) молча уходят в default. Через месяцы обнаруживаешь что default вырос до сотен миллионов строк — а никто не знал.

3. **Pruning на default** — сложнее. Planner должен доказать что value не попадает в default (то есть попадает в существующую партицию) — работает, но добавляет overhead.

Практика: **не создавайте default партицию в prod**. Лучше:
- Автоматизировать создание партиций (pg_partman создаёт следующие партиции заранее).
- Если запись не попадает — INSERT падает с явной ошибкой, приложение обрабатывает как invalid data.

## pg_partman: автоматизация

Ручное управление партициями (`CREATE TABLE ... PARTITION OF`, `DROP TABLE`, `ALTER TABLE`) быстро становится сложным при 12-100+ партициях. **pg_partman** — расширение автоматизирующее lifecycle: создание будущих партиций заранее (retention forward), удаление старых (retention backward), поддержка default partition правильно.

Установка и настройка:

```sql
CREATE EXTENSION pg_partman;

SELECT partman.create_parent(
    p_parent_table => 'public.orders',
    p_control => 'created_at',
    p_type => 'native',
    p_interval => '1 month',
    p_premake => 4  -- создать 4 будущие партиции заранее
);

-- Настроить retention: удалять партиции старше 6 месяцев
UPDATE partman.part_config 
SET retention = '6 months',
    retention_keep_table = false  -- реально DROP
WHERE parent_table = 'public.orders';
```

Периодический вызов `SELECT partman.run_maintenance()` через pg_cron или внешний scheduler — создаёт новые партиции, удаляет старые, поддерживает набор partitions в consistent состоянии.

**Реальный prod-паттерн**: развернуть pg_partman + pg_cron, настроить run_maintenance каждую ночь, забыть о ручной работе с партициями. Стандарт в enterprise системах где идёт time-series data (логи, события, транзакции).

## Практические паттерны миграции

Миграция обычной таблицы на partitioned без даунтайма — задача сложная. Три основных подхода:

**Подход 1: dual-write на уровне приложения**. Приложение начинает писать одновременно в старую и новую (partitioned) таблицы. Данные из старой переливаются в новую в фоне (batch INSERT-SELECT). Когда таблицы синхронизированы — переключается чтение на новую. Потом отключается запись в старую. Медленно, требует изменений в коде, но безопасно и обратимо.

**Подход 2: INSERT-SELECT в batch с использованием окна времени**. Создаётся partitioned таблица параллельно. Каждый батч переносит данные за окно (по 100k строк или по одному дню). Приложение продолжает писать в старую таблицу. Финальный переключатель — короткий downtime (секунды): LOCK старой таблицы, доперенос последних данных, RENAME старой → old, RENAME новой → правильное имя.

**Подход 3: ATTACH PARTITION готовой таблицы**. Если старая таблица уже помещается в диапазон одной партиции (например, все данные за 2020-2023, а новые партиции ежемесячные) — можно превратить старую таблицу в первую партицию через ATTACH. Не требует переноса данных! Но подходит только для конкретной ситуации.

Все три подхода подробнее в файле 91. Общее правило: не начинать миграцию без плана rollback (что делать если посреди процесса выяснится что новая схема не подходит).

## Диагностика в проде

**Проверить структуру партиционирования**:

```sql
SELECT 
    parent.relname AS parent_table,
    child.relname AS partition,
    pg_get_expr(child.relpartbound, child.oid) AS partition_bound,
    pg_size_pretty(pg_relation_size(child.oid)) AS size,
    pg_stat_get_live_tuples(child.oid) AS rows
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
WHERE parent.relname = 'orders'
ORDER BY child.relname;
```

Даёт список партиций, их bounds, размеры, число строк. Быстро видно неравномерность (одна партиция сильно больше остальных = hotspot).

**Проверить сработал ли partition pruning**:

```sql
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders WHERE created_at >= '2024-05-01' AND created_at < '2024-06-01';
```

Смотреть на:
- Число партиций в плане (должна быть одна для этого запроса).
- `Subplans Removed: N` — сработал execution-time pruning.
- `Rows Removed by Filter` — сколько строк отфильтровано после чтения (если много — pruning не помог, читалось всё).

Если pruning не работает — типичные причины: partition key не в WHERE напрямую (спрятан за функцией); type mismatch (`created_at :: date` vs partition key `timestamp`); prepared statement с параметром чужого типа.

**Найти самую большую партицию** (потенциальный hotspot):

```sql
SELECT 
    child.relname,
    pg_size_pretty(pg_total_relation_size(child.oid)) AS total_size,
    pg_stat_get_live_tuples(child.oid) AS rows,
    pg_stat_get_tuples_inserted(child.oid) AS inserted,
    pg_stat_get_tuples_updated(child.oid) AS updated,
    pg_stat_get_tuples_deleted(child.oid) AS deleted
FROM pg_inherits
JOIN pg_class parent ON inhparent = parent.oid
JOIN pg_class child ON inhrelid = child.oid
WHERE parent.relname = 'orders'
ORDER BY pg_total_relation_size(child.oid) DESC
LIMIT 5;
```

**Диагностика autovacuum на партициях**:

```sql
SELECT 
    child.relname,
    n_live_tup,
    n_dead_tup,
    round(100 * n_dead_tup::numeric / NULLIF(n_live_tup, 0), 2) AS dead_pct,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables psut
JOIN pg_class child ON child.oid = psut.relid
JOIN pg_inherits ON child.oid = inhrelid
JOIN pg_class parent ON inhparent = parent.oid
WHERE parent.relname = 'orders'
ORDER BY n_dead_tup DESC
LIMIT 10;
```

Партиции с большим `dead_pct` (>20%) и давним `last_autovacuum` — не успевает autovacuum, надо тюнить настройки per partition.

**Список активных запросов, работающих с конкретной партицией**:

```sql
SELECT pid, state, wait_event, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE query LIKE '%orders_2024_06%'
  AND state != 'idle';
```

Смотреть кто и как долго держит партицию. Если ATTACH ждёт часами — pg_blocking_pids покажет кого он блокирует.

## Типовые ошибки

**Партиционировать маленькую таблицу.** Overhead partitioning заметен. Планировщик обрабатывает все партиции даже с pruning (просто списком). Metadata растёт. Порог полезности — обычно 10+ млн строк или 10+ GB данных. Меньше — вреднее чем полезнее.

**Слишком много партиций.** 100-500 — норма. 1000+ — уже медленно на planning стороне (каждая новая партиция = дополнительная работа для planner'а). 10000 — почти нежизнеспособно без специальных настроек. Правило: если возникает соблазн иметь тысячи партиций — вероятно нужен другой подход (hash sub-partitioning или sharding).

**Partition key не в WHERE большинства запросов.** Партиционирование даёт максимум только если запросы фильтруют по partition key. Партиционировали по `country`, а 80% запросов по `user_id` — все запросы читают все партиции, эффект отрицательный.

**FK ломаются после партиционирования.** Забыли что FK на partitioned target должен включать partition key, миграция ложится. Пересмотреть контракты FK до партиционирования.

**Default partition оставили в проде.** Скрывает баги, тормозит будущие ATTACH. Убрать по возможности.

**UPDATE с row movement на hot таблице.** Каждый UPDATE partition key = DELETE+INSERT на индексах. Массовые изменения (миграция данных) через UPDATE лучше делать batch-мигрированием: INSERT в новую партицию → DELETE из старой. Или ATTACH готовой партиции если применимо.

**Забыли Composite PK.** Написали `PRIMARY KEY (id)`, получили ошибку `must include all partitioning columns`. Пришлось переделывать. При проектировании partitioned tables PK всегда содержит partition key.

## Заключение

Партиции в PostgreSQL — не просто «разбить большую таблицу». Это набор конкретных механизмов planner'а (partition pruning в plan-time и execution-time), storage layer (независимые heap и индексы), и operational tooling (ATTACH/DETACH, per-partition autovacuum). Понимание этих механизмов — разница между «партиционировали, всё стало быстрее» и «партиционировали, всё стало медленнее из-за planner overhead и cross-partition queries».

Ключевые механики: **plan-time pruning** работает при известных литералах в WHERE, **execution-time pruning** — для prepared statements и параметров (Subplans Removed в EXPLAIN). Ломается pruning типичными ошибками: partition key не в WHERE, type mismatch, функции вокруг partition key.

**Индексы локальные** на каждой партиции — нет глобального индекса через все партиции. Отсюда — unique constraint должен включать partition key, PRIMARY KEY становится composite. `CREATE INDEX ON parent` создаёт на всех child (с Postgres 11+), для CONCURRENTLY на всех — часовая операция.

**Foreign keys** работают полноценно с Postgres 12+ (as source), FK-колонки должны включать partition key целевой таблицы. Часто приводит к «дополнительному» переносу колонок в дочерние таблицы.

**ATTACH PARTITION** без предварительного CHECK берёт ACCESS EXCLUSIVE lock — блокирует всё. Правильный паттерн: сначала `CHECK ... NOT VALID` + `VALIDATE CONSTRAINT` (лёгкий lock), потом `ATTACH` мгновенно. **DETACH CONCURRENTLY** (Postgres 14+) — безопасный вариант отключения.

**Row movement** (UPDATE меняющий partition key) — автоматически работает с Postgres 11+, но дорого. Правило: делать partition key immutable в бизнес-смысле.

**Statistics per partition** — точнее planning, но stale статистика на больших редко меняющихся партициях. `ALTER TABLE ... SET (autovacuum_...)` для тюнинга per-partition.

**Autovacuum по одной партиции параллельно** максимум `autovacuum_max_workers` штук — при много активных партициях может отставать.

**Default partition — anti-pattern**. Ломает ATTACH, скрывает баги. Лучше автоматизация создания партиций через pg_partman.

**pg_partman + pg_cron** — стандартный prod-паттерн для time-series данных. Автоматически создаёт будущие партиции, удаляет старые.

Миграция обычной таблицы в partitioned — тяжело без даунтайма. Три подхода: dual-write, batch INSERT-SELECT с коротким окном для swap, ATTACH готовой таблицы как первой партиции.

Диагностика в проде: `pg_inherits` для структуры, `EXPLAIN ANALYZE` для проверки pruning (Subplans Removed), `pg_stat_user_tables` для per-partition метрик autovacuum и hotspot detection.

Типовые ошибки — партиционировать маленькое (overhead > выгода), слишком много партиций (planner тормозит), partition key не в WHERE (весь смысл теряется), забытый composite PK, оставшаяся default partition.

Postgres-специфичное практическое introduction — в 91. Здесь была глубина в механику: как реально работает pruning, что делает автовакуум per partition, где ловушки с FK и индексами, как правильно ATTACH без downtime, чем ловить проблемы в проде.
