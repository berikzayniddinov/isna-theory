# 105. Autovacuum tuning для больших таблиц — deep-dive

## Зачем это знать

Autovacuum в PostgreSQL — фоновая работа, которая должна происходить сама, незаметно, никого не беспокоя. И на маленьких таблицах — так и есть. Настройки defaults, никакого внимания, всё работает. Но как только таблица переваливает 10-50 миллионов строк, defaults начинают ломаться. К 100 миллионам они ломаются гарантированно, и приложение начинает страдать: медленные запросы, bloat, разросшийся диск, внезапные `ERROR: MultiXactId is too old`, freeze storms, ставящие БД раком на 30 минут.

Это одна из тех проблем, которые не видны в разработке. На локальной БД с 1000 строк всё летает. На staging с 10 миллионами — тоже норм. И потом на проде через 6 месяцев после запуска сервиса — БД начинает деградировать. VACUUM не поспевает, dead tuples накапливаются, размер таблицы растёт в 3-5 раз от её реального содержимого, `EXPLAIN` показывает Sequential Scan там, где раньше был Index Scan. И DBA пожимает плечами: «странно, autovacuum же default'ный».

Есть три причины, почему это нужно понимать глубоко. Первая — defaults настроены для таблиц 1990-х годов, когда 10 МБ считалось «большим». Параметры типа `autovacuum_vacuum_scale_factor = 0.2` означают «начинать VACUUM когда 20% таблицы изменилось». Для таблицы в 1000 строк это 200 строк — норм. Для таблицы в 100 миллионов — это **20 миллионов строк**, которые должны стать dead прежде чем autovacuum начнёт что-то делать. К тому моменту таблица уже bloated до неузнаваемости.

Вторая — VACUUM не только про место. Он про **XID wraparound** — фундаментальную вещь PostgreSQL, которая может убить БД целиком, если её игнорировать. Каждая транзакция получает 32-битный ID. При приближении к 2 миллиардам — если старые tuples не «заморожены» (frozen), возникает риск того, что новые транзакции будут видеть старые данные как «будущие», нарушая MVCC. PostgreSQL спасается тем, что при приближении к wraparound-lim переходит в defensive mode — начинает aggressive freeze VACUUM, блокирующий всё. Инцидент 2017 года у крупного онлайн-магазина: `VACUUM to prevent wraparound` работал на таблице несколько дней, полностью блокируя writes.

Третья — тонкое место продовой эксплуатации. `autovacuum_vacuum_cost_delay`, `autovacuum_naptime`, per-table settings через `ALTER TABLE` — все эти параметры имеют смысл только когда понимаешь, что реально делает VACUUM. Тюнить наугад — либо перегружать систему I/O операциями vacuum'а (медленные user queries), либо не поспевать за dead tuples (bloat растёт).

Мы разберём, что такое VACUUM физически: как он проходит таблицу, находит dead tuples, обновляет visibility map, помечает free space. Что такое **bloat** и как его измерить: pgstattuple, pg_stat_user_tables. Autovacuum internals: launcher, workers, как выбираются таблицы для обработки. Все ключевые параметры: `scale_factor`, `threshold`, `cost_delay`, `cost_limit`, `naptime`, `max_workers`. Почему defaults плохи для больших таблиц и как их изменить per-table через `ALTER TABLE`. Отдельная секция про XID wraparound и freeze VACUUM — почему это критично и как избежать «vacuum to prevent wraparound». VACUUM vs VACUUM FULL vs pg_repack — когда что использовать. Диагностика в проде: как понять, что autovacuum не поспевает, как читать `pg_stat_progress_vacuum`. И под конец — практическое руководство для КНП: реальные настройки для таблиц 100M-1B строк.

## VACUUM — что это делает

Начнём с фундамента. PostgreSQL использует MVCC (Multi-Version Concurrency Control): при UPDATE или DELETE старая версия row не удаляется сразу, а помечается как «удалённая транзакцией X». Пока какая-то активная транзакция может её видеть — она остаётся в файле таблицы.

Пример. Таблица `payments`, состояние в момент времени:

