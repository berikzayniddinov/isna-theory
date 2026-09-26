# 88. Locks в PostgreSQL и как читать план запроса

## Зачем это знать

Есть две темы, без которых senior-инженер не может работать с PostgreSQL в проде: локи и планы запросов. Первое — потому что 90% инцидентов «база висит» упирается в locks: кто-то кого-то ждёт, транзакции копятся, приложение тормозит. Второе — потому что любая оптимизация начинается с того, что ты читаешь план запроса, находишь узкое место, придумываешь как его убрать, применяешь fix и проверяешь через план ещё раз.

Обе темы часто спрашивают на собесах, и обе плохо покрыты в стандартной документации. Документация формально описывает восемь режимов lock'а и десяток типов узлов плана — но не объясняет, как это использовать в реальной жизни. Как понять, что твоя миграция ALTER TABLE положит систему на пять минут? Как отличить хороший план от плохого за секунду? Как найти в проде того, кто держит эксклюзивный лок два часа? Всё это — работа этого файла.

Мы разберём восемь режимов lock'а PostgreSQL и матрицу их совместимости — что с чем конфликтует. Пройдёмся по типовым операциям (SELECT, UPDATE, ALTER TABLE, CREATE INDEX) и посмотрим, какой лок берётся при каждой. Разберём row-level locks, advisory locks, `SELECT FOR UPDATE` и его модификации (`NOWAIT`, `SKIP LOCKED`). Покажем, как найти блокирующего в проде через `pg_locks` и `pg_blocking_pids`.

