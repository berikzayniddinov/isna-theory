# 116. PostgreSQL физически: диск, shared buffers, IO wait, bloat, TOAST — полная структура

## Зачем это знать

БД для многих разработчиков — «где хранятся данные». Работает через ORM, возвращает результаты. Но когда начинается прод — вопросы. Почему после массового DELETE миллиона строк таблица не уменьшилась, а осталась 10 GB. Почему `SELECT count(*)` на таблице в 5 GB занимает 10 секунд, а cache hit rate 100%. Почему один и тот же запрос иногда 5 мс, а иногда 500 мс — и в моменте 500 мс `iostat` показывает `%iowait 60%`. Что такое TOAST в логах и почему туда «попадает» большой JSON-документ. Почему VACUUM работал 3 часа и таблица всё равно 8 GB dead tuples.

Все эти вопросы — про **физическую структуру** PostgreSQL. Как реально данные лежат на диске. Как работает буферный менеджер в RAM. Что происходит между вызовом `SELECT` и приходом байтов из SSD. Что такое bloat и почему он появляется. Как большие значения хранятся отдельно от основной таблицы. Как WAL взаимодействует с heap файлами. Разница между `iowait` в top и реальной проблемой с диском.

Разница между «знаю SQL» и «понимаю PostgreSQL физически» — способность за минуту диагностировать: таблица 50 GB на диске но 5 GB данных → bloat, надо VACUUM FULL или pg_repack. Cache hit rate 99% а запросы медленные → не в RAM проблема, а в другом (JIT, complex plans, sort spill to disk). `iowait 60%` на сервере БД → диски задыхаются, добавить IOPS/RAM или упростить запросы. Огромный JSON в колонке медленный → TOAST decompression, оптимизировать структуру.

Разберём слой за слоем сверху вниз: что видит клиент → как это лежит в памяти → как оседает на диске. Layout data directory PostgreSQL (файлы, tablespace, WAL). Физическая структура таблицы (heap файлы, страницы 8 KB, tuples, служебные заголовки). TOAST для больших значений — почему нельзя просто хранить в heap. Free Space Map и Visibility Map — служебные файлы. Shared buffers — как работает буферный менеджер, LRU/clock sweep, dirty pages, eviction. OS page cache и double buffering. WAL — отдельный поток записи, почему на отдельном диске. Checkpoint — синхронизация памяти с диском, как это влияет на нагрузку. Bloat — что это фундаментально, как измерить, как чинить (VACUUM обычный, VACUUM FULL, pg_repack). Index bloat отдельно. Fill factor — почему важен для UPDATE-heavy таблиц. Temp files — spill to disk когда work_mem не хватает. IO wait на уровне OS — что это, что не это, как связано с БД. Практическая диагностика в проде — какие SQL запросы дают полную картину.

Cross-refs: 87 — pages/IO детально, 88 — locks/EXPLAIN, 89 — VACUUM/statistics, 105 — autovacuum tuning, 106 — ALTER TABLE lock queue, 107 — connection pool + I/O wait от приложения. Здесь фокус на физике самой БД.

## Data directory: где вообще PostgreSQL хранит файлы

PostgreSQL хранит всё в одной директории — **PGDATA** (`/var/lib/postgresql/data` в стандартной установке Linux). Внутри — структура файлов:

```
/var/lib/postgresql/data/
├── base/                    ← основные данные (файлы таблиц и индексов)
│   ├── 1/                   ← база 'template1'
│   ├── 13012/               ← база 'postgres'
│   └── 16384/               ← твоя база (номер = OID из pg_database)
│       ├── 16385            ← файл таблицы (номер = OID из pg_class)
│       ├── 16385_fsm        ← Free Space Map
│       ├── 16385_vm         ← Visibility Map
│       ├── 16386            ← файл индекса
│       └── ...
├── global/                  ← глобальные объекты (роли, tablespace)
├── pg_wal/                  ← WAL segments (было pg_xlog до PG 10)
│   ├── 000000010000000000000001    ← 16 MB WAL сегмент
│   ├── 000000010000000000000002
│   └── ...
├── pg_stat/                 ← runtime статистика
├── pg_stat_tmp/             ← временная статистика
├── pg_notify/               ← LISTEN/NOTIFY
├── pg_tblspc/               ← симлинки на tablespaces
├── postgresql.conf          ← конфиг
├── pg_hba.conf              ← аутентификация
└── PG_VERSION               ← версия
```

**Каждая таблица** — один или несколько файлов. Основной файл — heap (сами данные). Плюс _fsm (карта свободного места) и _vm (карта видимости). Плюс каждый индекс — свой файл.

**Segment size**. Один файл heap растёт до 1 GB (`RELSEG_SIZE`), затем создаётся следующий сегмент `.1`, `.2`, `.3`. Так таблица в 5 GB — 5 файлов: `16385`, `16385.1`, `16385.2`, `16385.3`, `16385.4`. Сделано для совместимости с файловыми системами не поддерживающими файлы > 2/4 GB (сейчас неактуально, но осталось).

**Tablespaces**. Можно разместить таблицу или индекс на отдельном диске:

```sql
CREATE TABLESPACE fastspace LOCATION '/mnt/nvme/postgres';
CREATE TABLE hot_data (...) TABLESPACE fastspace;
```