```
Page 0:
  Tuple 1: xmin=100, xmax=0     ← active row (создана tx 100, не удалена)
  Tuple 2: xmin=101, xmax=105   ← DEAD (создана tx 101, удалена tx 105)
  Tuple 3: xmin=105, xmax=0     ← active (создана tx 105 при UPDATE tuple 2)
  Tuple 4: xmin=102, xmax=0     ← active
```

Tuple 2 занимает место на диске, но не виден никому (все транзакции с snapshot после commit tx 105 видят Tuple 3, не Tuple 2). Это **dead tuple**. Пока VACUUM его не почистит — он остаётся.

Что делает VACUUM для таблицы:

1. **Проходит все страницы** таблицы (или подмножество, если есть Visibility Map).
2. **Для каждой страницы**:
   - Находит dead tuples (xmax < oldest_active_xid).
   - Удаляет их item pointers, освобождает место на странице.
   - Обновляет **Free Space Map** — говорит: «на этой странице теперь X свободного места».
   - Если все tuples на странице теперь all-visible → обновляет **Visibility Map** (для Index Only Scan).
3. **Проходит все индексы** таблицы, удаляет entries, указывающие на удалённые tuples.
4. **Обновляет `pg_class`**: `reltuples`, `relpages` (примерная статистика для planner).
5. **Не возвращает место OS**. VACUUM помечает страницы как reusable, но не сокращает файл. Для реального возврата места нужен VACUUM FULL или pg_repack.

Что делает VACUUM ANALYZE дополнительно:

- Пересчитывает статистику для planner (`pg_statistic`).
- MCV, histogram, n_distinct, correlation.

По сути, VACUUM — это операция «gc для таблицы». Без него старые версии накапливаются, файл разбухает (bloat), запросы замедляются.

## Bloat — что это и как его увидеть

Bloat — это разница между **логическим** размером данных и **физическим** размером файла. Ваша таблица содержит 10 миллионов live rows, но файл на диске — 15 GB, хотя реально нужно 5 GB. 10 GB — bloat.

Причины bloat:

- Dead tuples, не собранные VACUUM.
- Slack space внутри страниц (страницы никогда не 100% заполнены — `FILLFACTOR`).
- Removed indexes, старые index entries.

Как измерить.

**Простой подход через pg_stat_user_tables**:

