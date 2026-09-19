# 92. Backup, восстановление и безопасные миграции

## Backup, который не восстанавливали, — не backup

Есть старое правило системного администрирования, звучащее банально, но постоянно нарушаемое: **backup, который не тестировали восстановлением, backup'ом не является**. Пока не убедился что можешь реально из этого backup'а восстановить работающую базу — у тебя не backup, а файл на диске, про который ты предполагаешь что он полезен. Прод-инциденты полны историй, где backup существовал, но восстановление не работало по десятку причин.

Backup нужен от трёх типов проблем. Первое — hardware failure: сгорел диск, погибла VM, отказал storage. Второе — human error: DBA запустил `DROP TABLE production_users` вместо `staging_users`, разработчик написал `UPDATE users SET email = '...'` без WHERE. Третье — software bugs: приложение в новой версии несколько часов ломало данные, пока не заметили. Каждый тип требует разной стратегии восстановления, и хороший backup-план покрывает все три.

В PostgreSQL есть несколько принципиально разных механизмов backup'а. Логический (`pg_dump`) — переносимый snapshot схемы и данных в текстовом или бинарном виде, работает медленно на больших БД но универсален. Физический (`pg_basebackup`) — байтовая копия файлов данных, быстрее и позволяет PITR, но привязан к версии PostgreSQL. WAL archiving — хранение всех WAL-файлов, позволяет восстанавливаться на любой момент времени. Инструменты вроде WAL-G, pgBackRest, Barman строят продвинутые решения поверх этих базовых механизмов.

Параллельная тема — безопасные миграции схемы. Правильный backup спасает от катастроф, но лучше не допускать катастроф. Многие production инциденты происходят из-за ALTER TABLE, применённого в неподходящий момент или неправильно. Понимание, какие DDL операции безопасны, какие блокируют систему, как их правильно упаковывать — часть базовой компетенции работы с production PostgreSQL.

## Логический backup: pg_dump

`pg_dump` — простейший инструмент. Он читает базу и выдаёт скрипт для её пересоздания: CREATE TABLE, CREATE INDEX, INSERT для данных. По сути — снимок содержимого базы в переносимом формате.

Простейшее использование:

```bash
pg_dump -h host -U user -d database > backup.sql
```

Или в более компактный custom формат:

```bash
pg_dump -h host -U user -d database -F c -f backup.dump
```

Формат `-F c` (custom) — бинарный, сжатый, позволяет параллельное восстановление и селективный restore (только определённые таблицы).

`pg_dump` работает через обычное соединение PostgreSQL. Он берёт snapshot начала работы (через SET TRANSACTION SNAPSHOT), потом читает данные — все читатели видят consistent state на момент начала dump. Транзакции, начатые после старта dump, дампу не видны.

Плюсы pg_dump. Работает с любой версией PostgreSQL как источником, восстанавливается в любую (обычно), включая переход между major версиями. Переносимо между платформами (Linux → Windows, x86 → ARM). Селективный — можно backup'ить одну схему или несколько таблиц. Читает через обычное соединение, не требует доступа к файловой системе — подходит для managed сервисов (RDS, Cloud SQL).

Минусы. Медленный: full scan всех таблиц, сериализация в текст/бинарь, файловый I/O. Для базы на 500 GB dump занимает часы. Медленный restore: воссоздание индексов, foreign keys, triggers — с нуля, что для больших таблиц очень долго. Не даёт incremental — только полный snapshot. Не подходит для баз в терабайты.

Есть `pg_dumpall` — обёртка, дампящая весь кластер (все databases, roles, tablespaces). Полезно для полного backup'а кластера.

Restore:

```bash
psql -h host -U user -d database < backup.sql
# или для custom формата:
pg_restore -h host -U user -d database backup.dump
```

Параллельный restore через `-j N` — N worker'ов работают одновременно, ускоряет для custom формата.

Практическое применение pg_dump. Backup маленьких БД (до сотни GB). Регулярный snapshot для dev/staging окружений. Экспорт данных для миграций (между версиями PostgreSQL, между платформами, между on-prem и cloud). Селективный backup конкретных таблиц перед рискованными изменениями.

Не подходит для: production backup больших БД, PITR восстановление на конкретное время, near-zero RTO/RPO требования.