Полезно когда часть таблиц на NVMe (горячие), часть на HDD (архивные логи). Или WAL на отдельном диске от heap (см. ниже).

**Найти файл конкретной таблицы**:

```sql
SELECT pg_relation_filepath('my_table');
-- base/16384/16385
```

Или:

```sql
SELECT oid, relname FROM pg_class WHERE relname = 'my_table';
```

Полученный OID — номер файла в `base/`.

## Физическая структура таблицы: страницы 8 KB и tuples

Данные внутри heap файла организованы **страницами** (pages) фиксированного размера **8 KB** (`BLCKSZ`). Страница — единица чтения/записи. Меньше нельзя. Один SELECT одной строки — читает всю страницу где лежит эта строка.

Схема страницы:

```
┌───────────────────────────────────────────┐  0
│  Page Header (24 bytes)                   │
│  - LSN (log sequence number)              │
│  - checksum                               │
│  - flags                                  │
│  - pd_lower / pd_upper (границы free)     │
├───────────────────────────────────────────┤  24
│  Line pointers (item IDs):                │
│  [ID1: offset=8144, len=48]               │
│  [ID2: offset=8080, len=64]               │
│  [ID3: offset=8000, len=80]               │
│  ...                                      │
├───────────────────────────────────────────┤  pd_lower
│                                            │
│         Free space                         │
│                                            │
├───────────────────────────────────────────┤  pd_upper
│         Tuple 3                            │
│         Tuple 2                            │
│         Tuple 1                            │
├───────────────────────────────────────────┤  8144
│  Special space (для индексов)              │
└───────────────────────────────────────────┘  8192 (8 KB)
```