```sql
SELECT 
    schemaname, tablename,
    n_live_tup, n_dead_tup,
    n_dead_tup::float / NULLIF(n_live_tup + n_dead_tup, 0) AS dead_ratio,
    last_vacuum, last_autovacuum, last_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

`n_dead_tup / (n_dead_tup + n_live_tup)` — доля dead tuples. Если > 0.2 (20%) — таблица bloated. Норма — < 0.1.

`last_autovacuum` — когда autovacuum последний раз проходил таблицу. Если давно — autovacuum не справляется или пропускает.

**Более точный подход через pgstattuple extension**:

```sql
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstattuple('payments');
```

Возвращает детальные метрики:

```
table_len         | 15000000000  ← физический размер (15 GB)
tuple_count       | 10000000     ← live tuples
tuple_len         | 5000000000   ← размер live tuples (5 GB)
tuple_percent     | 33.33        ← только 33% данных полезны
dead_tuple_count  | 5000000
dead_tuple_len    | 2500000000
dead_tuple_percent| 16.67
free_percent      | 40.0         ← остальное — free space внутри страниц
```

**Практический сценарий**. Таблица `transactions` в КНП, 300 миллионов rows после года работы. Мониторинг показывает:

```
tuple_percent      = 22
dead_tuple_percent = 35
free_percent       = 43
```

Из 90 GB на диске только 20 GB содержат живые данные. 30 GB — dead tuples, 40 GB — slack space. Sequential Scan этой таблицы проходит 90 GB вместо 20 GB — в 4.5x медленнее, чем должно быть. Index Scan лучше, но fetching heap pages тоже страдает.

## Autovacuum — как работает

Autovacuum — фоновая инфраструктура PostgreSQL для автоматического VACUUM'а. Не отдельный процесс, а group of processes:

- **autovacuum launcher** — master process, запускается когда `autovacuum = on` (default).
- **autovacuum workers** — рабочие процессы, до `autovacuum_max_workers` (default 3) одновременно.

Как работает launcher:

1. Каждые `autovacuum_naptime` (default **1 min**) просыпается.
2. Читает `pg_stat_all_tables` — какие таблицы требуют внимания.
3. Для каждой таблицы вычисляет:
   ```
   vacuum_threshold = autovacuum_vacuum_threshold                     (default 50)
                    + autovacuum_vacuum_scale_factor * reltuples      (default 0.2)
   ```
   Если `n_dead_tup > vacuum_threshold` → таблица нуждается в VACUUM.
   
   Аналогично для ANALYZE:
   ```
   analyze_threshold = autovacuum_analyze_threshold                   (default 50)
                     + autovacuum_analyze_scale_factor * reltuples    (default 0.1)
   ```
4. Стартует worker для обработки этой таблицы (если есть свободный слот).

Worker выполняет VACUUM в фоне. Внутри — cost-based throttling:

- Каждая операция VACUUM имеет «cost»:
  - `vacuum_cost_page_hit = 1` (страница в shared_buffers).
  - `vacuum_cost_page_miss = 2` (страница из OS cache).
  - `vacuum_cost_page_dirty = 20` (dirty page, нужно записать).
- Накапливается `vacuum_cost_limit` (default 200 для manual, автовакуум использует свой `autovacuum_vacuum_cost_limit`, default -1 = использовать general).
- Когда лимит превышен → sleep `autovacuum_vacuum_cost_delay` (default 2ms в PG 12+, 20ms до этого).

Смысл — не насытить I/O. Autovacuum должен работать в фоне, не влиять на user queries.

## Ключевые параметры и что они значат

Разберём все параметры, значимые для autovacuum, с их defaults и рекомендациями.

**Глобальные (postgresql.conf)**:

- **`autovacuum = on`** — включён/выключен. НИКОГДА не выключайте на prod. Это одна из самых частых причин production disasters.

- **`autovacuum_naptime = 1min`** — интервал проверки launcher'ом. Оставить default.

- **`autovacuum_max_workers = 3`** — сколько workers параллельно. Для больших БД с многими таблицами — увеличить до 4-6.

- **`autovacuum_vacuum_threshold = 50`** — минимальный порог dead tuples для vacuum.

- **`autovacuum_vacuum_scale_factor = 0.2`** — доля таблицы, которая должна стать dead для vacuum. **Проблема для больших таблиц** (см. ниже).

- **`autovacuum_analyze_threshold = 50`** — минимальный порог.

- **`autovacuum_analyze_scale_factor = 0.1`** — 10% изменений триггерит ANALYZE.

- **`autovacuum_vacuum_cost_delay = 2ms`** — sleep между chunks vacuum работы.

- **`autovacuum_vacuum_cost_limit = -1`** — сколько cost units обработать между sleeps. -1 = использовать `vacuum_cost_limit` (default 200).

- **`autovacuum_freeze_max_age = 200 million`** — при какой age от XID wraparound начинать forced freeze vacuum. Критично, см. секцию про wraparound.

- **`vacuum_freeze_min_age = 50 million`** — сколько XID должно пройти, прежде чем tuple может быть заморожен.

- **`vacuum_freeze_table_age = 150 million`** — при какой age VACUUM автоматически будет aggressive.

**Per-table settings** (ALTER TABLE):

```sql
ALTER TABLE payments SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_vacuum_cost_delay = 1,
    autovacuum_vacuum_cost_limit = 1000,
    autovacuum_freeze_max_age = 400000000
);
```

Все параметры имеют per-table версию через `SET (option_name = value)`. Именно per-table tuning — ключевое для больших таблиц.

## Почему defaults плохи для больших таблиц

Возьмём таблицу `transactions` в КНП — 100 миллионов rows.

С default `autovacuum_vacuum_scale_factor = 0.2`:

```
vacuum_threshold = 50 + 0.2 * 100_000_000 = 20_000_050
```

Autovacuum начнёт работу когда накопится **20 миллионов dead tuples**.

При типичном UPDATE-heavy workload (например, обновление статусов транзакций) — за день накапливается 1-5 миллионов UPDATE'ов = столько же dead tuples. Autovacuum сработает через 4-20 дней. За это время:

- Bloat вырос на 20 миллионов dead tuples ≈ 2-5 GB extra диск.
- Index scans замедлились (каждый лишний heap fetch — miss'а больше).
- Sequential Scan читает лишние 5 GB.
- Query planner использует устаревшую reltuples для оценок.

И даже когда autovacuum стартует — он должен пройти всю таблицу. На 100 миллионах rows это часы. С default `cost_delay = 2ms` и `cost_limit = 200` — autovacuum работает в очень медленном темпе, чтобы не мешать. На больших таблицах может занять **дни**.

Итог: с defaults, большая таблица никогда не находится в «здоровом» состоянии. Bloat постоянно растёт быстрее, чем autovacuum успевает.

Решение — **уменьшить scale_factor для больших таблиц**:

```sql
ALTER TABLE transactions SET (autovacuum_vacuum_scale_factor = 0.01);
```

Теперь порог = 50 + 0.01 × 100M = 1M dead tuples. Autovacuum триггерится раньше, меньше bloat.

Плюс **увеличить cost_limit** чтобы vacuum шёл быстрее:

```sql
ALTER TABLE transactions SET (autovacuum_vacuum_cost_limit = 1000);
```

В 5x быстрее чем default.

## XID wraparound — самая большая опасность

PostgreSQL использует 32-битные transaction IDs (XID). Каждая транзакция получает свой XID. Через **2^32 ≈ 4 миллиарда транзакций** XID переполняется и начинается заново.

Проблема: если tuple имеет `xmin = 100`, а текущий XID сейчас 4 миллиарда, а потом станет 100 после wraparound — как отличить старый tuple от «будущего»? Никак. MVCC ломается.

Решение PostgreSQL — **freeze**. Когда tuple становится «достаточно старым» (xmin младше N от текущего XID), VACUUM меняет xmin на специальную константу `FrozenTransactionId` (2). Это говорит: «tuple был committed так давно, что видно всем текущим транзакциям, что бы там ни было с XID».

Параметры:

- **`vacuum_freeze_min_age = 50M`** — минимальный age чтобы tuple можно было freeze.
- **`autovacuum_freeze_max_age = 200M`** — максимальный age до **forced** VACUUM. Если таблица подходит к 200M — PostgreSQL стартует **VACUUM to prevent wraparound**, который **невозможно отменить** и работает пока не закончит.

Симптомы приближения к wraparound:

```sql
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY xid_age DESC LIMIT 20;
```

`age(relfrozenxid)` — сколько XID прошло с последнего freeze. Если приближается к 200M — на носу forced vacuum.

Если age > 2 миллиарда — БД начинает выдавать warnings и в конце концов **отключит writes** (`database is not accepting commands to avoid wraparound data loss`). Единственный выход — offline single-user mode VACUUM.

История. Sentry (сервис мониторинга ошибок) в 2015 году получил `wraparound` инцидент. Один из shard'ов остановил writes. Восстановление заняло 32 часа single-user VACUUM. Полная post-mortem — рекомендую погуглить, поучительно.

Как избежать:

1. **Autovacuum должен работать**. Никогда не отключайте.
2. **Мониторинг** `age(relfrozenxid)` — alert если > 100M.
3. **Уменьшить `autovacuum_freeze_max_age`** для важных таблиц если приближается — стартовать freeze раньше, когда нагрузка ниже:

```sql
ALTER TABLE transactions SET (autovacuum_freeze_max_age = 100000000);
```

4. **Batch-heavy workflows** — если есть job, генерирующий миллионы транзакций (bulk import) — потом сразу запустить `VACUUM (FREEZE)` вручную.

## VACUUM vs VACUUM FULL vs pg_repack

Есть три варианта для очистки bloat.

**VACUUM** (то же что делает autovacuum):

- Не блокирует запросы (SHARE UPDATE EXCLUSIVE lock, совместим с DML).
- Помечает dead tuples как reusable.
- **Не возвращает место OS**. Файл не сокращается.
- Быстро. Обычная операция.

**VACUUM FULL**:

- Полная перезапись таблицы. Создаёт новый файл, копирует все live tuples, удаляет старый.
- **ACCESS EXCLUSIVE lock** — блокирует ВСЁ на время работы.
- На таблице в 100M rows — часы блокировки. **Никогда в проде без planned downtime**.
- Возвращает место OS. Файл ужимается до реального размера.

**pg_repack** (extension):

- Онлайн-эквивалент VACUUM FULL. Работает без блокировки читателей и в основном без блокировки писателей.
- Механика: создаёт shadow table, копирует данные, использует триггеры для отслеживания изменений во время процесса, потом атомарно свопит таблицы.
- Требует ~2x disk space во время работы.
- Медленнее VACUUM FULL, но не блокирует.

**Выбор**:

- **Регулярно** — autovacuum с правильными настройками.
- **Bloat обнаружен, нужно сжать** — pg_repack, всегда.
- **VACUUM FULL** — только на maintenance windows, только для маленьких таблиц.

## Practical tuning для КНП — таблица на 100M+ rows

Возьмём таблицу `payments` в КНП. Профиль:

- 300 миллионов rows.
- ~5 миллионов UPDATE'ов в день (обновление статуса платежей).
- ~500 тысяч INSERT'ов в день.
- ~100 тысяч DELETE'ов в день (архивирование).

Итого dead tuples в день: 5M (UPDATE даёт dead + new) + 100K (DELETE) = **5.1 миллиона**.

С default autovacuum_vacuum_scale_factor=0.2:
- Threshold = 60M dead tuples.
- Autovacuum трогает таблицу раз в **12 дней**.
- К моменту старта — 60M dead tuples, taxbase раздулся на 30 GB.

Рекомендуемая настройка:

```sql
ALTER TABLE payments SET (
    -- Триггерить autovacuum когда 0.5% таблицы стали dead (1.5M rows)
    autovacuum_vacuum_scale_factor = 0.005,
    
    -- Триггерить ANALYZE аналогично (для актуальной статистики planner'а)
    autovacuum_analyze_scale_factor = 0.005,
    
    -- Позволить vacuum работать быстрее (без throttling)
    autovacuum_vacuum_cost_delay = 1,     -- было 2ms
    autovacuum_vacuum_cost_limit = 2000,  -- было 200
    
    -- Freeze раньше, чтобы не accumulated XID pressure
    autovacuum_freeze_max_age = 100000000  -- было 200M
);
```

Результат:

- Autovacuum триггерится когда 1.5M dead tuples накопились = каждые 6-8 часов.
- Bloat стабилизируется на 5-10% instead of 30-40%.
- Table size grows linearly with data, а не exponentially.
- Query planner имеет актуальную статистику через частые ANALYZE.
- XID wraparound не проблема.

## Мониторинг autovacuum в проде

Ключевые запросы для отслеживания.

**1. Bloat overview**:

```sql
SELECT 
    schemaname || '.' || relname AS table,
    pg_size_pretty(pg_total_relation_size(relid)) AS size,
    n_live_tup, n_dead_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0), 3) AS dead_ratio,
    last_autovacuum,
    autovacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