Потом переключимся на планы. Разберём все типы узлов (Seq Scan, Index Scan, Bitmap Heap Scan, все виды join'ов, Sort, Aggregate, Materialize и остальные). Научимся читать `EXPLAIN` — что означают cost, rows, width, actual time, loops, buffers. Разберём разницу `EXPLAIN` и `EXPLAIN ANALYZE`, зачем нужны `BUFFERS`, `VERBOSE`, `SETTINGS`. Пройдёмся по типичным «плохим» паттернам в планах: nested loop на большом результате, mismatched estimated vs actual rows, spill сортировки на диск, sequential scan там где должен быть index.

## Восемь режимов lock в PostgreSQL

Каждая операция в PostgreSQL берёт lock того или иного режима на объекты, с которыми работает. Режимов существует восемь, и они образуют иерархию от самого мягкого (совместимого с большинством других) до самого жёсткого (эксклюзивного). Названия могут показаться странными, но за ними стоит логика.

**ACCESS SHARE** — самый мягкий lock. Берётся простым SELECT'ом на таблице. Означает «я читаю таблицу, никто не должен её удалить или структурно изменить, пока я работаю». Множество ACCESS SHARE locks на одной таблице могут сосуществовать без конфликтов — это то, что позволяет тысяче параллельных SELECT'ов работать одновременно.

**ROW SHARE** — берётся `SELECT ... FOR UPDATE` или `SELECT ... FOR SHARE`. Название обманчиво — этот lock на уровне таблицы, а не строки. Row-level локи берутся отдельно, на конкретные строки. ROW SHARE на таблицу означает «в этой таблице сейчас есть строки под row-lock». Совместим с обычным SELECT'ом, но блокирует DDL, изменяющие структуру.

**ROW EXCLUSIVE** — берётся INSERT, UPDATE, DELETE, MERGE на таблице. Опять же название обманчиво: не эксклюзивный на строку, а обозначение «я модифицирую эту таблицу». Много ROW EXCLUSIVE могут сосуществовать (много писателей в разные строки таблицы). Конкретные строки защищены row-level локами.

**SHARE UPDATE EXCLUSIVE** — берётся VACUUM (без FULL), ANALYZE, CREATE INDEX CONCURRENTLY, REINDEX CONCURRENTLY, некоторые формы ALTER TABLE (например, добавление ограничения NOT VALID). Совместим с обычными SELECT/INSERT/UPDATE/DELETE, но блокирует одновременный VACUUM или ALTER, чтобы две административные операции не пересекались.

**SHARE** — берётся CREATE INDEX (не CONCURRENTLY). Блокирует любые изменения данных (INSERT/UPDATE/DELETE) на время построения индекса. Читатели по-прежнему работают. Обычно берётся на минуты или часы для больших таблиц — именно поэтому в проде используют CONCURRENTLY вариант.

**SHARE ROW EXCLUSIVE** — берётся CREATE TRIGGER и некоторые формы ALTER TABLE. Похож на SHARE, но также блокирует другие SHARE ROW EXCLUSIVE и SHARE. Редкий режим.

**EXCLUSIVE** — блокирует всё, кроме простых SELECT'ов (ACCESS SHARE). Берётся `REFRESH MATERIALIZED VIEW CONCURRENTLY`. Опять же редкий.

**ACCESS EXCLUSIVE** — самый жёсткий. Блокирует **всё**, включая простые SELECT'ы. Берётся DROP TABLE, TRUNCATE, REINDEX (не CONCURRENTLY), CLUSTER, VACUUM FULL, и большинством форм ALTER TABLE. Это тот lock, которого боятся в проде: пока он держится, ни один запрос к таблице не может даже прочитать данные. Если ALTER TABLE ждёт этот lock, а перед ним висит длинный SELECT, вся таблица становится недоступной для новых запросов, пока и SELECT не закончится, и ALTER не отработает.

Матрица совместимости показывает, какой lock с каким конфликтует. Полная матрица занимает 8×8, но ключевые правила простые. Первые три (ACCESS SHARE, ROW SHARE, ROW EXCLUSIVE) — «read/write locks», они между собой совместимы во всех комбинациях (кроме того, что ROW EXCLUSIVE конфликтует со SHARE и выше). Это позволяет нормальной OLTP-нагрузке работать полностью параллельно: много SELECT'ов, много UPDATE'ов в разные строки, всё идёт без блокировок на уровне таблицы.

Верхние пять (SHARE UPDATE EXCLUSIVE и выше) — «heavy locks», они начинают конфликтовать с обычной нагрузкой. VACUUM (SHARE UPDATE EXCLUSIVE) совместим с DML, но два VACUUM'а на одну таблицу конфликтуют. CREATE INDEX без CONCURRENTLY (SHARE) блокирует DML — все INSERT'ы будут стоять пока индекс не построится. DDL (ACCESS EXCLUSIVE) блокирует всё.

Одна важная деталь: **если процесс с сильным lock'ом ждёт слабый уже полученный, все более слабые lock'и после него тоже ждут**. Это делают через **FIFO очередь**. Пример: идёт длинный SELECT (ACCESS SHARE). Приходит ALTER TABLE (ACCESS EXCLUSIVE), встаёт в очередь. Приходят новые SELECT'ы — их ACCESS SHARE запросы совместимы с уже полученным ACCESS SHARE, но конфликтуют с ждущим ACCESS EXCLUSIVE. И PostgreSQL ставит их в очередь **после** ALTER'а, а не пропускает через. Результат: новые SELECT'ы висят, ожидая пока сначала закончится длинный SELECT, потом отработает ALTER, потом освободит lock. Таблица становится недоступной для новых запросов, даже для чтения.

Именно поэтому ALTER TABLE на нагруженной таблице — операция, требующая особой аккуратности. Даже если сам ALTER выполняется за секунду, если ждёт длинных SELECT'ов, он парализует таблицу на всё время ожидания. Правильный способ — использовать `lock_timeout`: `SET lock_timeout = '3s'` перед ALTER, чтобы если lock не получен за 3 секунды, операция упала, а не встала в очередь и не заблокировала всех. Повторить когда таблица меньше загружена.

## Row-level locks

До сих пор мы говорили о table-level локах. Но настоящая работа concurrency идёт на уровне строк. Именно row-level локи защищают конкретные записи от одновременного изменения двумя транзакциями.

Row-level lock устанавливается неявно при UPDATE и DELETE конкретной строки, или явно через `SELECT ... FOR UPDATE`. Есть четыре режима row-level lock, аналогичные table-level, но применяются к отдельным строкам: FOR KEY SHARE, FOR SHARE, FOR NO KEY UPDATE, FOR UPDATE.

**FOR UPDATE** — самый сильный. Транзакция намерена изменить или удалить эти строки. Никакая другая транзакция не сможет получить FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE или FOR KEY SHARE, и не сможет обновить или удалить эти строки, пока первая не закоммитит или не откатится.

**FOR NO KEY UPDATE** — почти как FOR UPDATE, но слабее: разрешает конкурентный FOR KEY SHARE. Используется PostgreSQL внутренне для foreign key проверок. Обычно приложениям не нужен.

**FOR SHARE** — «я читаю эти строки, никто не должен их изменить». Совместим с другими FOR SHARE и FOR KEY SHARE, но блокирует FOR UPDATE и FOR NO KEY UPDATE.

**FOR KEY SHARE** — самый мягкий. Блокирует только изменения key-колонок (обычно primary key). Другие изменения строки разрешены. Используется для foreign key checks.

Практическое применение FOR UPDATE — классика pessimistic locking. Пример: два оператора одновременно снимают деньги со счёта.

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 5 FOR UPDATE;
-- прочитали balance, например 1000
-- проверили что достаточно для операции
UPDATE accounts SET balance = balance - 300 WHERE id = 5;
COMMIT;
```

Без `FOR UPDATE` два оператора могли бы прочитать 1000, каждый решить что достаточно для снятия 700, и оба выполнить UPDATE. Итог: balance = -400 при том что каждый видел «достаточно». С FOR UPDATE второй оператор ждёт первого — читает уже 700, видит что 700 хватит только на 700, а не на 800 (если он снимает 800), выдаёт правильный отказ.

Row-level locks хранятся не в отдельной lock table, а в самом tuple'е heap — в его infomask битах и специальном поле xmax. Это делает их бесплатными по памяти (сколько бы row-locks ни было, они не занимают отдельного места в shared memory) и естественно масштабируемыми. Правда, есть ограничение: xmax — одно поле, поэтому только одна транзакция может владеть эксклюзивным lock'ом на строку. Для FOR SHARE, где может быть много владельцев одновременно, используется отдельная multi-xact механика — они называются MultiXactId, хранятся в отдельных SLRU буферах.

## SELECT FOR UPDATE NOWAIT и SKIP LOCKED

Два важнейших модификатора, которые превращают `SELECT FOR UPDATE` из простой блокировки в мощный инструмент.

**NOWAIT** — если строка уже заблокирована другой транзакцией, не ждать, а сразу упасть с ошибкой `ERROR: could not obtain lock on row in relation`. Полезно для fail-fast семантики: если критично не тормозить (например, ты обрабатываешь HTTP-запрос с жёстким latency budget), лучше упасть и вернуть 503, чем висеть неопределённое время.

**SKIP LOCKED** — самый интересный. Если строка заблокирована, **пропустить** её и вернуть только доступные. Идеально для queue-implementation через таблицу.

Классический паттерн — job queue на PostgreSQL:

```sql
BEGIN;
SELECT id, payload FROM jobs 
WHERE status = 'pending' 
ORDER BY created_at 
FOR UPDATE SKIP LOCKED 
LIMIT 1;
-- обработали job
UPDATE jobs SET status = 'done' WHERE id = <id>;
COMMIT;
```

Много воркеров могут параллельно запрашивать jobs. Каждый берёт **свою** job (заблокированные другими воркерами пропускаются), обрабатывает, коммитит. Никаких deadlocks, никакого contention. Это позволяет использовать обычную таблицу PostgreSQL как высокопроизводительную очередь без специализированных брокеров вроде RabbitMQ или Redis. Не для миллионов сообщений в секунду, но для типичных enterprise задач — вполне.

## Advisory locks

Отдельный класс блокировок, не связанный с таблицами и строками. Advisory lock — это просто lock на произвольный идентификатор (64-битное число), координированный PostgreSQL, но с семантикой которую определяет приложение.

```sql
SELECT pg_advisory_lock(12345);
-- сделали что-то
SELECT pg_advisory_unlock(12345);
```

Используется для координации между backend'ами при задачах, не связанных с таблицами. Классические примеры:

**Distributed locks для scheduled jobs**. Несколько подов приложения. Все хотят запустить nightly job, но нужно чтобы запустил только один. Каждый пытается взять advisory lock с идентификатором `hash('nightly-job-2024-01-15')`. Один получает, остальные — нет. Тот кто получил — работает, потом освобождает. ShedLock в Java-мире так и работает через PostgreSQL.

**Migration coordination**. Приложение стартует, хочет применить миграции. Много инстансов стартуют одновременно. Первый берёт advisory lock на «миграции», применяет, освобождает. Остальные ждут, потом видят что миграции применены и ничего не делают. Liquibase и Flyway так и работают.

**Rate limiting** на уровне БД. Если нужно гарантировать, что одна операция выполняется не чаще N раз в секунду глобально.

Advisory locks бывают session-scoped (держатся до конца сессии или явного unlock) и transaction-scoped (`pg_advisory_xact_lock`, автоматически освобождаются на COMMIT/ROLLBACK). Второй вариант обычно безопаснее — если приложение крашится не освободив, ничего не остаётся заблокированным.

Есть варианты shared vs exclusive advisory locks. `pg_advisory_lock` — exclusive, `pg_advisory_lock_shared` — shared. Работают по обычной read/write семантике.

## Как найти виновника в проде

Ты видишь: пул HikariCP исчерпан, запросы стоят по 30 секунд, приложение отвечает 503. Начинаешь диагностику.

Первый шаг — активные соединения и их состояния.

```sql
SELECT pid, state, wait_event_type, wait_event,
       NOW() - xact_start AS transaction_duration,
       NOW() - query_start AS query_duration,
       query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_start;
```

Ищешь запросы с большим `transaction_duration` или `wait_event_type = 'Lock'`. Если много сессий ждут Lock, где-то есть блокирующая транзакция.

Второй шаг — кто кого блокирует.

```sql
SELECT 
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocked.query AS blocked_query,
    blocking.query AS blocking_query,
    blocking.state AS blocking_state
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking 
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';
```

`pg_blocking_pids(pid)` возвращает массив pid'ов, блокирующих данную сессию. Через JOIN с `pg_stat_activity` получаешь их query и состояние.

Часто картина такая: одна транзакция висит в состоянии `idle in transaction`, её `query` показывает last executed statement (например обычный SELECT), но `xact_start` был час назад. Это значит: приложение сделало BEGIN, выполнило SELECT, потом ушло делать что-то другое (например HTTP call), и **забыло** commit/rollback. Транзакция открыта, держит какие-то локи, блокирует всех.

Чинить: убить эту транзакцию.

```sql
SELECT pg_cancel_backend(<pid>);  -- soft (SIGINT), обычно сработает
SELECT pg_terminate_backend(<pid>);  -- hard (SIGTERM), если cancel не помог
```

Долгосрочный fix: настроить `idle_in_transaction_session_timeout = 60s` в PostgreSQL. Тогда транзакции, простаивающие в состоянии `idle in transaction` больше 60 секунд, будут автоматически убиваться. Плюс в приложении — вылавливать паттерны, где BEGIN и COMMIT не в одном try-with-resources.

## Deadlocks

Deadlock — ситуация, когда две (или больше) транзакции ждут друг друга по кругу. T1 держит row A и хочет row B. T2 держит row B и хочет row A. Никто не может продвинуться.

PostgreSQL автоматически обнаруживает deadlocks. Каждые `deadlock_timeout` (по умолчанию 1 секунда) он проверяет граф ожиданий среди всех транзакций. Если находит цикл — выбирает жертву (обычно ту, что позже пришла) и убивает её с ошибкой `ERROR: deadlock detected`. Оставшиеся продолжают работать.

Пример типового deadlock'а:

```sql
-- Session 1                          -- Session 2
BEGIN;                                 BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
                                       UPDATE accounts SET balance = balance - 200 WHERE id = 2;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- waits for session 2's lock on id=2
                                       UPDATE accounts SET balance = balance + 200 WHERE id = 1;
                                       -- waits for session 1's lock on id=1
-- DEADLOCK DETECTED, one killed
```

Оба хотели перевести деньги между счетами 1 и 2, но взяли локи в разном порядке. Классика.

Как избежать. Первое правило — **consistent lock order**. Всегда захватывай ресурсы в одном и том же порядке. В примере выше — если обе транзакции сначала обновляют min(id_1, id_2), потом max — deadlock невозможен. В коде на Java это может быть реализовано как:

```java
Long firstId = Math.min(id1, id2);
Long secondId = Math.max(id1, id2);
account1 = accountRepo.findByIdForUpdate(firstId);
account2 = accountRepo.findByIdForUpdate(secondId);
```

Второе — **короткие транзакции**. Deadlock требует времени: две транзакции должны параллельно держать локи и параллельно запрашивать. Если каждая транзакция миллисекунды, вероятность deadlock'а мала. Долгие транзакции — deadlock waiting to happen.

Третье — **retry logic**. Deadlock — это transient ошибка, retry через несколько миллисекунд обычно проходит успешно (потому что «другая» транзакция уже закончилась). В Spring это делается через `@Retryable` на методах, где может возникнуть deadlock:

```java
@Retryable(value = {CannotAcquireLockException.class}, 
           maxAttempts = 3, 
           backoff = @Backoff(delay = 100, multiplier = 2))
@Transactional
public void transfer(...) { ... }
```

PostgreSQL при deadlock возвращает SQLSTATE 40P01, Spring переводит в `CannotAcquireLockException` (или `DeadlockLoserDataAccessException` в зависимости от версии).

## ALTER TABLE и почему он опасен

Как мы говорили, большинство ALTER TABLE берёт ACCESS EXCLUSIVE lock. Это критично: пока идёт ALTER, никто даже читать не может. И даже если сам ALTER быстрый — если приходится ждать открытых транзакций, ждут все за ним.

Одно из самых частых ловушек — `ALTER TABLE ADD COLUMN ... NOT NULL DEFAULT expr`. Раньше это было катастрофой: PostgreSQL перезаписывал все строки таблицы, чтобы установить default значение, что на большой таблице занимало часы под ACCESS EXCLUSIVE. С PostgreSQL 11 это исправлено: если default — простое непеременное значение (constant), таблица не перезаписывается, а информация о default'е хранится в метаданных. Быстро.

Но не всё изменения такие. Что реально безопасно:

`ADD COLUMN` без NOT NULL и без default — мгновенно.

`ADD COLUMN col type DEFAULT constant` (PG 11+) — быстро, метаданные.

`ADD COLUMN col type NOT NULL DEFAULT constant` (PG 11+) — быстро.

`DROP COLUMN` — быстро, колонка помечается как удалённая, реально удаляется при следующем VACUUM.

`ADD CHECK constraint NOT VALID` — быстро, но constraint пока не проверяется на существующих данных. Затем `VALIDATE CONSTRAINT` — берёт SHARE UPDATE EXCLUSIVE (менее агрессивный чем ACCESS EXCLUSIVE), медленно, но не блокирует запросы.

`ADD FOREIGN KEY ... NOT VALID` + `VALIDATE CONSTRAINT` — тот же паттерн.

Что опасно:

`ADD COLUMN NOT NULL` без DEFAULT — быстро (не перезапись), но требует ACCESS EXCLUSIVE.

`ADD COLUMN NOT NULL DEFAULT volatile_expr` — полная перезапись таблицы.

`ALTER COLUMN TYPE` — обычно полная перезапись, кроме некоторых binary-compatible преобразований.

`ADD CONSTRAINT` без `NOT VALID` — сканирование всей таблицы под ACCESS EXCLUSIVE.

`CREATE INDEX` без CONCURRENTLY — SHARE lock, блокирует DML на всё время построения индекса.

Для безопасных миграций в проде правило одно: **всегда `SET lock_timeout`** перед ALTER, чтобы если lock не получен быстро, миграция упала, не заблокировав систему. Всегда `CREATE INDEX CONCURRENTLY` в проде, а не обычный. Всегда `ADD CONSTRAINT NOT VALID` + отдельный `VALIDATE`. Всегда сначала `ADD COLUMN` nullable, потом backfill данных, потом установить NOT NULL — не одним шагом.

Если делаешь миграцию на большой таблице через Flyway/Liquibase, тестируй её на copy of prod, замеряй время, планируй окно. Не полагайся на «наверное быстро».

## Как читать EXPLAIN — базовое чтение плана

Переключаемся на планы. `EXPLAIN` показывает, как PostgreSQL собирается выполнять запрос. `EXPLAIN ANALYZE` реально выполняет запрос и показывает что случилось.

Простейший пример:

```sql
EXPLAIN SELECT * FROM users WHERE id = 5;
```

Вывод:

```
                              QUERY PLAN
------------------------------------------------------------------------
 Index Scan using users_pkey on users  (cost=0.29..8.30 rows=1 width=64)
   Index Cond: (id = 5)
```

Разберём что это значит. `Index Scan using users_pkey on users` — узел плана типа Index Scan, использует индекс `users_pkey` на таблице `users`. Это тип operator'а, реализующий доступ к данным через индекс.

`cost=0.29..8.30` — оценка стоимости. Первое число — стартовая стоимость (сколько работы нужно сделать, прежде чем вернуть первую строку). Второе — общая стоимость (для получения всех строк). Единица измерения — абстрактная, но привязана к стоимости чтения одной sequential-страницы (`seq_page_cost = 1.0` по умолчанию). Random-страница дороже (`random_page_cost = 4.0` для HDD, 1.1 для SSD).

Cost — это **оценка**, не фактическое время. Она нужна Optimizer'у для выбора между планами. Между собой планы сравниваются по cost'у, самый дешёвый выбирается. Но абсолютное значение cost не значит миллисекунды или секунды — это условная единица.

`rows=1` — оценка количества строк, которые вернёт этот узел. Оценивается на основе статистики: если primary key, значит уникальное значение, вернёт максимум 1 строку.

`width=64` — оценка среднего размера строки в байтах. Используется для оценки последующих операций (например, сортировки: если надо сортировать миллион строк по 100 байт, нужно 100 MB памяти).

`Index Cond: (id = 5)` — условие, применяемое непосредственно в индексе. Это важно: index cond находит правильные строки за одно спускание по дереву. Если бы было `Filter: (id = 5)`, это означало бы: сначала прочитать все строки, потом отфильтровать в памяти — гораздо хуже.

Теперь `EXPLAIN ANALYZE`:

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE id = 5;
```

Вывод:

```
                                              QUERY PLAN
--------------------------------------------------------------------------------------
 Index Scan using users_pkey on users  (cost=0.29..8.30 rows=1 width=64) 
                                        (actual time=0.023..0.024 rows=1 loops=1)
   Index Cond: (id = 5)
 Planning Time: 0.089 ms
 Execution Time: 0.048 ms
```

Появились дополнительные числа. `actual time=0.023..0.024` — реальное время в миллисекундах: до первой строки 0.023 мс, до всех 0.024 мс. `rows=1` — реально возвращено строк. `loops=1` — сколько раз выполнялся этот узел (для вложенных операций может быть больше 1).

`Planning Time` — сколько потратил Optimizer на построение плана. Для простых запросов доли миллисекунды, для сложных с многими join'ами может быть заметно.

`Execution Time` — сколько заняло реальное выполнение. Это то, что реально почувствовал бы клиент.

## Ключевое искусство: сравнение estimated vs actual

Одна из самых важных вещей при чтении EXPLAIN ANALYZE — сравнение оценки Optimizer'а (`rows` без actual) с реальностью (`rows` в actual блоке). Если они близки — Optimizer правильно понял ситуацию, план разумный. Если сильно отличаются — Optimizer работает вслепую, план может быть плохим.

Пример проблемы:

```
Seq Scan on huge_table (cost=... rows=100 width=...) (actual time=... rows=5000000 loops=1)
   Filter: (status = 'active')
   Rows Removed by Filter: 5000000
```

Optimizer оценил 100 строк, реально пришло 5 миллионов. 50000-кратная ошибка. Это значит: где-то устаревшая статистика. `ANALYZE huge_table;` может исправить.

Ещё частая проблема: Optimizer думает что joins дадут маленький результат, использует Nested Loop. Реально результат огромный, Nested Loop становится O(N*M) — часы. Правильный план был бы Hash Join.

Строка `Rows Removed by Filter: N` показывает, сколько строк было прочитано, но отфильтровано. Если это большое число — значит index не помог, пришлось читать много ненужного. Может стоит добавить или изменить индекс.

## BUFFERS — сколько I/O реально сделано

Добавляем `BUFFERS`:

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 12345;
```

Вывод содержит новые строки:

```
Index Scan using orders_user_id_idx on orders 
    (cost=... rows=100 width=...) 
    (actual time=... rows=87 loops=1)
   Index Cond: (user_id = 12345)
   Buffers: shared hit=95 read=3
```

`Buffers: shared hit=95 read=3` — этот узел работал с 98 страницами буфера. 95 из них были в shared_buffers (cache hit), 3 пришлось прочитать с диска (или из OS page cache, что тоже проходит как read).

`shared` — обычные страницы таблиц/индексов из shared_buffers.
`local` — временные локальные буферы (для temporary tables).
`temp` — temp files (spill сортировки/hashа).

Метрики:
- `hit` — cache hit в shared_buffers.
- `read` — cache miss, пошло на уровень OS.
- `dirtied` — эта операция сделала страницы dirty (модификации).
- `written` — сколько страниц было записано на диск.

Если видишь много `read` на запросе — либо cache холодный, либо запрос вычитывает больше данных чем есть в кэше. Часто повторение того же запроса второй раз даст `read=0` (данные уже в кэше), а первый запуск — `read=1000`. Это нормально, но если каждый запуск даёт много read, значит рабочий набор не помещается в shared_buffers.

Если видишь `temp read=` или `temp written=` — плохо. Значит какая-то операция (обычно Sort или Hash) не поместилась в work_mem и переехала на диск. Работает в разы медленнее. Либо увеличить work_mem, либо переписать запрос.

## Все основные типы узлов

Пройдёмся по узлам, которые встречаются в планах.

**Seq Scan** — последовательное чтение всех страниц таблицы. Читает всё, применяет фильтр, возвращает подходящие строки. Хорошо если нужна большая часть таблицы (>10-30%), плохо если 1%. `Filter` показывает, что фильтруется. `Rows Removed by Filter` — сколько было отброшено.

**Index Scan** — обход B-tree индекса плюс чтение соответствующих страниц heap. Каждая нужная строка = один random I/O в heap. Хорошо для селективных запросов. `Index Cond` — условие применяемое в индексе, `Filter` — дополнительный фильтр после чтения из heap.

**Index Only Scan** — как Index Scan, но не идёт в heap. Работает если все нужные колонки есть в индексе и страницы всех прочитанных строк отмечены как all_visible в Visibility Map. Существенно быстрее обычного Index Scan. `Heap Fetches` показывает сколько раз всё-таки пришлось сходить в heap (когда VM не позволил пропустить).

**Bitmap Index Scan + Bitmap Heap Scan** — гибрид. Сначала Bitmap Index Scan строит битмап страниц, содержащих нужные строки. Потом Bitmap Heap Scan читает эти страницы **в порядке возрастания page number** — превращая random access в sequential. Хорошо когда нужно много строк, разбросанных по таблице.

Пример:

```
Bitmap Heap Scan on orders (cost=... rows=50000)
   Recheck Cond: (status = 'active')
   -> Bitmap Index Scan on orders_status_idx (cost=... rows=50000)
        Index Cond: (status = 'active')
```

`Recheck Cond` появляется потому что битмап lossy при большом количестве строк (не хватило памяти запомнить конкретные строки, запомнил только страницы) — приходится перепроверять при чтении heap.

**Nested Loop** — join. Для каждой строки левой стороны выполняется правая. Хорошо когда левая маленькая и правая имеет индекс по join key. Плохо когда обе большие: O(N*M).

```
Nested Loop  (cost=... rows=100)
   -> Index Scan on users  (rows=1)
   -> Index Scan on orders (rows=100)
        Index Cond: (user_id = users.id)
```

Читаем: один user, для него 100 orders через индекс. Итого 100 строк. Быстро.

**Hash Join** — join через hash table. Строит hash по одной стороне (build side, обычно меньшая), сканирует другую сторону (probe side), для каждой строки делает lookup в hash. Хорошо для больших равных join'ов.

```
Hash Join  (cost=...)
   Hash Cond: (o.user_id = u.id)
   -> Seq Scan on orders o
   -> Hash  
      -> Seq Scan on users u
```

Читаем: строим hash по users (маленькая таблица), сканируем orders и для каждой строки ищем в hash.

**Merge Join** — join уже отсортированных потоков. Обе стороны сортируются (или уже отсортированы), потом сливаются как в merge sort. Хорошо когда обе стороны уже отсортированы по join key (например, обе имеют индекс).

**Sort** — сортировка. `Sort Method` показывает как: `quicksort` (в памяти), `external merge` (на диске), `top-N heapsort` (при LIMIT). `Sort Key` — по чему сортируется. Если видишь `external merge  Disk: 1234567kB` — сортировка не влезла в work_mem, стала медленной.

**Aggregate** — GROUP BY. `HashAggregate` использует hash table по grouping key, `GroupAggregate` требует уже отсортированный вход. Первый быстрее в общем случае.

**Materialize** — сохранение промежуточного результата в памяти для переиспользования (обычно в Nested Loop, когда правая сторона нужна много раз).

**Gather** и **Gather Merge** — parallel query. Запрос делится между несколькими worker'ами. Gather собирает результаты, Gather Merge — сохраняет порядок.

**CTE Scan** и **Subquery Scan** — обёртки над результатами CTE/подзапросов.

## Типичные плохие паттерны

Прочитав сотни планов, замечаешь повторяющиеся анти-паттерны, каждый из которых требует своей fix'ы.

**Sequential Scan там где должен быть Index Scan**. Смотришь план запроса с `WHERE id = 5` и видишь Seq Scan вместо Index. Причины: индекса нет, устаревшая статистика заставила Optimizer подумать что почти все строки подойдут под условие, тип колонки не match'ится с условием (например, колонка text, а сравниваешь с числом — implicit cast ломает использование индекса). Fix: создать индекс, ANALYZE, проверить типы.

**Nested Loop на большом результате**. Внутренний узел `rows=1000000, loops=100000` — значит 100 миллиардов операций. Optimizer выбрал Nested Loop, оценив что строк будет мало. Оценка ошиблась. Fix: часто ANALYZE помогает. Если нет — можно попробовать `SET enable_nestloop = off` временно, посмотреть какой план получится (обычно Hash Join), и оптимизировать через переписывание запроса.

**Sort с external merge на диск**. `Sort Method: external merge  Disk: 234567kB`. Означает что work_mem мал. Fix: `SET work_mem = '256MB'` для конкретной сессии перед тяжёлым запросом. Не увеличивай глобально — умножится на количество concurrent запросов.

**Rows Removed by Filter огромное**. Индекс есть, но фильтр читает много и выбрасывает. Значит index покрывает только часть условий. Fix: составной индекс, включающий все условия WHERE.

**Bitmap Heap Scan с Recheck**. Битмап lossy (не хватило памяти запомнить конкретные строки). Технически работает, но каждую страницу приходится перепроверять. Fix: увеличить work_mem.

**Estimated vs actual rows отличается в тысячи раз**. Optimizer работает вслепую. Fix: `ANALYZE table`, extended statistics для коррелированных колонок.

**JIT compilation на маленьких запросах**. Иногда видишь `JIT: Functions: 5, Options: Inlining true, ...` на быстром запросе. JIT добавил overhead компиляции, сам запрос быстрее не стал. Fix: увеличить `jit_above_cost` чтобы JIT включался только для тяжёлых запросов.

## Дополнительные параметры EXPLAIN

Кроме `ANALYZE` и `BUFFERS` есть ещё полезные опции.

`VERBOSE` — показывает полные имена (schema.table.column), список output колонок каждого узла. Полезно для сложных запросов с одинаковыми названиями таблиц через AS.

`COSTS` — включён по умолчанию, показывает cost. Можно выключить `COSTS false`, если не интересно.

`SETTINGS` — показывает non-default настройки, влияющие на этот план (work_mem, random_page_cost и т.д.). Полезно когда план исполняется в другой сессии с другими параметрами.

`FORMAT JSON` (или XML, YAML) — вывод в machine-readable формате. Удобно для парсинга инструментами вроде depesz.com или pev2.

Полный ANALYZE с всеми опциями:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS, FORMAT TEXT) 
SELECT ...;
```

Существует https://explain.depesz.com — сайт куда можно вставить план в текстовом виде и получить визуализацию: узлы, где сколько времени провёл, какие оценки соответствуют реальности. Полезно для сложных планов.

## Заключение

Locks и планы — два фундаментальных инструмента диагностики PostgreSQL в проде. Locks определяют concurrency: кто может делать что параллельно, кто должен ждать. Восемь режимов, матрица совместимости, FIFO очередь ожидания — понимая эти детали, можно предсказать поведение любой миграции или сложной транзакции. `SELECT FOR UPDATE SKIP LOCKED` превращает обычную таблицу в высокопроизводительную очередь. Advisory locks дают координацию между приложениями без специальных инструментов. Deadlocks — обычная часть жизни, нужен consistent order + короткие транзакции + retry logic.

Планы запросов — язык, на котором Optimizer объясняет свои решения. `EXPLAIN` показывает намерения, `EXPLAIN ANALYZE` — реальность, `BUFFERS` добавляет статистику I/O. Восемь основных типов узлов покрывают 95% планов: Seq Scan, Index Scan, Index Only Scan, Bitmap Scan, Nested Loop, Hash Join, Merge Join, Sort, Aggregate. Читать план надо от глубины к корню, обращая внимание на estimated vs actual (сильное расхождение = проблема статистики), buffers (много read = проблема кэша), Rows Removed by Filter (плохой индекс), external Sort (нехватка work_mem).

Когда в проде инцидент — начинай с двух запросов: `pg_stat_activity` с фильтром по wait_event и `pg_blocking_pids` для поиска блокирующих. Когда конкретный запрос медленный — прогони `EXPLAIN (ANALYZE, BUFFERS)`, найди самый дорогой узел, разберись почему он дорогой, применяй фикс.

Дальше — практика на реальных примерах. Открой pg_stat_statements своего проекта, возьми топ-10 по total_exec_time, прогоняй каждый через EXPLAIN ANALYZE. Замечай паттерны. Пробуй фиксить (добавлять индексы, переписывать запросы), замеряй улучшения. Каждый разобранный случай добавляет интуицию, которая приходит только через практику.