**Line pointers** растут сверху вниз (после header'а). Каждый — 4 байта: offset + length + flags. Указывает на реальный tuple ниже.

**Tuples** растут снизу вверх. Между ними и line pointers — free space.

Такая структура даёт **стабильные TID**: TID = (page number, line pointer index). Например TID `(5, 3)` = страница 5, item 3. Line pointer с индексом 3 может со временем указывать на разные offset'ы (если tuple перемещался внутри страницы), но TID остаётся тем же. Индексы хранят TID'ы — им не нужно переиндексироваться когда tuple двигается.

**Что физически внутри tuple**:

```
┌─────────────────────────────────────┐
│  Tuple Header (23-27 bytes):        │
│    - xmin (transaction ID created)  │  4 bytes
│    - xmax (transaction ID deleted)  │  4 bytes
│    - cmin/cmax (command IDs)        │  4 bytes
│    - CTID (self reference)          │  6 bytes
│    - flags, natts                   │  ~4 bytes
│    - NULL bitmap (если есть NULLs)  │  variable
├─────────────────────────────────────┤
│  Column values:                     │
│    - id (int8): 4 bytes             │
│    - name (text): 1 byte len + data │
│    - email (text): 1 byte len + data│
│    - created_at (timestamp): 8 bytes│
└─────────────────────────────────────┘
```

Каждый tuple несёт **overhead ~24 байта** служебных полей. Для маленьких строк (2-3 int колонки) overhead может быть больше самих данных. Для больших — незначителен.

**xmin/xmax** — сердце MVCC. Каждая транзакция при чтении tuple смотрит: xmin (кто создал) уже закоммитился до моего старта? xmax (кто удалил) уже закоммитился? По этому решает — видим ли tuple мне.

## TOAST: как хранятся большие значения

Проблема — tuple не может быть больше страницы (8 KB). Что если поле `description` содержит текст на 100 KB? Или JSON-документ на 200 KB?

**TOAST (The Oversized-Attribute Storage Technique)** — механизм хранения больших значений вне основной страницы.

Когда tuple вместе с полями превышает `TOAST_TUPLE_THRESHOLD` (~2 KB по умолчанию), PostgreSQL применяет к «большим» полям стратегию:

1. **Compress** — попытаться сжать значение (LZ-like алгоритм внутри PostgreSQL).
2. Если после compress'а всё ещё большое — **разбить на chunks** по 2 KB и сохранить в отдельную **TOAST таблицу**.
3. В основной странице оставить только указатель на TOAST записи.

**TOAST таблица** — отдельная таблица `pg_toast.pg_toast_<oid>` создаётся автоматически для каждой обычной таблицы имеющей потенциально большие поля (text, bytea, jsonb, arrays). Найти:

```sql
SELECT reltoastrelid::regclass 
FROM pg_class WHERE relname = 'my_table';
-- pg_toast.pg_toast_16385
```

**Стратегии TOAST** (задаётся per column):
- **PLAIN** — не сжимать, не выносить. Только для типов не поддерживающих TOAST.
- **EXTERNAL** — выносить, но не сжимать. Быстрее чтение, больше диск.
- **EXTENDED** (дефолт) — сжимать И выносить если нужно.
- **MAIN** — сжимать, но избегать выноса. Хранить в основной странице сжатым если возможно.

**Практическое следствие**. Большой JSONB — часто TOAST'ится. Каждое чтение — decompress + подтягивание chunks. Медленнее чем int. Отсюда — если JSONB часто читаешь целиком, но редко нужны отдельные ключи, эффективнее хранить как text и парсить в приложении.

**Проверить какая колонка TOAST'ится**:

```sql
SELECT attname, attstorage 
FROM pg_attribute 
WHERE attrelid = 'my_table'::regclass AND attnum > 0;
-- attstorage: p=PLAIN, e=EXTERNAL, m=MAIN, x=EXTENDED
```

**Симптом TOAST-проблем** — тяжёлые SELECT'ы простых полей плюс одного большого поля. Read amplification: планировщик читает основную страницу + TOAST страницы + de-toasts (decompress + assemble chunks). Если большое поле не нужно — `SELECT id, name, ...` без большого поля значительно быстрее чем `SELECT *`.

## Free Space Map и Visibility Map

Кроме основного heap файла — два служебных.

**Free Space Map (`_fsm`)** — компактная карта показывающая сколько свободного места в каждой странице. Когда INSERT ищет куда положить новую строку — читает FSM, находит страницу с достаточно free space. Без FSM пришлось бы линейно сканировать страницы.

FSM обновляется при каждом изменении страницы. Небольшой overhead. Обычно ~1/1000 от размера основного файла.

**Visibility Map (`_vm`)** — битовая карта, один бит на страницу: «все tuples в этой странице видны всем транзакциям» (all-visible) + «все frozen» (frozen).

Использование:
- **Index Only Scan** — если планировщик знает что все tuples страницы видны, не нужно ходить в heap за MVCC-проверкой. Достаточно данных из индекса. Значительно быстрее.
- **VACUUM оптимизация** — страницы помеченные all-frozen пропускаются при freeze-vacuum (антивраппераунд), экономит IO.

VM обновляется при VACUUM. Свежепринятые INSERT/UPDATE обнуляют бит для затронутых страниц (там теперь есть tuples видные не всем).

## Shared Buffers: буферный менеджер в RAM

Диск в тысячи раз медленнее RAM. PostgreSQL кэширует часто читаемые страницы в памяти — **shared buffers**.

**Что это**. Область shared memory (доступна всем backend процессам), содержащая кэшированные страницы heap и индексов. Размер задаётся `shared_buffers` в `postgresql.conf`. Дефолт — 128 MB. Обычно ставят **25% RAM сервера** (для 16 GB RAM → 4 GB shared buffers). Больше 40% — обычно не окупается, начинает конкурировать с OS page cache.

**Структура**. shared_buffers разбит на **буферы по 8 KB** (размер страницы). Каждый буфер имеет заголовок с метаданными: какая страница загружена, dirty ли, счётчик использования, pin count (кто-то читает прямо сейчас), lock состояние.

**Buffer descriptor** (упрощённо):

```
┌─────────────────────────┐
│ tag = (relfilenode, forknum, blocknum)  │ ← какая страница
│ state (dirty, valid)    │
│ usage_count (0-5)       │  ← как часто используется
│ refcount / pin count    │  ← кто читает
│ lock                    │
├─────────────────────────┤
│ Buffer content (8 KB)   │  ← собственно страница
└─────────────────────────┘
```

**Как читается страница**:

1. Backend хочет страницу X таблицы Y. Ищет в **buffer hash table**: `hash(Y, X) → buffer_id или NULL`.
2. **Cache hit** → страница в памяти. Pin buffer (защита от eviction), читать, unpin. Микросекунды.
3. **Cache miss** → надо загрузить с диска. Найти пустой buffer (или evict'нуть чей-то). Прочитать 8 KB с диска. Заполнить buffer. Обновить hash table.

**Eviction через Clock Sweep**. Когда нужен новый buffer, PostgreSQL идёт по кругу через все buffers:
- Если `pin count > 0` (кто-то держит) — пропустить.
- Если `usage_count > 0` — уменьшить на 1, идти дальше.
- Если `usage_count == 0` — evict этот buffer.

Аналог LRU но проще. Часто используемые страницы имеют высокий usage_count → «выживают».

При eviction dirty buffer'а — сначала записать на диск (flush). Отсюда — если dirty buffers много, eviction замедляется.

**Cache hit rate — метрика здоровья**:

```sql
SELECT sum(heap_blks_hit) * 100.0 / 
       nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS cache_hit_ratio
FROM pg_statio_user_tables;
```

- **99%+** — здоровая OLTP-нагрузка, hot data влезает в память.
- **95-99%** — приемлемо, но есть куда расти.
- **< 95%** — недостаточно памяти, рабочий набор не влезает. Увеличить shared_buffers или RAM.

**Как узнать что в shared buffers прямо сейчас**:

Установить extension `pg_buffercache`:

```sql
CREATE EXTENSION pg_buffercache;

SELECT c.relname, count(*) AS buffers,
       pg_size_pretty(count(*) * 8192) AS size
FROM pg_buffercache b 
JOIN pg_class c ON b.relfilenode = c.relfilenode
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 20;
```

Показывает топ таблиц/индексов по количеству буферов в кэше. Полезно понять что «горячее».

## OS page cache: второй уровень кэша

PostgreSQL читает файлы через обычные syscalls (`read`, `write`, `pread`). Linux имеет **свой кэш** — OS page cache, использует свободную RAM для кэширования файлов.

Значит одна и та же страница может быть **в двух местах одновременно**:
- В shared buffers PostgreSQL (8 KB).
- В OS page cache (та же 8 KB).

**Double buffering**. Из 16 GB RAM: 4 GB shared_buffers + 10 GB OS page cache (Linux использует). Значительная часть OS cache — те же страницы что уже в shared_buffers. «Пустая трата».

Почему PostgreSQL не отказывается от OS cache и не использует direct I/O (bypass OS cache)? Историческая причина — сложность, разные платформы. Реально overhead 20-30% RAM на дублирование считается приемлемым платой за простоту.

**Практическое следствие**. При настройке shared_buffers надо учитывать OS cache. Не ставить shared_buffers = 100% RAM — OS cache тоже нужен (плюс background процессы, connections memory, work_mem). 25% — sweet spot для большинства случаев.

**Разница cache hit rate PostgreSQL vs общий**:

- PostgreSQL cache hit rate 95% значит 5% чтений идут за пределы shared_buffers.
- Из этих 5% часть попадает в OS cache (быстро, микросекунды через memcpy).
- Часть идёт реально на диск (миллисекунды).

Значит реальный «disk hit rate» может быть 99%+ даже когда PostgreSQL cache hit 95%. Из-за OS cache. Смотреть отдельно.

## WAL: почему отдельный диск часто

Как обсуждалось (файл 116 предыдущая версия), WAL — write-ahead log, все изменения сначала пишутся туда для durability.

**Физически WAL**:
- Файлы в `pg_wal/` (было `pg_xlog/` до PG 10).
- **Segment size** — 16 MB (`--with-wal-segsize` при компиляции, менять сложно).
- Именование: `000000010000000000000001`, `...02` и т.д. — timeline + LSN.
- Circular usage — старые файлы **не удаляются** мгновенно, переиспользуются под новые записи (rename).

**Особый паттерн доступа**. WAL — **sequential write only**. Только пишем в конец файла, никогда не читаем (кроме recovery и репликации). Heap файлы — random read/write. Разные паттерны доступа = разная оптимальная организация диска.

**WAL на отдельном диске** — стандартная prod-практика. Причины:
1. Sequential write диска не мешает random read heap.
2. WAL disk может быть меньше (16-32 GB достаточно), но быстрее (NVMe с высокой write endurance).
3. Проблема с одним диском не убивает другой.

Настройка:
```bash
# Переместить pg_wal на отдельный диск
systemctl stop postgresql
mv /var/lib/postgresql/data/pg_wal /mnt/wal_nvme/pg_wal
ln -s /mnt/wal_nvme/pg_wal /var/lib/postgresql/data/pg_wal
systemctl start postgresql
```

**WAL write pattern**:
- Backend пишет WAL записи в **WAL buffers** (memory, дефолт 16 MB).
- **WAL writer** background процесс периодически flush'ит WAL buffers на диск.
- **COMMIT** — синхронный fsync WAL до момента коммита. Клиент ждёт пока диск подтвердит запись. Отсюда — WAL disk latency = COMMIT latency.

**Мониторинг**:

```sql
SELECT * FROM pg_stat_wal;
-- wal_records, wal_bytes, wal_write, wal_sync, wal_write_time, wal_sync_time
```

`wal_sync_time / wal_sync` = средняя латентность fsync. Норма < 1 мс для NVMe. > 10 мс — WAL disk медленный, ищи проблему.

## Checkpoint: синхронизация памяти с диском

Между COMMIT и физической записью в heap — задержка. Dirty pages в shared buffers ждут пока их запишут на диск. Что если сервер упадёт?

Recovery через WAL — replay всех записей начиная с последнего **checkpoint**.

**Checkpoint** — полная синхронизация shared buffers с диском:
1. Помечает точку в WAL (LSN).
2. Все dirty pages записываются на диск в heap.
3. После — WAL до этой точки не нужен для recovery (heap уже актуален).
4. Старые WAL segments могут быть удалены/переиспользованы.

**Когда checkpoint триггерится**:
- **По времени**: каждые `checkpoint_timeout` (дефолт 5 минут).
- **По объёму WAL**: когда написано `max_wal_size` (дефолт 1 GB) — форс checkpoint.
- **Явно**: `CHECKPOINT` команда.

**Проблема — checkpoint spike**. Всё dirty вываливается на диск. Если dirty много (несколько GB), диск загружен, все остальные запросы тормозят на 30-60 секунд. Симптомы: latency spikes каждые 5 минут, `%iowait` подскакивает.

**Smoothing через `checkpoint_completion_target`** (дефолт 0.9). Значит: checkpoint должен растянуться на 90% от `checkpoint_timeout`. При 5 минутах интервала — писать 4.5 минуты, чтобы IO нагрузка размазалась. Спайки становятся плоскими.

**Мониторинг checkpoints**:

```sql
SELECT * FROM pg_stat_bgwriter;
-- checkpoints_timed, checkpoints_req
-- buffers_checkpoint, buffers_clean, buffers_backend
-- checkpoint_write_time, checkpoint_sync_time
```

Если `checkpoints_req > checkpoints_timed` в разы — WAL заполняется быстрее чем чекпоинты успевают → увеличить `max_wal_size`. Если `checkpoint_write_time` растёт — write pressure, оптимизировать storage.

## Bloat: что это и почему появляется

**Bloat** — «раздувание» таблицы или индекса за счёт мёртвых tuples которые физически лежат в файлах но никому не нужны.

Причина — MVCC. UPDATE не меняет tuple, создаёт новую версию. Старая остаётся в heap с помеченным xmax. DELETE — то же самое, только новой версии нет. Физически данные остаются до VACUUM.

**Пример bloat**:

```sql
CREATE TABLE test (id serial primary key, data text);
INSERT INTO test (data) SELECT 'value ' || i FROM generate_series(1, 1000000) i;
-- Таблица ~50 MB.

UPDATE test SET data = 'updated ' || id;
-- Теперь ~100 MB на диске! 
-- 1M новых версий + 1M старых версий = 2M tuples физически.

VACUUM test;
-- Старые версии помечены как free space, но файл всё ещё 100 MB.
-- Новые INSERT'ы могут переиспользовать free space.

VACUUM FULL test;
-- Реально переписывает файл, освобождая пустоты. Таблица ~50 MB снова.
-- НО: ACCESS EXCLUSIVE lock на всё время работы — блокирует читателей!
```

**Формула bloat**:
```
bloat_ratio = (реальный_размер - идеальный_размер) / реальный_размер
```

Идеальный размер = размер данных если бы не было мёртвых tuples и было идеальное packing. Реальный — что на диске сейчас.

10-20% bloat — норма (для растущих таблиц всегда есть некоторый overhead). 30-50% — тревожно. 80-90% — катастрофа (90% таблицы — мертвецы).

**Почему автовакуум не спасает автоматически**:

- **Autovacuum триггерится по порогу**: `autovacuum_vacuum_scale_factor` (дефолт 0.2 = 20% dead tuples от live). Значит для таблицы в 100M строк autovacuum запустится когда 20M мертвецов накопилось.
- **Autovacuum медленный** по дефолту (`autovacuum_vacuum_cost_delay` — паузы между работами). Для больших таблиц может отставать.
- **VACUUM обычный** не уменьшает файл на диске. Только освобождает место для повторного использования. Если после cleanup паттерн insert'ов не заполняет освобождённое место — bloat остаётся физически.

Настройка autovacuum для big tables — файл 89, 105.

## Как найти bloated таблицы в проде

Approximate query (быстрая оценка):

```sql
SELECT schemaname, tablename,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS actual_size,
       n_dead_tup, n_live_tup,
       round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) 
           AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

`dead_pct > 20%` — кандидат на VACUUM (проверить когда был последний). > 50% — срочно.

Точная формула — extension `pgstattuple`:

```sql
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstattuple('my_table');
-- table_len | tuple_count | tuple_len | tuple_percent | dead_tuple_count | dead_tuple_len | dead_tuple_percent | free_space | free_percent
```

Показывает точные размеры: сколько байт живых tuples, сколько мёртвых, сколько free space. `dead_tuple_percent` > 20% — bloat.

## Как чинить bloated таблицы

**Обычный VACUUM** (уже разобрано выше):
- Освобождает место внутри файла (для reuse).
- Не уменьшает размер на диске.
- **Non-blocking** — можно делать в prod без остановки.
- Обычно достаточно если insert pattern заполняет освобождённое место.

**VACUUM FULL**:
- Реально переписывает файл, размер на диске уменьшается.
- **Блокирует таблицу целиком** (ACCESS EXCLUSIVE) на всё время работы. На большой таблице — часы.
- Использовать только в maintenance window.

**pg_repack** (extension) — production-friendly альтернатива VACUUM FULL:
- Создаёт копию таблицы в фоне.
- Догоняет её через триггеры на изменения.
- Атомарный swap когда готово.
- **Не блокирует** нормальные операции на протяжении процесса (только короткие exclusive lock'и на моменты swap).

```bash
# Ubuntu install
sudo apt install postgresql-16-repack

# Usage
pg_repack -d mydb -t my_table --no-order
```

Работает как VACUUM FULL по эффекту, но без длительной блокировки. Стандарт для больших prod таблиц.

**pg_squeeze** — альтернатива pg_repack без триггеров (использует logical replication). Работает похоже.

## Index bloat: отдельная история

Индексы тоже bloated. B-tree индекс при UPDATE'ах:
- Старая версия tuple → указатель в индексе на неё остаётся.
- Новая версия → новый указатель.
- VACUUM удаляет старые указатели, но структура B-tree может стать неоптимальной (полупустые страницы).

**Index bloat** может быть **больше** чем table bloat. Особенно на UPDATE-heavy таблицах, часто-меняющиеся индексы.

**Найти**:

```sql
SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(schemaname||'.'||indexname)) AS index_size,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size
FROM pg_indexes
JOIN pg_stat_user_indexes ON indexrelname = indexname
WHERE pg_relation_size(schemaname||'.'||indexname) > 100 * 1024 * 1024
ORDER BY pg_relation_size(schemaname||'.'||indexname) DESC;
```

Индекс больше самой таблицы на UPDATE-heavy — верный признак bloat.

Точно через `pgstatindex`:

```sql
SELECT * FROM pgstatindex('my_index');
-- avg_leaf_density, leaf_fragmentation
```

`avg_leaf_density < 60%` — значительный bloat. Fragmentation > 20% — тоже.

**Чинить**: **REINDEX**. С PG 12+ есть `REINDEX CONCURRENTLY` — без блокировки:

```sql
REINDEX INDEX CONCURRENTLY my_index;
```

Работает через build нового индекса в фоне, затем атомарный swap. Как pg_repack но для индексов, встроен в PostgreSQL.

Для больших индексов — часы, но безопасно для prod.

## Fill Factor: оптимизация для UPDATE-heavy таблиц

**Fill factor** — процент страницы который PostgreSQL заполняет при первичной вставке. Дефолт 100% — страница заполняется под завязку.

Проблема: UPDATE создаёт новую версию tuple. Если на текущей странице нет места — новая версия идёт в другую страницу. **HOT (Heap-Only Tuple) update** — оптимизация где новая версия помещается в ту же страницу что старая. HOT update не обновляет индексы, значительно быстрее.

Если страница заполнена на 100% (нет места для новой версии) — HOT невозможен, всё UPDATE — «cold», обновляют индексы, дорогие.

**Fill factor < 100%** оставляет место для новых версий:

```sql
CREATE TABLE hot_updated_table (...) WITH (fillfactor = 80);
-- 20% страницы оставляем свободным для HOT updates.

-- Или изменить существующую:
ALTER TABLE hot_updated_table SET (fillfactor = 80);
-- Применится к новым страницам. Существующие надо переписать через VACUUM FULL или pg_repack.
```

**Trade-off**:
- Fill factor 100% — экономия места, лучше для read-only таблиц.
- Fill factor 70-90% — быстрее UPDATE, меньше index bloat. Стандарт для heavily-updated таблиц.

**Мониторинг HOT updates**:

```sql
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0 * n_tup_hot_upd / nullif(n_tup_upd, 0), 2) AS hot_pct
FROM pg_stat_user_tables
WHERE n_tup_upd > 1000
ORDER BY n_tup_upd DESC;
```

`hot_pct` > 80% — большинство UPDATE'ов оптимальны. < 50% — рассмотреть уменьшение fill factor.

## Temp files: когда work_mem не хватает

Некоторые операции требуют памяти для промежуточных результатов: sorting (`ORDER BY`), hashing (`HASH JOIN`, `GROUP BY`), materialization (`WITH` CTE в некоторых случаях).

Параметр `work_mem` (дефолт 4 MB) — сколько памяти операция может использовать. Если не хватает — данные **сваливаются на диск** во временные файлы (`base/pgsql_tmp/`).

**Sort spill to disk**:

```
EXPLAIN ANALYZE 
SELECT * FROM orders ORDER BY total_amount;