## Физический backup: pg_basebackup и WAL archiving

Для production обычно используется физический backup. Это байтовая копия файлов кластера — включая все data files, WAL, конфиг. Восстановление — не пересоздание из SQL, а копирование файлов на новое место + запуск PostgreSQL, который сам приведёт всё в consistent state через WAL replay.

Основной инструмент — `pg_basebackup`. Он подключается к PostgreSQL как streaming replication клиент, копирует файлы кластера, параллельно принимая WAL сгенерированный во время копирования. Результат — backup, из которого можно восстановиться в consistent state на момент завершения копирования.

Простое использование:

```bash
pg_basebackup -h host -U replicator -D /backup/base -Fp -Xs -P -R
```

`-D` — куда сохранять. `-Fp` — plain format (директория с файлами). `-Xs` — stream WAL параллельно (необходимо для consistent backup). `-P` — progress reporting. `-R` — записать recovery config.

Плюсы физического backup'а. Быстрее pg_dump на больших БД (просто копирование файлов). Восстановление тоже быстрое — не нужно пересоздавать индексы. Основа для PITR (point-in-time recovery).

Минусы. Backup и восстановление на **точно ту же major версию** PostgreSQL. Не переносимо между платформами (иногда, зависит от byte order и alignment). Требует доступа к файловой системе или replication connection.

Одиночный pg_basebackup — это snapshot на один момент. Для PITR нужен ещё **WAL archiving**: PostgreSQL сохраняет все WAL-файлы после того как они завершены. При восстановлении можно применить WAL до нужного момента, получая состояние базы на любую точку времени между base backup и последним WAL-файлом.

Настройка WAL archiving:

```
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /wal_archive/%f && cp %p /wal_archive/%f'
```

`archive_command` — команда, которую PostgreSQL вызывает для каждого завершённого WAL-файла. `%p` — путь к файлу, `%f` — только имя. В простом случае — копирование. В реальном проде — WAL-G, который загружает в S3.

При PITR восстановлении: копируешь base backup в место восстановления, конфигурируешь `restore_command` для получения WAL из архива, устанавливаешь `recovery_target_time = '2024-01-15 14:30:00'`, запускаешь. PostgreSQL восстановится, применит WAL до указанного времени и остановится в этой точке.

## Продвинутые инструменты: WAL-G, pgBackRest, Barman

Ручное управление pg_basebackup + WAL archiving — сложно, ошибкоопасно, не масштабируется. Инструменты автоматизируют весь workflow.

**pgBackRest** — самый популярный в enterprise. Написан на Perl+C, работает как отдельный процесс. Умеет parallel backup (несколько worker процессов копируют разные части параллельно), incremental и differential backups (только изменения с прошлого раза), compression и encryption, backup verification, integration с cloud storage.

Простая настройка pgBackRest в конфиге:

```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=7
repo1-cipher-type=aes-256-cbc

[main]
pg1-path=/var/lib/postgresql/data
```

Ежедневный full backup:

```bash
pgbackrest --stanza=main backup --type=full
```

Restore на point in time:

```bash
pgbackrest --stanza=main --type=time \
    --target='2024-01-15 14:30:00' restore
```

**WAL-G** — более легковесная альтернатива, написана на Go. Оптимизирована под cloud storage (S3, GCS, Azure Blob). Меньше фич чем pgBackRest, но проще и достаточно для большинства.

**Barman** — итальянское решение, работает как централизованный backup server, забирает backups с всех PostgreSQL инстансов через сеть. Хорошо для случая, когда управляешь много кластеров.

Выбор инструмента зависит от контекста. Для нового проекта на AWS — WAL-G с backup в S3 хорош. Для энтерпрайза с on-prem storage — pgBackRest. Все три — надёжные, хорошо поддерживаемые проекты.

## Стратегия backup: полные, incremental, retention

Правильная стратегия — компромисс между RTO (recovery time objective, как быстро восстановиться), RPO (recovery point objective, сколько данных можно потерять), стоимостью хранения, нагрузкой на прод.

Типичная prod-стратегия:

**Weekly full backup**. Раз в неделю (обычно в выходной с низкой нагрузкой) — полный backup. Это база, от которой всё дальнейшее строится.

