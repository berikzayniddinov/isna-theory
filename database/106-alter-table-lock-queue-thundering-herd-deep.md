# 106. ALTER TABLE на горячей таблице: lock queue, retry storm, memory blow-up — deep-dive

## Зачем это знать

Есть один класс production-инцидентов, который любой senior видел хотя бы раз, а часто устраивал сам. Идея простая: «нужно добавить колонку в таблицу». Кажется тривиально — ALTER TABLE, работы на микросекунды, миграция через Liquibase. Что может пойти не так?

В реальном инциденте на таблице `meta_document` в tax-rep (400+ миллионов строк) — всё. Liquibase попал в crashloop backoff. Таблица залочилась намертво. Backend'ы прода забились в очередь на этот lock. HikariCP таймауты каждые 30 секунд, клиенты retry, очередь квадратично разрастается. Когда через 10 минут lock наконец отпустился — «прорыв плотины»: сотни отложенных запросов и их ретраев ринулись одновременно. Каждый — свой backend процесс, каждый — со своим work_mem'ом на hash join / sort, каждый — с parallel workers. RAM с 30 GB за 30 минут вырос до 75 GB. Вся система лежит.

И самое поучительное — реальной работы ALTER'а было **микросекунды**. Он не переписывал таблицу, не создавал новый файл, не обновлял ни одной существующей строки. Просто изменил tuple descriptor в системном каталоге. Микросекунды работы вызвали 30 минут downtime и десятки GB RAM.

Понимать эту цепочку событий критично по трём причинам. Первая — DDL на горячих таблицах бывает у всех и всегда. Migration через Liquibase — стандарт enterprise. Не понимая, что именно делает ALTER, разработчик пишет наивный `ADD COLUMN` в migration, деплой запускается в busy hour, вся система ложится. Классика.

Вторая — это учебник по каскадированию отказов. Инцидент начинается с маленькой проблемы (lock на секунду), но каскадирует через несколько независимых механизмов: FIFO lock queue блокирует новые запросы, HikariCP retry генерирует дубликаты, work_mem умножается на количество backends, background workload (fno migration + аналитика) добивает I/O. Каждый компонент отдельно — безобидный. Вместе — катастрофа.

Третья — есть правильный способ делать ALTER на больших таблицах. `lock_timeout`, expand-contract pattern, разбиение на маленькие изменения, `SET (fillfactor)` заранее, `CONCURRENTLY` где возможно. Понимая **почему** наивный подход не работает, вы правильно применяете эти техники.

Мы разберём весь путь этого инцидента детально. Что такое ACCESS EXCLUSIVE lock и почему его берёт даже catalog-only ALTER (не переписывающий таблицу). Что такое relcache и почему изменение tuple descriptor требует блокировки всех читателей. FIFO очередь lock manager'а — почему длинный SELECT впереди ALTER'а блокирует и все последующие SELECT'ы. Механика retry storm через HikariCP: как 30-секундный timeout превращается в квадратично растущую очередь. Прорыв плотины и thundering herd — что происходит когда lock освобождается. Memory blow-up: почему сотни backend'ов пожирают десятки GB — не потому что кэшируют, а потому что каждый аллоцирует work_mem под hash join. Как это диагностировать в проде: `pg_locks`, `pg_stat_activity`, `pg_blocking_pids`, память по PID. И самое важное — как избежать: `lock_timeout`, `NOT VALID` для constraints, `CREATE INDEX CONCURRENTLY`, expand-contract, deploy в maintenance windows, feature flags вокруг DDL.

## Анатомия инцидента: что произошло по времени

Восстановим цепочку событий в meta_document инциденте. Числа примерные, но паттерн реален.

**16:00** — фоновый workload: fno-миграция активно вставляет в `meta_document` (~270 msg/s), плюс аналитические запросы `count(distinct f.id)` по трём JOIN'ам. Диск загружен на 40-80% iowait весь день. БД disk-bound.

**16:43** — Liquibase migration запускается: `ALTER TABLE meta_document ADD COLUMN file_desc VARCHAR(255)`. Работы — микросекунды. Но требует ACCESS EXCLUSIVE lock.

**16:43** — Lock manager: посмотреть текущие locks на meta_document. Есть длинный аналитический SELECT, взявший ACCESS SHARE (обычный SELECT). ALTER не может получить ACCESS EXCLUSIVE — ждёт в **очереди**.