Sort  (cost=... rows=...)
  Sort Method: external merge  Disk: 245000kB   ← 245 MB на диске!
  Sort Key: total_amount
  ->  Seq Scan on orders ...
```

`Sort Method: external merge` = spill to disk. **В разы медленнее** чем sort в памяти. Особенно медленно на HDD, приемлемо на NVMe.

**Fix**: увеличить `work_mem` для этого запроса:

```sql
SET work_mem = '512MB';
SELECT * FROM orders ORDER BY total_amount;
RESET work_mem;
```

Внимание: `work_mem` — **per operation per connection**. Если 100 concurrent connections × 3 sorts × 512 MB = 150 GB. Не ставить глобально высокие значения. Only per-session для конкретных heavy запросов.

**Мониторинг temp files**:

```sql
SELECT datname, temp_files, pg_size_pretty(temp_bytes) 
FROM pg_stat_database 
ORDER BY temp_bytes DESC;
```

Растёт temp_files/temp_bytes = запросы часто spill'ят на диск. Кандидаты для оптимизации (индексы для ORDER BY, увеличение work_mem, переписать запрос).

## IO Wait: что это и что не это

`iostat`, `top`, `htop` показывают `%iowait`. Что это значит для БД?

**`%iowait`** — процент времени CPU idle И в системе есть хотя бы один процесс ожидающий disk I/O. **CPU не занят** этим ожиданием. Он свободен, мог бы что-то делать. Просто в этот момент никто больше не хочет CPU, а вон те процессы ждут диск.

Ключевая деталь: **высокий iowait не значит высокая загрузка CPU**. Значит: диски медленные + кто-то их ждёт.

Пример `top`:

```
%Cpu(s):  5.0 us,  2.0 sy,  0.0 ni, 30.0 id, 60.0 wa,  0.0 hi,  3.0 si
```

- `id 30%` — CPU idle 30% времени.
- `wa 60%` — iowait 60%. Из 30% idle 60% времени в системе кто-то ждал диск.
- CPU суммарно свободен 90% времени, но приложение чувствует замедление.

**Проверить реально ли диски задыхаются** — `iostat -xz 1`:

```
Device  r/s   w/s   rkB/s   wkB/s   await  r_await  w_await  %util
nvme0n1 120   340   1800    5100    12.4   3.1      15.8     78.2
```

- `await` — latency I/O операции в мс. Норма для NVMe < 1 мс, SSD < 5 мс, HDD 5-20 мс. > 50 мс = проблема.
- `%util` — процент времени диск занят. > 80% на HDD = насыщение. Для NVMe со внутренним параллелизмом даже 100% не всегда насыщение (смотреть await).
- `r/s`, `w/s` — IOPS.

Если `await` высокий → диски перегружены, оптимизировать storage или уменьшить IO нагрузку.

**Связь с PostgreSQL**. Что реально ждёт диск в PostgreSQL:

- **Cache miss в shared_buffers** → read страницы с диска. Много cache miss = много disk reads.
- **WAL writes и fsync** при COMMIT → sequential writes на WAL disk.
- **Checkpoint** → burst writes dirty pages из shared_buffers на heap disk.
- **VACUUM / autovacuum** → sequential reads всей таблицы + writes VM/FSM.
- **Backup** (pg_basebackup, pg_dump) → intense sequential reads.

Симптом «БД тормозит + высокий iowait» — обычно комбинация: shared_buffers мал, checkpoint слишком частый, autovacuum отстаёт.

## Практическая диагностика в проде: SQL-запросы

Полный набор запросов для быстрой диагностики physical state базы.

**Размер БД и таблиц**:

```sql
-- Общий размер БД
SELECT pg_database.datname, 
       pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;