**Daily differential** или **incremental**. Ежедневно — только изменения с прошлого раза. Быстрее полного, меньше места.

**WAL streaming/archiving непрерывно**. Все WAL-файлы уезжают в архив. Позволяет PITR на любую секунду.

**Retention**. Обычно: 4 недели полных backup'ов, 3 месяца еженедельных, 1 год ежемесячных. Baseline для типичных требований compliance.

Разные типы данных требуют разных RPO. Финансовые транзакции — не терять ни секунды (synchronous replication + WAL archiving + backup). Аналитические данные — можно потерять час (async replication достаточно). Логи приложения — можно потерять день. Backup стратегию нужно строить исходя из этого.

Одна забываемая практика — регулярное **testing restore**. Раз в месяц или квартал брать backup и восстанавливать в отдельное окружение, проверять что база стартует, что данные консистентны, что нет побитых страниц. Это единственный способ обнаружить проблемы в backup process до того как они станут критичны.

## Point-in-time recovery в деталях

PITR — самая гибкая форма восстановления. Пример сценария: разработчик в 14:32 запустил в проде `DELETE FROM users WHERE last_login < '2024-01-01'` — вместо `UPDATE users SET active = false WHERE ...`. Удалилось 500 тысяч важных записей. Backup ежедневный, последний был в 4 утра.

Что делать. Без PITR: восстанавливать из вчерашнего backup, теряя всё что было сегодня до 14:32. Плохо.

С PITR: восстанавливаем базу на 14:31 (за минуту до катастрофы). Все изменения с 4 утра до 14:31 сохранены. Потеря — только та минута.

Как выглядит:

Шаг 1. Обычно PITR восстановление делается в **отдельное окружение**, не в prod напрямую. Слишком опасно затирать текущую базу.

Шаг 2. Копируешь base backup в новое место. Конфигурируешь restore parameters:

```
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2024-01-15 14:31:00'
recovery_target_action = 'pause'
```

`pause` означает — по достижении target time приостановиться (не promote в writable). Даёт возможность проверить состояние базы.

Шаг 3. Стартуешь PostgreSQL. Он читает WAL из архива, применяет до целевого времени, останавливается. Ты подключаешься, проверяешь — users на месте, данные consistent.

Шаг 4. Если всё хорошо — извлекаешь удалённые записи (например, через pg_dump конкретной таблицы), возвращаешь их в prod. Если что-то не так — retry с другим target time.

Тонкости. `recovery_target_time` работает по последней committed транзакции до этого времени, но нельзя гарантировать что она включит именно 14:30:59.999 а не 14:30:58. Есть также `recovery_target_lsn` (по конкретному LSN) и `recovery_target_xid` (по конкретному XID) — точнее.

`recovery_target_inclusive` — включать ли транзакцию точно на target или только до неё.

`recovery_target_action` может быть `pause`, `promote` (сразу переключить в writable), `shutdown`.

После восстановления база находится на **новом timeline** — parallel history, отделившийся от исходного. Если попробовать применить дальнейший WAL с исходного timeline, будут конфликты. Это защита от случайного смешивания истории.

## Verify — проверка целостности backup

Одна из самых недооценённых практик — **регулярная проверка** что backup можно восстановить. Как минимум:

Проверка что base backup читается: `pg_verify_checksums` или встроенный `pg_checksums --check`. Проверяет CRC каждой страницы.

Test restore в отдельное окружение. Автоматизировано, регулярно (раз в неделю или месяц). Проверить что PostgreSQL стартует, что sanity queries работают.

Verify консистентность данных. `pg_amcheck` (PG 14+) — проверка B-tree индексов на соответствие heap.

Random sample sanity. Нахуй проверять таблицу целиком — cross-check несколько случайных row по бизнес-логике: балансы правильные, foreign keys не нарушены.

Backup, который не тестировали, статистически имеет 20-30% шанс быть неработоспособным по разным причинам: битый архив, забыли WAL сегмент, изменения в PostgreSQL версии, coeruptedchecksum. Testing — единственный способ обнаружить.

## Безопасные миграции: базовые правила

Переходим от backup к другой практической теме — как менять схему БД без положения прода. Плохие миграции — вторая по частоте причина инцидентов после плохих запросов.

