# 103. Index Scan vs Sequential Scan в PostgreSQL — глубокая теория

## Зачем это знать

Каждый разработчик, который хоть раз смотрел EXPLAIN, видел строки типа `Seq Scan on users` или `Index Scan using users_pkey`. Многие знают правило: «Seq Scan плохо, Index Scan хорошо». Но это не правило, а миф. Sequential Scan часто быстрее Index Scan. Более того, PostgreSQL сознательно выбирает Seq Scan в определённых случаях — потому что это правильное решение. Понимая, почему planner выбирает то или другое, и как эти сканы работают физически, вы либо избежите тысяч неоптимальных запросов, либо превратите свою БД в вечный тормоз, добавляя индексы «на всякий случай».

Есть несколько причин, почему это важно копать глубже. Первая — 90% медленных запросов в проде связаны либо с использованием неправильного скана, либо с невозможностью planner'а выбрать хороший план из-за устаревшей статистики или неоптимальных индексов. Sequential Scan на таблице в 500 миллионов строк ищущей 10 записей — это часы. Index Scan на маленькой таблице в 1000 строк — часто медленнее, чем Seq Scan. Понимая cost model, вы можете предсказать, что выберет planner, и настроить БД так, чтобы он выбирал правильно.

Вторая — типы сканов не заканчиваются на этих двух. Есть Index Only Scan (когда все нужные колонки в индексе, не идём в heap), Bitmap Index Scan + Bitmap Heap Scan (гибридный подход для средней селективности), Parallel Seq Scan (параллельное чтение таблицы несколькими worker'ами). Знать когда какой применяется и почему — базовое умение для оптимизации.

Третья — индексы не бесплатны. Каждый индекс замедляет INSERT/UPDATE/DELETE (нужно обновлять и его тоже). Занимает disk space. Требует времени на maintenance (VACUUM, REINDEX). Добавление «на всякий случай» — прямой путь к производительному аду. Понимая, когда индекс реально помогает, а когда только вредит, вы будете добавлять их обдуманно.

Мы разберём Sequential Scan детально — что происходит физически, когда PostgreSQL его выбирает, почему это часто оптимальный план. Index Scan — как работает B-tree walk, поиск tuples в heap через ctid, MVCC visibility check. Index Only Scan и Visibility Map — почему это самый быстрый вариант. Bitmap Index Scan + Bitmap Heap Scan — как гибрид работает, когда используется, что такое recheck и lossy bitmap. Cost model и как planner оценивает — random_page_cost, seq_page_cost, selectivity estimation. Что такое статистика и как она собирается через ANALYZE. Разберём типичные проблемы: устаревшая статистика, bloat, wrong plan из-за неправильного selectivity, cache effects. Отдельно — как заставить planner использовать нужный план (hints, config tweaks). Composite indexes, partial indexes, expression indexes, covering indexes — когда что применять. Реальные примеры из production с EXPLAIN ANALYZE. И под конец — практическое руководство: как диагностировать плохой план и как его исправить.

## Sequential Scan — читаем таблицу целиком

Начнём с самого простого скана. Sequential Scan — это последовательное чтение всех страниц таблицы от начала до конца. Никаких индексов, никаких shortcut'ов. Прочитали page — применили фильтр — вернули matching строки.

Физически это выглядит так. Таблица `users` хранится в файлах `base/<db_oid>/16385`, `16385.1`, `16385.2`, ... — каждый файл до 1 GB. Файлы разбиты на страницы (pages) по 8 KB. PostgreSQL знает общее количество страниц из `pg_class.relpages`. При Seq Scan он последовательно читает страницу за страницей: page 0, page 1, ..., page N.

Для каждой страницы:

1. Kernel читает страницу с диска (или из OS page cache, если она там) в shared_buffers.
2. Backend получает pin на buffer.
3. Проходит по item pointers (в page header) — они указывают на все tuples на странице.
4. Для каждого tuple:
   - Проверяет MVCC visibility (xmin/xmax vs snapshot).
   - Если visible — применяет WHERE filter.
   - Если passes filter — возвращает.

Ключевой момент: **последовательное чтение с диска — быстро**. SSD/NVMe оптимизирован для sequential access. Даже HDD выдаёт 100-200 MB/s sequential, но 10-100x медленнее random. Читая страницы одну за другой (increasing block number), мы даём kernel возможность делать read-ahead — предзагружать следующие страницы в page cache пока текущая обрабатывается.

Отсюда важный вывод: **Sequential Scan — не всегда медленно**. Если таблица помещается в page cache (обычная ситуация для small/medium tables) или мы читаем большую часть строк (>10-20% таблицы), Seq Scan часто быстрее Index Scan. Потому что Index Scan делает много random I/O — для каждой найденной записи прыгает в heap по ctid, что на HDD — 10 ms latency per lookup. При 1 миллионе rows это часы.

Пример plan:

```
EXPLAIN ANALYZE SELECT * FROM users WHERE status = 'active';

Seq Scan on users (cost=0.00..18334.00 rows=500000 width=64)
                  (actual time=0.012..234.567 rows=498234 loops=1)
   Filter: (status = 'active')
   Rows Removed by Filter: 501766
   Buffers: shared hit=8334
 Planning Time: 0.089 ms
 Execution Time: 245.234 ms
```

Разберём. `Seq Scan on users` — сам оператор. `cost=0.00..18334.00` — planner оценил стоимость (стартовая 0.00, полная 18334). Cost — абстрактная единица, привязанная к `seq_page_cost=1.0` (стоимость чтения одной страницы sequential). У таблицы 8334 страницы, потому cost.total ≈ 18334 (8334 страниц × 1.0 + чтение tuples × 0.01).

`rows=500000` — planner ожидал 500K строк подойдёт под фильтр. `actual time=0.012..234` — реально startup был 0.012 мс, полное выполнение 234 мс. `rows=498234` — реально вернулось. Estimate был очень близок (500K vs 498K) — хорошая статистика.

`Rows Removed by Filter: 501766` — при sequential scan мы прочитали ВСЕ строки таблицы (1 миллион), потом отфильтровали. Половину пришлось выкинуть.

`Buffers: shared hit=8334` — все 8334 страницы были в shared_buffers (cache hit). Если бы часть была на диске — было бы `shared read=X`.

`Execution Time: 245 ms` — реальное время. При Seq Scan на 1 миллион строк с cache hit — 245 мс. Это довольно быстро.

Когда PostgreSQL выбирает Seq Scan:

- **Большая часть таблицы подходит под фильтр** (>10-20% selectivity). Читать через индекс — куча random I/O, суммарно медленнее.
- **Маленькая таблица** (несколько страниц). Даже полный scan — миллисекунды.
- **Нет подходящего индекса**.
- **Планировщик решает, что индекс не поможет** — устаревшая статистика или неправильные оценки.

## Parallel Sequential Scan

С PostgreSQL 9.6+ появился Parallel Seq Scan — таблица читается несколькими worker процессами параллельно. Каждый worker обрабатывает свой diapason страниц.

```
Gather (cost=1000.00..50000.00 rows=500000 width=64)
       (actual time=0.234..123.456 rows=498234 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on users (cost=0.00..15000.00 rows=200000 width=64)
                                  (actual time=0.010..80.234 rows=166078 loops=3)
         Filter: (status = 'active')
```

`loops=3` — потому что 3 processes работали (leader + 2 workers). Каждый прошёл ~1/3 таблицы. Gather собирает результаты.

Настройки:

```
max_worker_processes = 8              # общий лимит worker'ов
max_parallel_workers_per_gather = 2   # per-query лимит
min_parallel_table_scan_size = 8MB    # порог для parallelization
```

Parallel Seq Scan хорош для аналитических запросов на больших таблицах. Для OLTP с маленькими запросами overhead координации не окупается.

## Index Scan — обход B-tree и поиск в heap

Index Scan — принципиально другой подход. Вместо чтения всей таблицы мы:

1. Открываем индекс (обычно B-tree).
2. Спускаемся по дереву от root к leaf.
3. В leaf находим ключи, подходящие под условие.
4. Каждый leaf entry содержит **ctid** — tuple identifier (block number, offset within page).
5. Читаем нужную страницу heap по ctid.
6. Извлекаем tuple по offset.
7. Проверяем MVCC visibility.
8. Возвращаем.

Ключевое: два уровня I/O. Сначала читаем index pages (обычно уже в памяти, они горячие). Потом для каждой найденной записи — random read страницы heap.

Пример plan:

```
EXPLAIN ANALYZE SELECT * FROM users WHERE id = 42;

Index Scan using users_pkey on users (cost=0.29..8.30 rows=1 width=64)
                                     (actual time=0.023..0.024 rows=1 loops=1)
   Index Cond: (id = 42)
   Buffers: shared hit=4
 Execution Time: 0.048 ms
```

Cost `0.29..8.30` — startup 0.29 (спуск по дереву), total 8.30 (плюс чтение одной heap page).

`Index Cond: (id = 42)` — условие, применённое **непосредственно в индексе**. Это важно: index cond находит правильные строки за одно спускание по дереву. Если бы было `Filter: (id = 42)`, это означало бы: сначала прочитать все записи, потом отфильтровать в памяти — гораздо хуже.

`Buffers: shared hit=4` — 4 страницы прочитано (3 уровня B-tree + 1 heap page). Всё в кэше.

`Execution Time: 0.048 ms` — 48 микросекунд. В 5000 раз быстрее Seq Scan для точечного поиска.

Разберём B-tree walk детально.

```
Query: SELECT * FROM users WHERE id = 42
Index: users_pkey (btree on id)

B-tree structure (упрощённо):

                          Root page
                     ┌─────────────┐
                     │  50  |  100 │           ← 2 keys, 3 pointers
                     └──┬───┴──┬───┘
                        │      │
              ┌─────────┘      └─────────┐
              ▼                          ▼
        Internal              Internal
      ┌──────────────┐     ┌──────────────┐
      │ 20 | 35 | 45 │     │ 70 | 85 | 95 │
      └───┬──┬──┬──┬─┘     └──────────────┘
          │  │  │  │
          ▼  ▼  ▼  ▼
         Leaf pages (linked list)
         ┌──────────────────────────────────────────┐
         │ [40→ctid(0,17)][42→ctid(0,25)][45→ctid(0,89)] │
         └──────────────────────────────────────────┘

Спуск:
  1. Root: 42 < 50 → пойти по левому pointer
  2. Internal: 42 между 35 и 45 → пойти по среднему pointer
  3. Leaf: найти key=42 → получить ctid(0,25)
  4. Read heap page 0, offset 25 → tuple найден
  5. MVCC check → visible
  6. Return
```

Каждый уровень B-tree — одна page read (обычно cache hit — index pages горячие). Depth дерева для типичной таблицы: 3-4 уровня даже для миллиардов записей (B-tree широкий, fanout 100-1000).

`ctid` — это (block_number, item_offset). block_number — номер страницы в heap файле. item_offset — indexes item pointer в page header, который указывает на actual tuple.

Важно про MVCC: index entry указывает на конкретный tuple. Если tuple был обновлён — на его же ctid другая versия tuple. Index не хранит xmin/xmax. Значит после обхода index'а мы должны прочитать heap page чтобы проверить visibility. Это и делает Index Scan.

## Index Only Scan — не идём в heap

Идеальная оптимизация. Если все нужные колонки есть в индексе, и мы можем гарантировать что все rows visible — не идём в heap вообще.

```
EXPLAIN ANALYZE SELECT id FROM users WHERE id = 42;

Index Only Scan using users_pkey on users (cost=0.29..4.30 rows=1 width=8)
                                          (actual time=0.015..0.016 rows=1 loops=1)
   Index Cond: (id = 42)
   Heap Fetches: 0
   Buffers: shared hit=3
 Execution Time: 0.028 ms
```

Разница с обычным Index Scan: `Heap Fetches: 0`. Мы прочитали только index pages (3), heap не тронули. Cost меньше (4.30 vs 8.30).

Как работает. Каждая страница heap имеет **Visibility Map (VM)** — небольшой битмап, где 1 бит на страницу. Если бит установлен, значит все tuples на странице visible всем текущим транзакциям (frozen). Такая страница не менялась с момента, когда все существующие транзакции могли её увидеть.

Когда PostgreSQL делает Index Only Scan:

1. Спускается по индексу, находит entry с ctid.
2. Проверяет VM для страницы из ctid.
3. Если бит установлен — tuple точно visible, возвращает данные прямо из index entry.
4. Если бит не установлен — идёт в heap как обычный Index Scan (это `Heap Fetches`).

VACUUM обновляет VM. Frequently updated tables имеют плохую VM, много Heap Fetches → Index Only Scan теряет преимущество.

Условия для Index Only Scan:

- Все нужные колонки должны быть в индексе.
- VM должна показывать, что страницы all-visible.

Для того, чтобы index покрывал query, можно использовать **covering indexes** (PG 11+):

```sql
CREATE INDEX users_email_idx ON users(email) INCLUDE (name);
```

`INCLUDE` добавляет non-key column в leaf page индекса. `email` — search key (используется для равенства/range), `name` — просто payload. Query `SELECT email, name FROM users WHERE email = 'x'` — Index Only Scan.

Практический смысл огромен. Index Only Scan обычно 3-10x быстрее обычного Index Scan для типичных read-heavy workload'ов, потому что не нужно читать random heap pages.

## Bitmap Index Scan + Bitmap Heap Scan — гибрид

Компромисс между Index Scan и Sequential Scan. Полезен когда:

- Условие match'ит много rows (десятки тысяч).
- Rows разбросаны по таблице.
- Index Scan делал бы слишком много random I/O.

Идея: сначала прочитать индекс, но не идти в heap за каждым row. Вместо этого построить **bitmap** — двумерную структуру, где помечены все страницы (и в некоторых случаях offsets), содержащие matching tuples. Потом читать heap **в порядке возрастания страниц** — sequential access, оптимально для kernel read-ahead.

```
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';

Bitmap Heap Scan on orders (cost=1234.56..20000.00 rows=50000 width=128)
                            (actual time=15.234..123.456 rows=48234 loops=1)
   Recheck Cond: (status = 'pending')
   Heap Blocks: exact=3421
   ->  Bitmap Index Scan on orders_status_idx (cost=0.00..1222.34 rows=50000 width=0)
                                              (actual time=13.123..13.123 rows=48234 loops=1)
         Index Cond: (status = 'pending')
   Buffers: shared hit=3560
 Execution Time: 145.678 ms
```

Разберём. Два узла:

**Bitmap Index Scan** — только читает индекс, строит битмап. Каждый matching ctid отмечается в битмапе. Возвращает bitmap как результат (rows=0 в actual, потому что реальные rows ещё не читались из heap).

**Bitmap Heap Scan** — принимает bitmap, читает страницы heap в возрастающем порядке. Для каждой страницы проверяет tuples по битмапу и filter.

`Recheck Cond` — важная деталь. Bitmap может быть **exact** (запомнены точные offsets) или **lossy** (только страницы, без offsets). Lossy когда бит-мап не помещается в work_mem — тогда сохраняются только страницы. При Bitmap Heap Scan мы читаем всю страницу и rechecking условие для каждого tuple на ней. Пример lossy:

```
Bitmap Heap Scan on orders
   Recheck Cond: (status = 'pending')
   Rows Removed by Index Recheck: 12345
   Heap Blocks: exact=100 lossy=3000
```

`Heap Blocks: exact=X lossy=Y` — сколько страниц exact vs lossy. `Rows Removed by Index Recheck` — сколько tuples на lossy pages не подошли под recheck. Fix: увеличить `work_mem` для этой сессии/query.

Когда planner выбирает Bitmap:

- Условие match'ит 1-30% таблицы (грубо).
- Есть подходящий индекс.
- work_mem достаточно для битмапа.

Bitmap Scan также поддерживает **combining bitmaps**:

```sql
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2026-01-01';

Bitmap Heap Scan on orders
   Recheck Cond: ((status = 'pending') AND (created_at > '2026-01-01'))
   ->  BitmapAnd
         ->  Bitmap Index Scan on orders_status_idx
               Index Cond: (status = 'pending')
         ->  Bitmap Index Scan on orders_created_at_idx
               Index Cond: (created_at > '2026-01-01')
```

Планировщик читает два индекса, строит два bitmap, объединяет через AND, потом читает heap. Позволяет использовать несколько single-column индексов вместо composite index.

## Cost model — как planner выбирает

PostgreSQL — cost-based optimizer. Для каждого возможного plan он вычисляет cost (в абстрактных единицах), выбирает cheapest.

Ключевые параметры конфигурации:

- **seq_page_cost = 1.0** — стоимость sequential чтения одной страницы. Reference unit.
- **random_page_cost = 4.0** (default для HDD, 1.1 для SSD) — стоимость random чтения. Отражает медленность random I/O на HDD.
- **cpu_tuple_cost = 0.01** — стоимость обработки одного tuple.
- **cpu_operator_cost = 0.0025** — стоимость одной операции (сравнение, WHERE eval).
- **cpu_index_tuple_cost = 0.005** — обработка одного index entry.
- **effective_cache_size = 4GB** (обычно 50-75% RAM) — оценка размера доступного cache.

Формулы (упрощённо):

**Seq Scan cost**:

```
cost = relpages * seq_page_cost + reltuples * cpu_tuple_cost
```

Для таблицы 8334 страниц, 1 миллион rows:

```
cost = 8334 * 1.0 + 1000000 * 0.01 = 8334 + 10000 = 18334
```

Это то, что мы видели в EXPLAIN.

**Index Scan cost** (для точечного поиска):

```
cost = index_pages_read * random_page_cost 
     + heap_pages_read * random_page_cost 
     + selectivity * reltuples * cpu_tuple_cost
     + index_depth * cpu_index_tuple_cost
```

Для точечного поиска (1 row): index depth ~4, heap 1 page.

```
cost = 4 * 1.1 + 1 * 1.1 + 0.000001 * 1000000 * 0.01 + 4 * 0.005
     ≈ 4.4 + 1.1 + 0.01 + 0.02 = 5.53
```

Индекс намного дешевле для 1 row.

**Пороги переключения**. Для селективности `s` (доля matching rows):

- `s < 0.001` — Index Scan почти всегда выигрывает.
- `0.001 < s < 0.05` — Bitmap Index Scan часто оптимален.
- `s > 0.1-0.3` — Seq Scan.

Точные пороги зависят от `random_page_cost / seq_page_cost` ratio. На SSD (`random_page_cost=1.1`) индексы более competitive для больших selectivity.

## Selectivity — ключевой фактор

Селективность — доля rows, подходящих под условие. Например, `WHERE status = 'active'` — если 50% users active, selectivity = 0.5. `WHERE id = 42` — selectivity ≈ 1/reltuples (уникальный ключ).

PostgreSQL оценивает selectivity через **statistics**, собранную ANALYZE:

- **`pg_statistic`** — статистика per column.
- Хранит: `null_frac`, `n_distinct`, `most_common_values (MCV)`, `most_common_freqs (MCF)`, `histogram_bounds`.

Для `WHERE status = 'active'`:

- Если 'active' есть в MCV → selectivity = MCF['active'].
- Иначе — 1 / n_distinct.

Для `WHERE created_at > '2026-01-01'`:

- Используется histogram_bounds для интерполяции доли rows выше границы.

**Точность статистики критична**. Если статистика показывает `n_distinct = 100`, а реально 100000 — оценки будут неверные, planner выберет плохой plan.

`ANALYZE` обновляет статистику. По умолчанию autovacuum запускает его периодически. Но:

- После большого INSERT — статистика устарела до следующего autovacuum.
- Для больших таблиц ANALYZE читает sample (`default_statistics_target = 100` — 300 × 100 = 30000 rows sample). Sample может быть непредставительным.

**Extended statistics** (PG 10+) — для коррелированных колонок:

```sql
CREATE STATISTICS users_stats (dependencies) ON city, country FROM users;
ANALYZE users;
```

Позволяет planner'у знать, что `city='Astana' AND country='KZ'` не мультипликативно (все Astana в KZ), а не независимо.

## Estimated vs Actual — сравнение в EXPLAIN ANALYZE

Ключевая практика диагностики. Смотрим:

```
Seq Scan on huge_table (cost=... rows=100 width=...) 
                       (actual time=... rows=5000000 loops=1)
   Filter: (status = 'active')
   Rows Removed by Filter: 5000000
```

Optimizer оценил 100 rows подойдут под фильтр. Реально 5 миллионов. **50000× ошибка**. Это значит: устаревшая или неправильная статистика. `ANALYZE huge_table` часто исправит.

Правило: если estimated rows отличается от actual в 10+ раз, план может быть неоптимальный. Optimizer работал вслепую.

Ещё частая проблема:

```
Nested Loop (cost=... rows=100 width=...)
            (actual time=... rows=1000000 loops=1)
   ->  Index Scan on t1 (rows=1)
   ->  Index Scan on t2 (rows=100000 loops=1)
```

Nested Loop оценил ~100 rows на результат. Реально 1 миллион. Nested Loop становится O(N*M) — часы. Правильный план был бы Hash Join.

## Проблемы и антипаттерны

Несколько типичных проблем, которые встречаются в проде.

**Sequential Scan там, где должен быть Index Scan**. Смотришь план запроса `WHERE id = 5` и видишь Seq Scan. Причины:

- Индекса нет. Fix: `CREATE INDEX`.
- Устаревшая статистика заставила planner думать что почти все rows подойдут. Fix: `ANALYZE table`.
- Тип колонки не match'ится с условием. Например колонка `text`, условие `column = 5` → implicit cast ломает использование индекса. Fix: `column = '5'::text` или изменить тип.
- Function/expression в WHERE не соответствует expression index. `WHERE lower(email) = 'x'` — обычный `INDEX ON email` не подходит, нужен `INDEX ON lower(email)`.
- Условие содержит LIKE с wildcard в начале: `WHERE name LIKE '%berik'`. B-tree не помогает. Fix: `pg_trgm` GIN index.

**Nested Loop на большом результате**. Внутренний узел `rows=1000000, loops=100000` — 100 миллиардов операций. Optimizer выбрал Nested Loop, оценив что строк будет мало. Оценка ошиблась. Fix: часто ANALYZE помогает. Если нет — можно попробовать `SET enable_nestloop = off` временно, посмотреть какой план получится (обычно Hash Join), и оптимизировать через переписывание запроса.

**Rows Removed by Filter огромное**. Индекс есть, но фильтр читает много и выбрасывает. Значит index покрывает только часть условий. Fix: составной индекс, включающий все условия WHERE. Например `WHERE status = 'active' AND created_at > '2026-01-01'` — вместо индекса только на `status`, сделать composite `(status, created_at)`.

**Bitmap Heap Scan с Recheck**. Bitmap lossy (не хватило памяти запомнить конкретные строки). Технически работает, но каждую страницу приходится перепроверять. Fix: увеличить work_mem.

**Estimated vs actual rows отличается в тысячи раз**. Optimizer работает вслепую. Fix: `ANALYZE table`, extended statistics для коррелированных колонок.

**JIT compilation на маленьких запросах**. Иногда видишь `JIT: Functions: 5, ...` на быстром запросе. JIT добавил overhead компиляции, сам запрос быстрее не стал. Fix: увеличить `jit_above_cost` чтобы JIT включался только для тяжёлых запросов.

## Как заставить planner использовать нужный план

PostgreSQL не поддерживает hints в стиле Oracle (`/*+ INDEX(t idx) */`). Есть только косвенные способы влиять на выбор плана.

**Session-level flags** — отключить конкретные типы планов:

```sql
SET enable_seqscan = off;
SET enable_indexscan = off;
SET enable_bitmapscan = off;
SET enable_hashjoin = off;
SET enable_nestloop = off;
```

Осторожно: отключение — грубая мера. Например `enable_seqscan=off` не запрещает Seq Scan (planner сможет выбрать его если нет альтернатив), но сильно penalizes cost. Полезно для дебага — «а какой был бы план без Seq Scan?».

**Изменение параметров cost**:

```sql
SET random_page_cost = 1.5;  -- ближе к SSD
SET seq_page_cost = 1.0;
```

**`pg_hint_plan` extension** — если реально нужны hints:

```sql
/*+ IndexScan(users users_pkey) */
SELECT * FROM users WHERE id = 42;
```

Redkiy use case — обычно правильнее исправить статистику или запрос.

**Переписать запрос**. Часто самый эффективный способ:

- Разбить сложный запрос на CTE или temp tables.
- Изменить условие так, чтобы planner легче видел selectivity.
- Materialized view для сложных агрегаций.

**Увеличить statistics target**:

```sql
ALTER TABLE users ALTER COLUMN email SET STATISTICS 1000;
ANALYZE users;
```

Больший target = точнее histogram = лучше оценки для важных колонок.

## Типы индексов — когда что применять

Быстрый обзор с фокусом на использовании в scans.

**B-tree** (default). Универсальный. Поддерживает `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN`, `IS NULL`, `LIKE 'prefix%'`. Для 90% случаев. Именно этот тип используется Index Scan, Index Only Scan, Bitmap Index Scan по умолчанию.

**Composite (multi-column) B-tree**:

```sql
CREATE INDEX users_status_created_idx ON users(status, created_at);
```

Работает для условий:
- `WHERE status = 'x'` — да.
- `WHERE status = 'x' AND created_at > '...'` — да, самый эффективный.
- `WHERE created_at > '...'` — нет (лидирующая колонка не задана).

**Правило**: колонки в индексе — в порядке от наиболее селективных к менее. И equality perto range: `(status, created_at)` лучше чем `(created_at, status)`, если фильтр — `status = 'x' AND created_at > '...'`.

**Partial index**:

```sql
CREATE INDEX users_active_email_idx ON users(email) WHERE status = 'active';
```

Индекс только на подмножестве rows. Меньше по размеру, быстрее. Используется только когда query также содержит условие `status = 'active'`.

Идеально когда:
- Большинство queries фильтруют по определённому условию (`is_deleted = false`).
- Хочешь index только на "живые" rows.

**Expression index**:

```sql
CREATE INDEX users_lower_email_idx ON users(lower(email));
```

Используется когда query имеет `WHERE lower(email) = 'x'`. Позволяет case-insensitive поиск через индекс.

**Covering (INCLUDE) index** (PG 11+):

```sql
CREATE INDEX users_email_idx ON users(email) INCLUDE (name, phone);
```

`email` — search key. `name`, `phone` — просто в leaf, не участвуют в поиске. Query `SELECT name, phone FROM users WHERE email = 'x'` — Index Only Scan.

**GIN** — для inverted-style indexes:
- JSONB: `WHERE data @> '{"key": "value"}'`.
- Arrays: `WHERE tags && ARRAY['tag1']`.
- Full-text search: `WHERE text @@ 'query'`.

**GiST** — для geometry, ranges, similarity.

**BRIN** — для очень больших таблиц с naturally-sorted data (time-series).

**Hash** — только equality, редко нужен (B-tree обычно достаточно).

## Пример из проекта КНП

Возьмём типичную таблицу — `transactions` (500M rows):

```sql
CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,
    taxpayer_id BIGINT NOT NULL,
    amount NUMERIC(15,2),
    status VARCHAR(20),
    created_at TIMESTAMPTZ,
    tax_period_id INT
);

-- Индексы:
CREATE INDEX transactions_taxpayer_id_idx ON transactions(taxpayer_id);
CREATE INDEX transactions_created_at_idx ON transactions(created_at);
CREATE INDEX transactions_taxpayer_period_idx 
    ON transactions(taxpayer_id, tax_period_id);
```

Запрос: получить транзакции налогоплательщика за 2025 год.

```sql
SELECT * FROM transactions 
WHERE taxpayer_id = 12345 
  AND created_at BETWEEN '2025-01-01' AND '2025-12-31'
ORDER BY created_at;
```

Возможные планы:

1. **Index Scan on `transactions_taxpayer_id_idx`** — найти все transactions для taxpayer 12345, отфильтровать по date, сортировать. Если у налогоплательщика 10K транзакций всего, 3K за 2025 — 10K index reads + 3K heap reads + sort.

2. **Bitmap Index Scan on `transactions_created_at_idx`** — найти все transactions за 2025 год (миллионы), отфильтровать по taxpayer. Слишком дорого — не выберет.

3. **Index Scan on `transactions_taxpayer_period_idx`** — не подходит, `tax_period_id` не тот, что `created_at`.

Planner выберет вариант 1. Но если у налогоплательщика миллион транзакций, лучше был бы composite `(taxpayer_id, created_at)`:

```sql
CREATE INDEX transactions_taxpayer_created_idx 
    ON transactions(taxpayer_id, created_at);
```

Теперь Index Scan работает по обоим условиям сразу:

```
Index Scan using transactions_taxpayer_created_idx on transactions
   Index Cond: ((taxpayer_id = 12345) 
                AND (created_at >= '2025-01-01')
                AND (created_at <= '2025-12-31'))
   Buffers: shared hit=45
```

`Index Cond` теперь match'ит обе части. Меньше tuples читается из индекса, меньше heap fetches, сортировка free (индекс уже sorted).

## Практическое руководство — как оптимизировать запрос

Пошагово, когда получаешь медленный запрос.

**Шаг 1**: EXPLAIN ANALYZE с BUFFERS.

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT ...;
```

**Шаг 2**: Найти самый дорогой узел. Смотри на `actual time` (не cost — это оценка), обращая внимание на largest total time.

**Шаг 3**: Проверить estimated vs actual rows на этом узле. Если сильное расхождение (>10x) — статистика проблема.

**Шаг 4**: Если Seq Scan там, где ожидал Index Scan — проверь:
- Индекс есть? `\d table` в psql.
- ANALYZE недавно? `SELECT last_analyze FROM pg_stat_user_tables WHERE relname='table'`.
- Условие match'ится с индексом (типы, expressions)?
- Селективность действительно низкая? Может, 90% строк подходят и Seq Scan действительно оптимален?

**Шаг 5**: Если Index Scan медленный — проверь:
- Много Heap Fetches? Может, стоит INCLUDE column.
- Много Rows Removed by Filter? Нужен composite index.
- Random I/O не помещается в cache? Buffers hit/read ratio.

**Шаг 6**: Если Bitmap Scan имеет много lossy blocks и recheck — увеличь work_mem.

**Шаг 7**: Если Nested Loop с большими loops — вероятно wrong plan. Проверь statistics, попробуй `SET enable_nestloop = off` временно.

**Шаг 8**: Если ничего не помогло — попробуй переписать запрос. Иногда простая переформулировка кардинально меняет plan.

## Заключение

Sequential Scan и Index Scan — два фундаментальных способа доступа к данным в PostgreSQL. Правильный выбор между ними — не «Index всегда лучше», а компромисс, зависящий от селективности, размера таблицы, доступности cache, и десятка других факторов. PostgreSQL cost-based optimizer старается выбрать лучший, опираясь на статистику. Когда он ошибается — обычно виновата устаревшая статистика или неоптимальный индекс.

**Sequential Scan** — читаем всю таблицу подряд. Быстро для sequential access. Оптимально когда selectivity высокая (>10-20%) или таблица маленькая. Parallel Seq Scan для больших аналитических запросов.

**Index Scan** — обход B-tree + heap lookup по ctid. Быстро для точечных поисков. Random I/O — узкое место на HDD, менее критично на SSD.

**Index Only Scan** — не идём в heap благодаря Visibility Map. Самый быстрый вариант, если все нужные колонки в индексе. VACUUM должен работать нормально, чтобы VM была актуальна.

**Bitmap Index Scan + Bitmap Heap Scan** — гибрид. Читает индекс, строит битмап страниц, читает heap в порядке страниц (sequential). Оптимально для средней селективности (1-30%). Расширение — BitmapAnd/BitmapOr для комбинации нескольких индексов.

**Cost model** — planner выбирает cheapest по формулам с параметрами `seq_page_cost`, `random_page_cost`, `cpu_*_cost`, `effective_cache_size`. Селективность оценивается через `pg_statistic`, обновляемый ANALYZE.

**Типичные проблемы**: устаревшая статистика (fix — ANALYZE), неправильные типы (fix — cast или изменить тип), missing composite index (fix — CREATE), lossy bitmap (fix — increase work_mem), wrong plan из-за неточных оценок (fix — extended statistics или переписать запрос).

**Индексы** — не бесплатны. Каждый замедляет write, занимает disk. Composite index лучше, чем несколько single-column для типичных запросов. Partial и expression indexes — для специфичных случаев. INCLUDE column для Index Only Scan.

Практический совет: возьми свою БД в проде, включи pg_stat_statements, выбери топ-20 запросов по total_exec_time, прогоняй каждый через EXPLAIN (ANALYZE, BUFFERS). Смотри на largest actual time узел. Проверяй estimated vs actual. Если plan использует Seq Scan на большой таблице — попробуй добавить индекс, проверь снова. Если Index Scan, но много Rows Removed by Filter — попробуй composite. Каждый разобранный случай добавляет интуицию, которая приходит только через практику. Через 10-20 таких оптимизаций ты будешь предсказывать plan за секунду, глядя на запрос.

Дальше — экспериментируй в staging: создавай индексы, наблюдай эффект на конкретные queries, замеряй время до/после. Только через эксперименты приходит настоящее понимание — какие индексы окупаются, а какие только тормозят writes без пользы.