-- Топ таблиц по размеру (с индексами)
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) 
                    - pg_relation_size(schemaname||'.'||tablename)) AS indexes_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog','information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

**Bloat detection**:

```sql
-- Approximate через pg_stat_user_tables
SELECT schemaname, relname,
       n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
       last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY dead_pct DESC
LIMIT 20;

-- Точный через pgstattuple (медленнее, требует read всей таблицы)
SELECT * FROM pgstattuple('my_bloated_table');
```

**Cache hit rate**:

```sql
-- Общий
SELECT sum(heap_blks_hit) * 100.0 / 
       nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS cache_hit_pct
FROM pg_statio_user_tables;

-- Per table
SELECT relname,
       heap_blks_hit, heap_blks_read,
       round(100.0 * heap_blks_hit / nullif(heap_blks_hit + heap_blks_read, 0), 2) 
           AS hit_pct
FROM pg_statio_user_tables
WHERE heap_blks_read > 10000
ORDER BY heap_blks_read DESC
LIMIT 20;
```

Таблицы с большим `heap_blks_read` и низким hit_pct = хотят RAM.

**Что в shared buffers сейчас** (extension pg_buffercache):

```sql
SELECT c.relname, count(*) AS buffers,
       pg_size_pretty(count(*) * 8192) AS size,
       round(count(*) * 100.0 / 
             (SELECT setting::int FROM pg_settings WHERE name = 'shared_buffers'), 2) 
       AS pct_of_shared_buffers
FROM pg_buffercache b 
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE c.relnamespace NOT IN 
       (SELECT oid FROM pg_namespace WHERE nspname IN ('pg_catalog', 'information_schema'))
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 20;
```