Базовое правило: **любая DDL операция на большой таблице потенциально опасна**. Даже если сама команда быстра, она может ждать lock'а, а за ней встанут все SELECT'ы. Как разбирали в файле про locks — ACCESS EXCLUSIVE lock блокирует всё, включая читателей.

Всегда используй `lock_timeout` перед миграциями:

```sql
SET lock_timeout = '3s';
ALTER TABLE users ADD COLUMN new_col TEXT;
```

Если ALTER не может получить lock за 3 секунды — операция падает с ошибкой, не блокируя ничего. Retry позже, когда таблица менее занята.

`statement_timeout` — тоже полезно для длительных миграций. Не даёт ALTER работать больше N минут.

Правило второе — **каждую миграцию тестируй на копии prod**. Не полагайся на «наверное быстро». Замерь реальное время. Если 30 минут — планируй maintenance window.

Правило третье — **делай изменения обратимыми**. Каждая миграция должна иметь rollback скрипт. Что если после deployment окажется что новая колонка ломает старое приложение? Быстрый rollback критичен.

## Опасные и безопасные ALTER

Перечислим типичные операции, какие безопасны и какие нет.

**Безопасные (быстро, минимум блокировок)**.

`ADD COLUMN` без DEFAULT и без NOT NULL — просто добавляет колонку в метадату, файлы данных не трогает. Мгновенно.

`ADD COLUMN col type DEFAULT constant` в PostgreSQL 11+. Default хранится в метаданных, существующие строки виртуально получают его при чтении. Быстро.

`ADD COLUMN col type NOT NULL DEFAULT constant` — комбинация выше, тоже быстро.

`DROP COLUMN` — метаданные, колонка помечается как удалённая. Физически данные удаляются при следующем VACUUM или UPDATE.

`ADD CHECK constraint NOT VALID` — constraint добавлен, но проверка на существующих данных не запускается. Быстро. Потом `VALIDATE CONSTRAINT` под менее агрессивным lock'ом.

`ADD FOREIGN KEY ... NOT VALID` + `VALIDATE CONSTRAINT` — тот же паттерн.

`CREATE INDEX CONCURRENTLY` — построение индекса без блокировки writes. Занимает дольше обычного CREATE INDEX (несколько проходов таблицы), но не мешает работе. Единственный правильный способ в проде.

**Опасные (долго, ACCESS EXCLUSIVE lock)**.

`ADD COLUMN NOT NULL` без DEFAULT — быстро, но требует ACCESS EXCLUSIVE и заставляет проверить все строки на NULL (implicitly). На большой таблице — минуты.

`ADD COLUMN NOT NULL DEFAULT volatile_expression` — например, `DEFAULT random()`. PostgreSQL не может использовать оптимизацию с метаданными (значения разные для каждой строки), приходится переписать всю таблицу. Часы для большой.

`ALTER COLUMN TYPE` — почти всегда переписывает всю таблицу. Даже безобидные преобразования (VARCHAR(50) → VARCHAR(100)) в старых версиях делали rewrite. В PG 9.2+ некоторые binary-compatible преобразования быстрые.

`ADD CONSTRAINT` без `NOT VALID` — сканирование всей таблицы для проверки, под ACCESS EXCLUSIVE.

`CREATE INDEX` без CONCURRENTLY — SHARE lock блокирует все INSERT/UPDATE/DELETE. На большой таблице — час блокировки записей.

`REINDEX` без CONCURRENTLY — ACCESS EXCLUSIVE, блокирует всё.

`ALTER TABLE ... SET LOGGED/UNLOGGED` — переписывает всю таблицу.

`VACUUM FULL` — переписывает таблицу.

`CLUSTER` — переписывает таблицу в порядке индекса.

## Правильный паттерн: expand-contract

Для сложных изменений схемы используется паттерн **expand-contract**. Идея: не менять существующие структуры за один шаг, а расширить (add new alongside old) → переключить приложение → удалить старое.

Пример: переименование колонки `email` в `contact_email`.

Плохой путь: `ALTER TABLE users RENAME COLUMN email TO contact_email`. Быстро, но требует одновременного обновления кода приложения. Между моментом deployment миграции и моментом deployment кода — база в состоянии где `email` уже не существует, а код ещё его использует. Downtime.