Alert: `dead_ratio > 0.2` на важных таблицах.

**2. XID wraparound risk**:

```sql
SELECT 
    relname,
    age(relfrozenxid) AS xid_age,
    round(100 * age(relfrozenxid)::numeric / 
          current_setting('autovacuum_freeze_max_age')::int, 1) AS pct_to_forced_freeze
FROM pg_class
WHERE relkind = 'r'
  AND relfrozenxid <> 0
ORDER BY xid_age DESC
LIMIT 20;
```

Alert: `pct_to_forced_freeze > 60`.

**3. Autovacuum activity in progress**:

```sql
SELECT 
    p.pid, p.datname, p.relid::regclass AS table,
    p.phase,                                       -- какая фаза
    p.heap_blks_scanned, p.heap_blks_total,        -- сколько уже прошло
    round(100 * p.heap_blks_scanned::numeric / 
          NULLIF(p.heap_blks_total, 0), 2) AS pct_done,
    a.query,
    now() - a.xact_start AS duration
FROM pg_stat_progress_vacuum p
JOIN pg_stat_activity a ON a.pid = p.pid;
```

Показывает активные VACUUM'ы, прогресс, длительность. Если VACUUM работает часами — check settings.

**4. Long-running autovacuum блокирует что-то?**:

```sql
SELECT 
    blocked.pid, blocked.query,
    blocking.pid, blocking.query, 
    blocking.wait_event_type, blocking.wait_event
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocking.query LIKE 'autovacuum:%';
```