**Checkpoints и WAL statistics**:

```sql
SELECT * FROM pg_stat_bgwriter;
-- checkpoints_timed / checkpoints_req
-- buffers_checkpoint / buffers_clean / buffers_backend
-- Если buffers_backend высокий — backend'ы сами flush'ат, тормозит запросы

SELECT * FROM pg_stat_wal;
-- wal_write / wal_sync
-- Средняя latency sync = wal_sync_time / wal_sync
```

**Temp files**:

```sql
SELECT datname, temp_files, pg_size_pretty(temp_bytes) 
FROM pg_stat_database 
WHERE temp_files > 0
ORDER BY temp_bytes DESC;
```

Растёт — запросы spill'ят. Найти какие через `pg_stat_statements`.

**Активные запросы и wait_events**:

```sql
SELECT pid, usename, state, wait_event_type, wait_event,
       now() - query_start AS duration,
       left(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```

`wait_event_type = 'IO'` + `wait_event = 'DataFileRead'` — запрос ждёт read с диска. Много таких — диск задыхается или shared_buffers мал.

## Полная physical архитектура: слои сверху вниз

Сборка всего в одну картину:

```
Клиент SQL
     │
     ▼
┌──────────────────────────────────────┐
│  Backend Process (per connection)     │
│  - Parser, Planner, Executor          │
│  - work_mem, temp_buffers (per proc)  │
└─────────┬────────────────────────────┘
          │ читает / пишет страницы
          ▼
┌──────────────────────────────────────┐
│  Shared Buffers (~25% RAM)            │
│  - 8 KB buffers                       │
│  - Buffer descriptors (dirty, usage)  │
│  - Clock sweep eviction               │
│  - pg_buffercache для инспекции       │
└─────────┬────────────────────────────┘
          │ cache miss → read
          │ eviction → write
          ▼
┌──────────────────────────────────────┐
│  OS Page Cache (Linux)                │
│  - Использует всю свободную RAM       │
│  - Дублирует shared_buffers           │
└─────────┬────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────┐
│  Physical Disk                        │
│  ├── PGDATA/base/                     │
│  │   ├── heap файлы таблиц            │
│  │   ├── _fsm (Free Space Map)        │
│  │   ├── _vm (Visibility Map)         │
│  │   └── индекс файлы                 │
│  ├── pg_toast/ (большие значения)     │
│  ├── pg_wal/ (WAL, часто на другом    │
│  │           диске)                    │
│  └── pgsql_tmp/ (temp files spill)    │
└──────────────────────────────────────┘

Параллельно:
- WAL Writer: buffers → pg_wal
- Checkpointer: dirty pages → heap
- Autovacuum: cleanup dead tuples
- Background Writer: dirty pages → OS
```