**16:43+ε** — Новые запросы к meta_document (SELECT'ы, INSERT'ы от fno-миграции) приходят. Их ACCESS SHARE / ROW EXCLUSIVE **совместим** с текущим SELECT'ом впереди. Но **несовместим** с ждущим в очереди ALTER'ом. Lock manager ставит их **после** ALTER'а. Все SELECT'ы блокированы, хотя таблица «читается».

**16:43-16:52** — Очередь растёт. Приложение шлёт запросы, HikariCP отправляет их в БД, backend'ы висят в `wait_event = 'relation'` lock. Каждые 30 секунд HikariCP `connectionTimeout` в приложении — Java-код получает `SQLException`, throws, retry logic пробует снова → **тот же запрос ещё раз в очередь**. За 10 минут — сотни ретраев поверх сотен оригинальных запросов.

**16:52** — Длинный аналитический SELECT наконец закончился. Lock manager видит: ALTER первый в очереди, может получить ACCESS EXCLUSIVE. Даёт lock. ALTER выполняется за миллисекунды. Освобождает lock.

**16:52+ε** — **Прорыв плотины**. Все ждавшие запросы (плюс их ретраи) стартуют одновременно. Каждый — отдельный backend процесс. Каждый анализирует свой query, аллоцирует work_mem, стартует hash join, sort, parallel workers.

**16:53-17:15** — Memory usage растёт нелинейно. 200 concurrent backends × 100 MB work_mem × 4 parallel workers = 80 GB. RSS сервера с 30 GB → 75 GB. OS OOM killer близко. Postgres backend процессы падают. HikariCP timeouts, retries. Каскад продолжается.

**17:15** — Ops команда прибегает, ручной `kill -9` большинства backends, connection pool перезапускается, приложение перезагружается. Стабилизация.

Разберём каждое звено этой цепочки детально.

## ACCESS EXCLUSIVE lock и tuple descriptor

Первое звено — почему ALTER взял lock, блокирующий даже SELECT'ы.

В PostgreSQL 8 уровней table-level locks, от слабого ACCESS SHARE (обычный SELECT) до сильнейшего ACCESS EXCLUSIVE. Матрица совместимости определяет, кто с кем может сосуществовать.

Ключевое правило: **уровень lock определяется типом операции, а не объёмом работы**. `SELECT count(*) FROM billion_row_table` (часы работы) берёт мягкий ACCESS SHARE. `ALTER TABLE small_table ADD COLUMN x INT` (миллисекунды работы) берёт **жёсткий** ACCESS EXCLUSIVE.

Почему такая асимметрия? Всё сводится к консистентности **tuple descriptor**.

Каждая таблица имеет tuple descriptor — метаданные, описывающие структуру строки: список колонок, их типы, значения по умолчанию, ограничения. Descriptor хранится в системном каталоге (`pg_attribute`, `pg_class`) и **закэширован в памяти каждого backend процесса** (это называется relcache — relation cache).

Когда backend делает SELECT, он читает tuple descriptor из своего relcache, парсит page байты в соответствии с описанием: «поле 1 — int4, поле 2 — varchar с offset 4, поле 3 — timestamp». Descriptor + физическая структура tuple на странице **должны совпадать**. Если backend думает, что колонок 3, а на странице записаны с четвёртой — получим garbage, crash, corruption.

Теперь представим наивную реализацию `ADD COLUMN` без ACCESS EXCLUSIVE:

- Backend A читает старый descriptor (3 колонки), SELECT'ит tuple, парсит.
- В этот момент другой backend B обновляет pg_attribute: descriptor становится 4 колонки.
- Backend A ещё в середине парсинга — его код думает 3 колонки, но физический tuple может быть уже с 4-й (если INSERT'ы пришли после ALTER).

Результат — неопределённое поведение. PostgreSQL этого не может допустить.

Решение: **изменения descriptor'а требуют, чтобы никто не читал таблицу в этот момент**. ACCESS EXCLUSIVE lock даёт эту гарантию. Плюс: PostgreSQL при commit ALTER TABLE рассылает **relcache invalidation** всем backends — «выброси свой закэшированный descriptor, перечитай из каталога». Только после этого новые SELECT'ы могут работать с обновлённой схемой.

Отсюда следствие: **все ALTER TABLE, меняющие tuple descriptor, берут ACCESS EXCLUSIVE**, независимо от объёма работы. `ADD COLUMN` без DEFAULT в PG 11+ — работы **ноль** (метадата только), но lock жёсткий. `RENAME COLUMN`, `DROP COLUMN`, `ADD CONSTRAINT` без NOT VALID, `ALTER COLUMN TYPE` — все ACCESS EXCLUSIVE.

Есть исключения: `CREATE INDEX CONCURRENTLY` берёт SHARE UPDATE EXCLUSIVE (совместим с DML), `ADD CONSTRAINT ... NOT VALID` — тоже мягче. Но большинство ALTER'ов — ACCESS EXCLUSIVE.

Понимая это, ключевой вывод: **опасность ALTER на горячей таблице — всегда про lock, почти никогда про I/O**. Если это не rewrite-вариант (`ALTER COLUMN TYPE`, `ADD COLUMN NOT NULL DEFAULT volatile_expr` до PG 11), то реальная работа — микросекунды. Опасность — во времени ожидания lock'а в очереди.

## FIFO lock queue — почему длинный SELECT блокирует всех

Второе звено — почему lock, который держит один SELECT, вдруг блокирует все последующие SELECT'ы, хотя между собой они совместимы.

PostgreSQL lock manager использует **FIFO очередь**. Пришла заявка на lock — ставится в конец очереди. Wait until getting to the head + все впереди совместимы.

Пример. На таблице meta_document состояние:

```
Time  Event
─────  ───────────────────────────────────────
T=0   Backend A: LONG_SELECT берёт ACCESS SHARE
      Lock holders: [A: ACCESS SHARE]
      Queue: []

T=1   Backend B: ALTER TABLE просит ACCESS EXCLUSIVE
      ACCESS EXCLUSIVE несовместим с ACCESS SHARE → wait
      Lock holders: [A: ACCESS SHARE]
      Queue: [B: ACCESS EXCLUSIVE waiting]

T=2   Backend C: обычный SELECT просит ACCESS SHARE
      На первый взгляд C совместим с A. Но C сзади B в очереди.
      PostgreSQL: если пропущу C через B, то B никогда не получит lock
        (новые SELECT'ы будут постоянно приходить).
      Правило: C ждёт своей очереди ЗА B.
      Lock holders: [A: ACCESS SHARE]
      Queue: [B: ACCESS EXCLUSIVE, C: ACCESS SHARE]

T=3   Backend D: SELECT → в конец очереди
T=4   Backend E: INSERT → в конец очереди
...
```

Это анти-starvation защита. Без FIFO order сильные lock'и никогда бы не получили — постоянный поток слабых блокировал бы их. С FIFO — заявка на lock рано или поздно дождётся.

Но следствие: **один длинный SELECT + один ALTER = блокировка всей таблицы**. Никакие новые запросы не проходят, даже совместимые с текущим holder'ом.

В meta_document инциденте: длинный аналитический SELECT `count(distinct f.id) FROM fno JOIN fno_revision JOIN fno_version` работал несколько минут (disk-bound, миллионы rows). ALTER пришёл и встал в очередь. Все последующие запросы приложения к meta_document — тоже в очередь. Таблица «сдохла».

Ключевое: **длинные SELECT'ы — не невинны на busy таблицах**. Они блокируют потенциальные ALTER'ы, а через них — всех. В идеале аналитические запросы должны идти на read replica, не на primary. Или хотя бы иметь low priority + timeout.

Как посмотреть, кто держит lock и кто ждёт:

```sql
SELECT 
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocked.query AS blocked_query,
    NOW() - blocked.query_start AS blocked_duration,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocking.query AS blocking_query,
    NOW() - blocking.query_start AS blocking_duration,
    blocking.state AS blocking_state
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking 
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock'
ORDER BY blocked_duration DESC;
```

В момент инцидента этот запрос показал бы: сотни backends в состоянии `Lock` wait, все ждут одного pid — ALTER. Тот в свою очередь ждёт длинный SELECT.

## HikariCP retry storm — как очередь растёт квадратично

Третье звено — почему очередь не просто линейно растёт со временем, а *квадратично*.

Стандартная настройка Java-приложений: HikariCP с `connectionTimeout = 30s`. Это время, которое приложение ждёт connection из пула. Если pool exhausted или ответ от БД не приходит — SQLException.

В инциденте что происходит:

- **T=0**: пришёл HTTP-запрос → приложение делает SELECT из meta_document → идёт через HikariCP → backend в БД → ждёт lock.
- **T=30s**: HikariCP не дождался, SQLException. Приложение (или клиент через retry mechanism, например Spring Retry, или пользователь через F5) повторяет запрос.
- **T=30s + ε**: новый запрос → новый backend → тоже ждёт lock. **Старый backend продолжает висеть** — Postgres не знает, что Java-клиент уже сдался. Backend в БД будет ждать lock, пока получит его или пока клиент явно `pg_cancel_backend`.

Проблема: **отказ HikariCP не отменяет backend в БД**. Java-код думает «я retry'нул», в БД теперь два backend'а с одинаковым запросом, оба ждут lock.

Плюс есть feedback loop:

- Java пользователь получает 500 → нажимает F5 → ещё один запрос.
- Приложение делает несколько ретраев сам (Spring Retry с backoff).
- Async jobs с очередями (Kafka, RabbitMQ consumer) — если message не acknowledged, retry, ещё message.

Через 5 минут ожидания lock'а: если каждый timeout генерирует 3 ретрая (клиент + retry lib + кэш), а timeout — 30 секунд, то за 300 секунд один оригинальный запрос порождает 3^10 = 59049 ретраев. **Экспоненциально**.

На практике не так плохо (retry backoff, circuit breakers), но грубая формула: **очередь растёт квадратично от времени ожидания**.

В meta_document инциденте: за 10 минут lock ожидания очередь выросла до сотен backends на одну таблицу.

Мониторинг это видит:

```sql
-- Активные backends по таблице (ждут lock)
SELECT count(*) AS waiting_count, mode, granted
FROM pg_locks
WHERE relation = 'meta_document'::regclass
GROUP BY mode, granted
ORDER BY waiting_count DESC;
```

`granted = false` — те, кто в очереди. Норма — 0-5. В инциденте — 500+.

## Прорыв плотины — thundering herd effect

Четвёртое звено — что происходит, когда lock освобождается.

ALTER выполняется, ACCESS EXCLUSIVE отпускается. Lock manager проходит по очереди: следующий может получить lock? Да? Wake up backend, дай lock.

Проблема: **вся очередь просыпается одновременно**. Все pending SELECT'ы совместимы между собой (ACCESS SHARE), все стартуют параллельно.

Это классический **thundering herd** паттерн. 500 backends, ждавших 10 минут, вдруг все стартуют в первую секунду после освобождения lock'а.

Что делают эти backend'ы:

- Каждый парсит свой SQL (используя новый tuple descriptor — relcache invalidated).
- Планирует запрос — вычисляет cost, выбирает план.
- Выполняет — читает страницы из shared_buffers или диска.
- Для JOIN'ов, sorts, aggregates — аллоцирует **work_mem** (default 4 MB, но в prod часто повышают до 64-256 MB).
- Может запустить **parallel workers** — до `max_parallel_workers_per_gather` (обычно 2-4).

Каждый worker — тоже свой backend процесс, тоже свой work_mem.

Итого одна большая аналитическая query может потребить:

```
Leader backend RSS baseline:   50-100 MB
+ work_mem (× число операций sort/hash): 64 MB × 4 = 256 MB
+ Parallel workers: 3 × (50 + 256) = 900 MB
= ~1.2 GB на один сложный запрос
```

При 500 таких запросов одновременно: **600 GB**. Естественно, сервер с 96 GB RAM не выдерживает.

Даже если реальность мягче (не все запросы такие тяжёлые), рост RAM 30 → 75 GB за 30 минут абсолютно объясним.

Что видит мониторинг:

```
Time   Postgres RSS   User CPU  IO Wait   Description
────   ────────────   ────────  ───────   ───────────
16:00  30 GB          20%       40%       Baseline (disk-bound fno)
16:43  30 GB          20%       80%       ALTER waiting, queue growing
16:52  30 GB          20%       80%       ALTER completes
16:53  35 GB          40%       80%       Queue draining, parsing
16:55  45 GB          70%       60%       Parallel queries starting
17:00  60 GB          90%       50%       Hash joins, sorts allocated
17:10  70 GB          95%       40%       At capacity
17:15  75 GB          100%      —         OOM, backend crashes
```

Ключевой момент: **RAM растёт из anonymous memory backends, не из page cache**. Page cache стабильный (данные уже загружены). Anonymous — временный, для операций текущих запросов.

## Memory blow-up: work_mem — cкрытая multiplier

Понимая механику блокировки, самая недооценённая часть — memory blow-up.

`work_mem` — параметр PostgreSQL, определяющий, сколько памяти одна операция (hash join, sort, aggregate) может использовать в памяти прежде чем «спилл» на диск.

Default `work_mem = 4MB`. Часто в проде повышают до 64-256 MB для аналитики.

Ключевое: **work_mem — per operation per connection**. Не глобальный лимит. Не per query. Именно **per operation**.

Один сложный запрос с 3 hash joins и 2 sorts:

- Hash join 1: 64 MB
- Hash join 2: 64 MB  
- Hash join 3: 64 MB
- Sort 1: 64 MB
- Sort 2: 64 MB
= 320 MB для одного запроса

Плюс parallel workers — каждый worker имеет свой work_mem для каждой операции:

- Leader: 320 MB
- 4 workers × 320 MB = 1280 MB
= 1.6 GB для одного запроса

При 100 concurrent таких запросов — 160 GB.

Формула для оценки max memory postgres при busy period:

```
max_postgres_memory ≈ 
    shared_buffers (fixed, обычно 25% RAM)
  + effective_cache_size (виртуальный, не потребляется)
  + (max_connections × avg_backend_rss)             ← baseline процессов
  + (concurrent_queries × avg_work_mem_per_query)   ← ключевой multiplier
```

`concurrent_queries` — обычно 10-50 в normal load. В thundering herd — 200-500. Multiplier × 10.

Настройка work_mem — тонкий баланс. Слишком мало — операции спиллят на диск, медленно (external merge sort — часы вместо секунд). Слишком много — при burst нагрузке RAM blow-up.

## Что стало ключевым триггером

Собираем инцидент воедино. Что реально произошло в meta_document:

1. Приложение весь день работало на disk-bound грани (fno migration + аналитика).
2. Один длинный аналитический SELECT завис на несколько минут (диск-bound).
3. Пришёл Liquibase migration с `ALTER TABLE ADD COLUMN` — застрял в lock queue за длинным SELECT.
4. Все новые запросы к meta_document встали за ALTER'ом в FIFO очередь.
5. HikariCP на приложении: 30-секундные timeouts, retry, retry, retry — очередь растёт квадратично.
6. Ждавшие backends в БД не отменялись — все ждут одного lock'а.
7. Через 10 минут длинный SELECT закончился, ALTER мгновенно выполнился, отпустил lock.
8. Thundering herd: сотни pending queries одновременно стартовали.
9. Каждая query: parse + plan + execute с work_mem × parallel workers.
10. Memory usage взрыв 30 → 75 GB за 30 минут.
11. OS memory pressure → OOM killer → backends крашатся → чтобы Liquibase таблицу оставил не мигрированной → crashloop backoff.

Ключевой урок: **ALTER TABLE был последней каплей**, но не единственной причиной. Уже был disk-bound baseline, длинный SELECT впереди, приложение с ретраями. Один невинный ALTER стал триггером каскада.

## Правильный подход: как делать ALTER TABLE на горячей таблице

Понимая механику, есть проверенные patterns.

**1. Всегда `SET lock_timeout` перед ALTER**.

```sql
SET lock_timeout = '5s';
ALTER TABLE meta_document ADD COLUMN file_desc VARCHAR(255);
```

Если lock не получен за 5 секунд — ALTER падает с error, миграция откатывается, никого не блокирует. Повторяем в maintenance window или когда таблица менее занята.

**Никогда не запускайте ALTER без lock_timeout в проде**. Это первое правило.

В Liquibase changeset:

```xml
<changeSet id="add-file-desc" author="berik">
    <sql>SET lock_timeout = '5s'</sql>
    <addColumn tableName="meta_document">
        <column name="file_desc" type="VARCHAR(255)"/>
    </addColumn>
</changeSet>
```

**2. Retry logic на уровне миграции**.

Если lock_timeout сработал, повторить через некоторое время. Полезно для eventually-successful миграций:

```sql
DO $$
DECLARE
    attempt INT := 0;
    max_attempts INT := 20;
BEGIN
    WHILE attempt < max_attempts LOOP
        BEGIN
            SET LOCAL lock_timeout = '3s';
            ALTER TABLE meta_document ADD COLUMN file_desc VARCHAR(255);
            EXIT;  -- успех
        EXCEPTION WHEN lock_not_available THEN
            attempt := attempt + 1;
            PERFORM pg_sleep(30);  -- ждём 30 сек между попытками
        END;
    END LOOP;
    IF attempt = max_attempts THEN
        RAISE EXCEPTION 'Could not acquire lock after % attempts', max_attempts;
    END IF;
END $$;
```

За 10 минут — 20 попыток. Каждая быстрая — 3 сек timeout. Если lock свободен — миграция проходит без blocking.

**3. Убить блокеров перед ALTER**.

Иногда есть один-два длинных запроса, которые всегда блокируют. Скрипт: посмотреть текущих holders, kill'нуть их, потом сразу ALTER.

```sql
-- Найти долгие queries, держащие ACCESS SHARE на нужной таблице
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE query LIKE '%meta_document%'
  AND state = 'active'
  AND NOW() - query_start > interval '1 minute';

SET lock_timeout = '2s';
ALTER TABLE meta_document ADD COLUMN file_desc VARCHAR(255);
```

Опасно, но иногда нужно. Пользователи получат ошибки, но лучше это, чем 30-минутный downtime.

**4. Разбивать ALTER на маленькие изменения**.

Плохо:

```sql
ALTER TABLE users 
    ADD COLUMN full_name VARCHAR(255),
    ADD COLUMN address TEXT,
    ADD COLUMN phone VARCHAR(50);
```

Один ALTER, один lock, вся работа скопом.

Лучше:

```sql
SET lock_timeout = '3s';
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
-- Успешно? Пауза 30 сек, чтобы дать pending queries пройти.

SET lock_timeout = '3s';
ALTER TABLE users ADD COLUMN address TEXT;
-- ...

SET lock_timeout = '3s';
ALTER TABLE users ADD COLUMN phone VARCHAR(50);
```

Три отдельных ALTER, каждый — маленький lock окно. Меньше вероятность попасть на длинный SELECT.

**5. NOT VALID для constraints**.

Плохо: `ALTER TABLE ADD CONSTRAINT ...` — сканирует всю таблицу под ACCESS EXCLUSIVE.

Хорошо:

```sql
-- Быстрый, только проверяет новые/изменённые rows
ALTER TABLE meta_document ADD CONSTRAINT chk_size CHECK (size > 0) NOT VALID;

-- Отдельно, без блокировки DML, валидирует существующие rows
ALTER TABLE meta_document VALIDATE CONSTRAINT chk_size;
```

`ADD CONSTRAINT NOT VALID` — очень быстрый (только меняет catalog), не сканирует. `VALIDATE CONSTRAINT` — сканирует, но берёт SHARE UPDATE EXCLUSIVE (совместим с DML).

**6. CREATE INDEX CONCURRENTLY**.

Обычный `CREATE INDEX` — SHARE lock, блокирует DML. На большой таблице — часы.

```sql
CREATE INDEX CONCURRENTLY idx_meta_document_file_desc ON meta_document(file_desc);
```

Не блокирует DML. Работает медленнее (два прохода таблицы, вместо одного), но онлайн.

Ловушка: `CONCURRENTLY` нельзя в транзакции. Liquibase может ругаться — нужно `<sql splitStatements="false" endDelimiter=";">` или явно `<createIndex clustered="false" ... concurrently="true"/>` (некоторые версии).

**7. Expand-contract pattern для типов колонок**.

`ALTER COLUMN type` — обычно полная перезапись таблицы под ACCESS EXCLUSIVE. Часы блокировки.

Правильно — expand-contract через два этапа:

```sql
-- Step 1 (expand): добавить новую колонку
ALTER TABLE users ADD COLUMN id_new BIGINT;

-- Step 2: заполнить постепенно (background job)
UPDATE users SET id_new = id::BIGINT WHERE id_new IS NULL LIMIT 10000;
-- ...повторять пока WHERE условие даёт rows

-- Step 3: свитч в приложении на использование id_new
-- Deploy new version.

-- Step 4 (contract, через недели): drop old
ALTER TABLE users DROP COLUMN id;
ALTER TABLE users RENAME COLUMN id_new TO id;
```

Долго, но безопасно.

## Настройки уровня приложения — как не сделать storm ещё хуже

Со стороны БД можно много всего. Но приложение тоже вносит свой вклад в retry storm.

**1. Правильный `statement_timeout`**. По умолчанию Postgres не имеет timeout для запросов. Один SELECT может ждать lock часами. Установить:

```sql
-- Глобально или per-role
ALTER ROLE app_user SET statement_timeout = '30s';
```

Если lock не получен + не начался запрос за 30 секунд — Postgres cancel'ит. Backend освобождается, не висит бесконечно.

**2. HikariCP не должен infinite retry**.

Многие приложения имеют `spring.retry` или Resilience4j. Проверьте, что retry имеет:

- Ограничение количества попыток (3-5, не бесконечно).
- Exponential backoff (не immediate retry).
- Circuit breaker — после N failures переставать пробовать некоторое время.

Иначе получаете infinite retry loop, добивающий БД.

**3. Timeout HikariCP чуть больше statement_timeout**.

Если HikariCP таймаутит раньше, чем Postgres statement_timeout — Java-код думает «сорвалось», retry, но старый backend в БД ещё работает. Дубли.

Правильный порядок:

```
statement_timeout (Postgres):        20s
connectionTimeout (HikariCP):        30s
retry_delay (application):           60s
```

HikariCP терпит Postgres statement_timeout. Retry ждёт достаточно, чтобы старый запрос точно отвалился.

**4. Read replica для аналитических запросов**.

Длинные `count(distinct ...)` — не должны быть на primary. Read replica с streaming replication:

- Primary: только OLTP.
- Read replica: аналитика, отчёты.

Даже если реплика лагает — аналитика допускает stale data. Zero impact на writes и на lock queue primary.

**5. Feature flags вокруг DDL**.

Migration с DDL деплоится в час пик. Плохо. Если DDL в отдельном feature flag:

- Deploy кода с новой колонкой в OFF режиме.
- В час пик — код работает без использования колонки.
- Ночью, low traffic — flag включается, миграция запускается.
- После успеха — код начинает использовать новую колонку.

Не всегда возможно, но правильный паттерн для рискованных DDL.

## Диагностика в проде — что смотреть

Сценарий: users жалуются на медленные запросы. Проверить, не lock storm ли.

**1. Активные locks и waiting**:

```sql
SELECT 
    l.pid,
    l.mode,
    l.granted,
    a.query,
    a.wait_event_type,
    a.wait_event,
    NOW() - a.xact_start AS xact_duration,
    NOW() - a.query_start AS query_duration
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.relation = 'meta_document'::regclass
ORDER BY l.granted, xact_duration DESC;
```

`granted = false` — те, кто в очереди. Если много (десятки) — проблема.

**2. Blocking chain**:

```sql
WITH RECURSIVE lock_chain AS (
    SELECT 
        pid, 
        pid AS root_blocker,
        query,
        ARRAY[pid] AS chain,
        0 AS depth
    FROM pg_stat_activity
    WHERE pid = ANY (
        SELECT DISTINCT pg_blocking_pids(pid) 
        FROM pg_stat_activity 
        WHERE wait_event_type = 'Lock'
    )
    AND pid NOT IN (SELECT unnest(pg_blocking_pids(pid)) FROM pg_stat_activity)
    
    UNION ALL
    
    SELECT 
        a.pid,
        lc.root_blocker,
        a.query,
        lc.chain || a.pid,
        lc.depth + 1
    FROM pg_stat_activity a
    JOIN lock_chain lc ON a.pid = ANY(lc.chain) OR lc.pid = ANY(pg_blocking_pids(a.pid))
    WHERE NOT a.pid = ANY(lc.chain)
)
SELECT * FROM lock_chain ORDER BY root_blocker, depth;
```

Показывает всю цепочку lock waits: root blocker и все, кто за ним ждут.

**3. Kill root blocker**:

```sql
SELECT pg_terminate_backend(<root_blocker_pid>);
```

Осторожно. Клиент root blocker'а получит error, но остальная система разблокируется.

**4. Memory usage postgres backends**:

```bash
# На хосте где Postgres
ps auxf | grep postgres | sort -nrk 6 | head -20
```

Показывает топ backend'ов по RSS. Если один backend занимает несколько GB — вероятно тяжёлый query с большим work_mem.

**5. pg_stat_activity память per query**:

```sql
SELECT 
    pid, usename, application_name,
    NOW() - xact_start AS xact_duration,
    state, wait_event, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_start;
```

`application_name` подскажет какой микросервис запустил запрос.

## Постмортем — что делать после инцидента

После инцидента:

**1. Timeline из логов**. Postgres log_lock_waits (log_lock_waits = on, log_min_duration_statement < deadlock_timeout). Прометей метрики. Соберите точный timeline.

**2. Найти root cause**. Обычно это длинный holder lock + retry storm. Что можно было бы предотвратить?

**3. Fix'ы**:

- Настроить `statement_timeout` — 30-60 сек.
- Настроить `lock_timeout` в DDL миграциях — 5 сек.
- Мониторинг: alert на `pg_locks` количество waiting > 20.
- Мониторинг: alert на `pg_stat_activity` где xact_duration > 5 минут.
- Переместить аналитику на read replica.
- Retry policy приложения: ограниченный, с backoff.

**4. Runbook**. Документация для оперативной команды: если lock storm — что делать. Скрипт для kill root blockers.

**5. Load test**. Симулировать сценарий: длинный query + ALTER + burst. Убедиться, что новые настройки не приводят к каскаду.

## Заключение

meta_document инцидент — учебник по каскадированию отказов в PostgreSQL. Один невинный ALTER TABLE ADD COLUMN, работы на микросекунды, превратился в 30-минутный downtime с memory blow-up 30 → 75 GB.

Ключевые механизмы:

- **ACCESS EXCLUSIVE lock** для любого DDL, меняющего tuple descriptor — независимо от объёма работы. Из-за relcache инвалидации.
- **FIFO очередь lock manager** — один длинный SELECT + один ALTER = блокировка всех последующих запросов, даже совместимых.
- **HikariCP retry storm** — 30-секундные timeouts + Java retry logic = квадратичный рост очереди.
- **Thundering herd** при освобождении lock — сотни queries стартуют одновременно.
- **Memory blow-up** через work_mem × concurrent queries × parallel workers.

**Правильный подход**:

- `SET lock_timeout = '5s'` **всегда** перед ALTER в проде. Fail fast лучше 30-минутного downtime.
- Retry logic в миграции с exponential backoff и лимитом попыток.
- Разбивать большие ALTER на маленькие. Пауза между ними.
- `ADD CONSTRAINT NOT VALID` + `VALIDATE CONSTRAINT` вместо прямого ADD.
- `CREATE INDEX CONCURRENTLY` вместо CREATE INDEX.
- Expand-contract pattern для типов колонок.
- Read replica для аналитики — не блокировать primary DDL.
- `statement_timeout` в приложении — не дать backend'ам висеть.
- Правильный retry policy приложения — bounded, backoff, circuit breaker.

**Диагностика**: `pg_locks` для waiting, `pg_stat_activity` для xact_duration, `pg_blocking_pids` для chain, `ps aux` для memory per backend. Мониторинг с alerts на waiting locks и long xacts.

Практический совет для КНП: ревьюите все существующие Liquibase migrations, добавьте `lock_timeout` в те, которые делают ALTER на таблицах >10M rows. Настройте `statement_timeout` глобально для app_user. Настройте Prometheus alert на `pg_locks_count{granted="false"} > 20 for 30 seconds`. Прогонов один production ALTER с новыми настройками, наблюдайте graceful failure и retry. Через неделю ваши миграции будут либо мгновенно проходить, либо cleanly фейлиться — никакого 30-минутного downtime.

Дальше — читайте документацию PostgreSQL про MVCC и Lock Manager, статьи GitLab про их zero-downtime migrations, Percona blog про safe ALTER TABLE patterns. И самое важное — сделайте post-mortem вашего meta_document инцидента с timeline и action items. Через несколько таких post-mortem'ов интуиция «что может пойти не так с этой миграцией» становится второй природой.