Обычно autovacuum совместим с DML, но если делает `VACUUM (FULL)` или aggressive freeze — может блокировать.

**5. Prometheus / postgres_exporter метрики**:

- `pg_stat_user_tables_n_dead_tup`
- `pg_stat_user_tables_last_autovacuum`
- `pg_stat_activity_max_tx_duration{state="active",datname=""}`

Grafana dashboard "PostgreSQL Vacuum" — визуализация.

## Ручные VACUUM в проде — когда

Обычно autovacuum должен справляться. Но иногда — ручные VACUUM полезны.

**После bulk import**:

```sql
COPY payments FROM '/data/2026-Q1.csv';
-- Импортировали 50M rows

VACUUM ANALYZE payments;
-- Обновляем статистику сразу, не ждём autovacuum
```

Autovacuum видит `n_ins` не так активно как `n_dead`. Для INSERT-heavy workloads — вручную ANALYZE после больших вставок.

**Перед крупным release / migration**:

```sql
-- Перед миграцией схемы — убедиться что таблица «здорова»
VACUUM (VERBOSE, ANALYZE) transactions;
```

Уменьшает bloat, обновляет статистику, миграция будет предсказуемее.

**Aggressive freeze для приближающегося wraparound**:

```sql
-- Age приближается к 200M — форсировать freeze
VACUUM (FREEZE, VERBOSE) transactions;
```