**Каждый уровень имеет свои метрики и симптомы проблем**:

- Backend уровень → connections, queries, plans (`pg_stat_activity`, EXPLAIN)
- Shared buffers → cache hit rate, buffer pool utilization (`pg_stat_bgwriter`, pg_buffercache)
- OS cache → нет прямой видимости в PostgreSQL, смотреть через OS (`free -h`, `cat /proc/meminfo`)
- Disk → iostat, await, %util
- WAL → pg_stat_wal, checkpoint статистика
- Bloat → pg_stat_user_tables, pgstattuple
- Temp files → pg_stat_database

## Заключение

**Физическая структура PostgreSQL** — три слоя: клиентские процессы (backend) → буферный менеджер в RAM (shared buffers + OS page cache) → диск (heap files + WAL + служебные).

**Data directory** — PGDATA. Каждая таблица — файл (или несколько сегментов по 1 GB). Плюс _fsm и _vm служебные. Каждый индекс — свой файл. WAL в pg_wal (часто на отдельном диске).

**Страницы 8 KB** — единица чтения/записи. Внутри: header + line pointers (сверху) + tuples (снизу) + free space посередине. TID = (page, item pointer) — стабильный адрес строки.

**Tuple header** несёт xmin/xmax (MVCC), CTID, флаги. Overhead ~24 байта на строку. Для маленьких строк overhead значительный.