Хороший путь (expand-contract):

Шаг 1 (expand): `ALTER TABLE users ADD COLUMN contact_email TEXT`. Быстро, безопасно.

Шаг 2: обновить код чтобы писал в **обе** колонки. Deploy.

Шаг 3: скопировать существующие данные из email в contact_email. Batch UPDATE:

```sql
UPDATE users SET contact_email = email 
WHERE id BETWEEN 1 AND 10000 AND contact_email IS NULL;
-- ... следующие batches
```

Шаг 4: обновить код чтобы читал из contact_email. Deploy.

Шаг 5: обновить код чтобы не писал больше в email. Deploy.

Шаг 6 (contract): `ALTER TABLE users DROP COLUMN email`. Финальная очистка.

Долго — но zero-downtime. Каждый deploy маленький, откатываемый. Даже если между шагами кто-то заметит проблему, catch-forward или rollback просты.

Для добавления NOT NULL constraint аналогично:

```sql
-- Шаг 1: добавить колонку nullable
ALTER TABLE users ADD COLUMN age INT;

-- Шаг 2: backfill в batches
UPDATE users SET age = calc_age(birth_date) 
WHERE id BETWEEN 1 AND 10000 AND age IS NULL;
-- ... следующие batches

-- Шаг 3: добавить constraint как NOT VALID
ALTER TABLE users ADD CONSTRAINT users_age_not_null CHECK (age IS NOT NULL) NOT VALID;

-- Шаг 4: validate constraint (менее агрессивный lock)
ALTER TABLE users VALIDATE CONSTRAINT users_age_not_null;

-- Шаг 5 (опционально): преобразовать check constraint в NOT NULL column property
-- В PG 12+ это делается автоматически при DROP CHECK + ALTER COLUMN NOT NULL
```

## Foreign key с NOT VALID

Добавление foreign key на большую таблицу занимает много времени под ACCESS EXCLUSIVE (сканирование всей таблицы для проверки). Правильно:

```sql
-- Шаг 1: добавить FK как NOT VALID (быстро, минимум блокировок)
ALTER TABLE orders ADD CONSTRAINT orders_user_fk 
    FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;

-- Шаг 2: validate — сканирует таблицу под SHARE UPDATE EXCLUSIVE
--        (совместим с обычным DML, только блокирует другие DDL)
ALTER TABLE orders VALIDATE CONSTRAINT orders_user_fk;
```

Между шагами constraint частично работает: **новые** INSERT/UPDATE проверяются на референциальную целостность. Только **существующие** строки не проверены до VALIDATE. Это обычно приемлемо: новые данные защищены, старые данные валидируются медленно в фоне.

## CREATE INDEX CONCURRENTLY

Одна из самых важных практических возможностей — построение индексов без блокировки writes. Обычный `CREATE INDEX` берёт SHARE lock, блокируя все INSERT/UPDATE/DELETE. На большой таблице — час недоступности записей.

`CREATE INDEX CONCURRENTLY` работает через два прохода таблицы, между ними принимает обычную нагрузку. Медленнее обычного CREATE INDEX (в 2-3 раза), но не блокирует.