Опасно на проде — блокирует. Только в maintenance window.

**Preventive VACUUM night jobs** — некоторые DBA настраивают nightly cron:

```bash
0 3 * * * psql -d knp -c "VACUUM ANALYZE transactions;"
```

Contentious — если autovacuum правильно настроен, cron не нужен. Но для safety net в enterprise часто оставляют.

## Anti-patterns и типичные ошибки

**1. Отключение autovacuum глобально**. `autovacuum = off` в postgresql.conf. Причина обычно — «autovacuum создаёт нагрузку в busy hours». Решение — cost_delay, наплатить.

Отключить autovacuum = гарантированный wraparound через 6-12 месяцев + огромный bloat.

**2. `VACUUM FULL` в prime time**. Кто-то видит bloated таблицу, лечит `VACUUM FULL`. ACCESS EXCLUSIVE lock — приложение падает. Используйте pg_repack.

**3. `SET autovacuum = off` для конкретной таблицы**. Обычно рекомендуется как оптимизация для bulk load. НО — если забыть включить обратно, wraparound подкрадётся незаметно.

**4. Слишком aggressive tuning — cost_limit = 10000**. Autovacuum съедает всё I/O, user queries тормозят. Найдите баланс.

**5. Игнорирование Freezing**. Настроили vacuum для регулярного cleanup, но не думают о freeze. XID wraparound через год.

**6. VACUUM manual, забывая ANALYZE**. `VACUUM users` без ANALYZE — очистили dead tuples, но статистика planner устарела. Всегда `VACUUM ANALYZE`.

**7. Не мониторят n_dead_tup**. Bloat растёт незаметно.

## Как найти таблицы, требующие attention

Скрипт для аудита БД:

```sql
WITH stats AS (
    SELECT 
        schemaname || '.' || relname AS table_name,
        pg_total_relation_size(relid) AS total_bytes,
        n_live_tup, n_dead_tup,
        n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) AS dead_ratio,
        last_autovacuum,
        EXTRACT(EPOCH FROM (now() - last_autovacuum)) / 3600 AS hours_since_autovacuum,
        (SELECT reloptions FROM pg_class WHERE oid = relid) AS reloptions
    FROM pg_stat_user_tables
)
SELECT 
    table_name,
    pg_size_pretty(total_bytes) AS size,
    n_live_tup, n_dead_tup,
    ROUND(dead_ratio * 100, 1) AS dead_pct,
    ROUND(hours_since_autovacuum, 1) AS hrs_since_vac,
    CASE 
        WHEN dead_ratio > 0.3 THEN '🔴 HEAVY BLOAT'
        WHEN dead_ratio > 0.15 THEN '🟡 MODERATE BLOAT'
        WHEN hours_since_autovacuum > 48 THEN '🟡 STALE STATS'
        ELSE '🟢 OK'
    END AS status,
    reloptions
FROM stats
WHERE total_bytes > 100 * 1024 * 1024  -- таблицы > 100 MB
ORDER BY dead_ratio DESC, total_bytes DESC;
```