**TOAST** для больших значений — сжатие + разбиение на chunks 2 KB → отдельная таблица `pg_toast.pg_toast_<oid>`. Настройка `PLAIN`/`EXTERNAL`/`EXTENDED`/`MAIN` per column. Read amplification при чтении TOAST'нутых полей.

**Free Space Map (_fsm)** — где есть место для INSERT. **Visibility Map (_vm)** — какие страницы all-visible (для Index Only Scan) / all-frozen (для VACUUM оптимизации).

**Shared buffers** — кэш страниц в RAM. 25% RAM оптимально. Clock sweep eviction. Cache hit rate 99%+ = здоровая OLTP. `pg_buffercache` extension для инспекции содержимого.

**OS page cache** — второй уровень кэша, double buffering. Часть RAM «теряется» на дубли. Учитывать при sizing shared_buffers.

**WAL** — sequential write only, часто на отдельном диске (NVMe для низкой latency fsync). COMMIT ждёт fsync — WAL disk latency = COMMIT latency. `pg_stat_wal.wal_sync_time / wal_sync` — средняя latency.

**Checkpoint** — полная синхронизация dirty pages с диском. Триггер по времени (5 мин) или объёму WAL (max_wal_size). Spike нагрузки → smoothing через `checkpoint_completion_target = 0.9`. Мониторинг `pg_stat_bgwriter`.

**Bloat** — dead tuples от MVCC остаются физически. UPDATE миллиона строк → таблица физически 2x. Формула: `(actual - ideal) / actual`. 10-20% норма, > 30% тревожно, > 80% катастрофа. Мониторинг через `pg_stat_user_tables.n_dead_tup / n_live_tup`, точно через `pgstattuple`.

**Чинить bloat**: обычный VACUUM (non-blocking, освобождает место внутри файла, размер не уменьшается) → VACUUM FULL (блокирует всё, реально переписывает) → **pg_repack** (production-friendly, non-blocking альтернатива VACUUM FULL). Autovacuum обычно справляется но требует настройки для больших таблиц.

**Index bloat** отдельно. UPDATE на индексируемой колонке = новый указатель, старый до VACUUM. `REINDEX CONCURRENTLY` (PG 12+) — non-blocking rebuild.

**Fill factor** — процент заполнения страницы. Дефолт 100%. Для UPDATE-heavy таблиц ставить 70-90% — оставляет место для HOT updates (быстрее, без обновления индексов). `pg_stat_user_tables.n_tup_hot_upd / n_tup_upd` — мониторинг.

**Temp files** — spill to disk когда work_mem не хватает (sorting, hashing). `Sort Method: external merge` в EXPLAIN. В разы медленнее. Fix — увеличить `work_mem` per session или оптимизировать запрос (индексы для ORDER BY).

**IO wait** в top — процент CPU idle И в системе есть ждущий диск процесс. Не значит «CPU занят», значит «диски медленные + кто-то ждёт». Реальная диагностика диска — `iostat -xz 1`, смотреть `await` (норма для NVMe < 1 мс, SSD < 5 мс, HDD 5-20 мс) и `%util`.

**Практика в проде** — набор SQL для быстрой диагностики: размеры таблиц/индексов, dead_tup, cache hit rate, что в буферах через pg_buffercache, checkpoint/WAL статистика через pg_stat_bgwriter/pg_stat_wal, temp files, активные запросы с wait_event_type='IO'.

**Все слои связаны**: cache miss → OS cache → disk → влияет на shared_buffers → влияет на eviction rate → влияет на backend performance. Autovacuum отстаёт → bloat растёт → cache miss увеличивается → disk load растёт → wait time растёт → autovacuum отстаёт ещё больше. Порочные круги реальны в PostgreSQL, надо ловить симптомы рано.

Cross-refs:
- 87 — pages/IO детально
- 88 — locks + EXPLAIN
- 89 — VACUUM/statistics/slow queries
- 105 — autovacuum tuning для больших таблиц
- 106 — ALTER TABLE lock queue + thundering herd
- 107 — connection pool + I/O wait от приложения
- 99 — postgres monitoring in production

Здесь — полная физическая структура и storage internals в одной ментальной модели.