Ограничения. Нельзя внутри транзакции (CREATE INDEX CONCURRENTLY autocommit'ит сам). Не может делать SET DEFAULT одновременно. Если сборка индекса упала на середине — оставит **invalid index**, который нужно вручную DROP и создавать заново.

Проверить invalid indexes:

```sql
SELECT indexrelid::regclass, indrelid::regclass 
FROM pg_index WHERE indisvalid = false;
```

Правило прода: **всегда CONCURRENTLY**. Даже если знаешь что таблица маленькая. Дисциплина.

## pg_repack как non-blocking VACUUM FULL

Таблица сильно раздулась от bloat'а. `VACUUM FULL` вернёт место, но требует ACCESS EXCLUSIVE lock на всё время работы — часы недоступности.

`pg_repack` (extension) делает аналогичную работу без блокировок. Внутри он: создаёт новую пустую таблицу с той же структурой, устанавливает trigger на исходную для отслеживания изменений, копирует данные из старой в новую, применяет накопленные изменения, атомарно swap'ает имена, удаляет старую таблицу.

Использование:

```bash
pg_repack -d knp -t orders
```

Работает часами на большой таблице, но приложение всё это время в норме (за исключением моментов ATO EXCLUSIVE на самом swap'е — миллисекунды). Требует свободное место на диске равное размеру исходной таблицы. Стандартный инструмент для реального сжатия таблиц в prod.

Есть также опция repack'ать только индексы: `pg_repack --only-indexes`. Полезно, когда индексы раздуты сильнее чем сама таблица.

## Migration frameworks

В приложениях миграции обычно управляются через фреймворки: Flyway, Liquibase (Java-мир), Alembic (Python), migrate (Go). Они организуют миграции как последовательность версионированных скриптов, отслеживают что применено в базе через специальную табличку (обычно `schema_version` или подобная), позволяют откатывать.

Flyway структура:
- `V1__create_users.sql`
- `V2__add_email_column.sql`
- `V3__create_orders.sql`

При старте приложения Flyway проверяет свою табличку, находит непримененные скрипты, применяет по порядку.

Liquibase более гибкий: изменения описываются в XML/YAML/JSON или SQL, поддерживает rollback скрипты нативно.

Правила использования migration frameworks в проде:

**Каждая миграция — атомарная и обратимая**. Один DDL step, легко откатить. Не делай десять ALTER в одном скрипте — если один упадёт, состояние непонятное.

**Не меняй уже применённые миграции**. Никогда не редактируй V5 после того как она задеплоилась в прод. Ошибку исправляй новой миграцией V6.

**Тестируй миграции на copy of prod**. Локальная база — не показатель. Прод-подобная нагрузка выявит проблемы, которых нет в dev.

**Используй `SET lock_timeout` внутри каждой миграции**. Уже говорили выше.

**Крупные data migrations — не через фреймворк**. Backfill миллионов строк должен быть отдельным скриптом, запускаемым в контролируемое окно. Migration framework — для быстрых DDL.

Flyway и Liquibase используют advisory locks в PostgreSQL для координации. При одновременном старте нескольких инстансов приложения — только один применяет миграции, остальные ждут его или пропускают.

## Заключение

Backup и recovery — не «служебная тема», а фундамент надёжности. Три типа проблем — hardware failure, human error, software bug — требуют разных стратегий. Логический backup (pg_dump) — переносимый, для маленьких БД. Физический backup (pg_basebackup + WAL archiving) — быстрый, для production. PITR — восстановление на любую точку времени через WAL replay поверх base backup.

Продвинутые инструменты (pgBackRest, WAL-G, Barman) автоматизируют workflow: full + incremental + differential + WAL streaming + retention + verification. Cloud storage интеграция. Test restore регулярно — единственный способ убедиться что backup работает.

Безопасные миграции — вторая критическая тема. ALTER TABLE может парализовать прод, если делать без осторожности. Всегда `lock_timeout`. Всегда CONCURRENTLY для CREATE INDEX. Constraint через NOT VALID + VALIDATE. Expand-contract паттерн для сложных изменений: расширить (nullable колонка) → backfill (batches) → переключить приложение → удалить старое.

Миграционные фреймворки (Flyway, Liquibase) организуют миграции как версионированные скрипты. Каждая маленькая и атомарная. Тестировать на copy of prod. Крупные data migrations — отдельные скрипты в maintenance windows, не через framework.

Для КНП правильная стратегия: pg_basebackup каждый день (или чаще), WAL archiving непрерывно в S3-совместимое хранилище (через WAL-G или pgBackRest), retention 30 дней ежедневных + 12 месяцев ежемесячных. Test restore каждый месяц в отдельном окружении. Все миграции через Liquibase с lock_timeout, все индексы CONCURRENTLY, все constraints через NOT VALID + VALIDATE, expand-contract для breaking changes.

Дальше — практика. Настрой pg_basebackup + WAL-G у себя (можно локально). Сделай backup, потом «сломай» базу (DROP TABLE случайной), восстанови через PITR на минуту до drop. Прогони все типы ALTER на большой таблице (миллион строк), замерь время каждого. Каждый час эксперимента даёт больше уверенности в проде, чем недели чтения документации.