Прогоняйте раз в неделю. Список таблиц с bloat > 15% — кандидаты на per-table tuning.

## Специальные случаи

**Insert-only таблицы** (audit logs, event stores). Нет UPDATE/DELETE → нет dead tuples → autovacuum думает «нечего делать». Но XID пожирается INSERT'ами, wraparound подкрадывается. Fix — trigger vacuum по `autovacuum_freeze_max_age` (в PG 13+ также `autovacuum_vacuum_insert_threshold` и `autovacuum_vacuum_insert_scale_factor`):

```sql
-- PostgreSQL 13+
ALTER TABLE audit_log SET (
    autovacuum_vacuum_insert_scale_factor = 0.05
);
```

Триггерит vacuum когда 5% данных вставлено (freeze new tuples раньше).

**Partitioned tables**. Каждая партиция — своя таблица со своим autovacuum. Обычно ok, но:

- Autovacuum workers могут не поспевать (много партиций, `max_workers = 3` не хватает).
- Увеличить `autovacuum_max_workers = 6` или больше.
- Настройки применяются на конкретную партицию, не на parent.

**Temporary tables**. Autovacuum их не трогает (private to session). Нужно ANALYZE вручную если много данных и потом queries.

**Materialized views**. Autovacuum обрабатывает как обычные таблицы. При REFRESH — полная перезапись, VACUUM всё почистит на следующем цикле.

## Заключение

Autovacuum — не «настроил и забыл» для больших таблиц. Defaults разработаны для маленьких таблиц. На таблицах 100M+ rows они гарантированно приведут к bloat и wraparound risk.

**Ключевые правила**:

- **`autovacuum = on` всегда**. Никогда не отключайте.
- **Per-table tuning для больших таблиц** — `scale_factor = 0.005-0.02`, `cost_limit = 1000-2000`, `cost_delay = 1ms`.
- **Мониторить n_dead_tup** через pg_stat_user_tables. Alert на bloat > 20%.
- **Мониторить age(relfrozenxid)** — alert на > 60% от freeze_max_age.
- **pg_repack для сжатия** bloated таблиц, **не VACUUM FULL**.
- **ANALYZE после bulk INSERT** — не ждать autovacuum.
- **Insert-only таблицы** — использовать `autovacuum_vacuum_insert_scale_factor` (PG 13+).
- **Partitioned tables** — увеличить `autovacuum_max_workers`.

**VACUUM vs VACUUM FULL vs pg_repack**:

- VACUUM — регулярно, не блокирует, не возвращает место.
- VACUUM FULL — только в maintenance window, блокирует всё, возвращает место.
- pg_repack — онлайн-эквивалент VACUUM FULL, для проды.

**XID wraparound** — самая большая опасность. Игнорирование = потенциальный `database refuses to accept commands`. Уменьшить `autovacuum_freeze_max_age` для важных таблиц, мониторить `age(relfrozenxid)`.

Практический совет для КНП: возьмите свою прод БД, прогоните audit script (см. выше), найдите топ-10 таблиц с bloat. Для каждой — рассчитайте нужный `scale_factor` по формуле «сколько dead tuples в день = сколько раз в сутки хотим autovacuum». Application `ALTER TABLE` per-table. Через неделю — снова audit, увидите, что bloat стабилизировался или уменьшился. Через месяц — забудете о N+1 проблемах в query planner из-за stale statistics.

Дальше — читайте статью Percona «Deep Dive into PostgreSQL Vacuum» и Vlad Mihalcea про MVCC. Настройте Grafana dashboard с метриками bloat, xid age, vacuum activity. Каждый инцидент с медленной БД начинайте с проверки `pg_stat_user_tables` — половина проблем окажется в autovacuum, о котором никто не думал.
