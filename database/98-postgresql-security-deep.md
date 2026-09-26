# 98. Безопасность PostgreSQL: RLS, роли, шифрование, аудит

## Почему безопасность БД — отдельная большая тема

Приложение обычно защищается на уровне authentication и authorization — user залогинился, роль проверена, endpoint доступен. Это верхний уровень. Но между приложением и базой данных существует ещё несколько слоёв защиты, о которых часто не думают. Пользователь БД, от имени которого приложение работает — какие у него права? Что произойдёт, если этот пользователь скомпрометирован? Как защитить данные если сервер БД украли физически? Как узнать, кто и когда изменил критические данные?

Для государственных систем вроде КНП security — не опция, а обязательное требование compliance. Персональные данные налогоплательщиков подпадают под регуляции. Финансовые данные — под аудит. Любое несанкционированное изменение или утечка — юридические последствия. Поэтому security на уровне БД должен быть спроектирован, а не добавлен post-factum.

В этом файле разберём все ключевые механизмы. Как устроена система ролей и привилегий PostgreSQL, чем role отличается от user, как правильно организовать permission model для приложения. Row-Level Security (RLS) — таких мощная фича для multi-tenant систем и compliance. Шифрование: at rest (данные на диске), in transit (TLS соединения), column-level (отдельные поля). Password policies и best practices аутентификации. Audit logging через pgAudit и другие механизмы. Data masking для non-prod окружений.

## Роли и привилегии

PostgreSQL не имеет отдельного понятия "user". Всё — **роли** (roles). Роль может логиниться в базу (тогда она functionally как user), может владеть объектами, может иметь дочерние роли (тогда она functionally как group), или всё вместе. `CREATE USER foo` — это синоним `CREATE ROLE foo WITH LOGIN`.

Это упрощает model: одна абстракция вместо users и groups. Один и тот же механизм работает для всего.

Ключевые атрибуты роли (задаются при создании):

**LOGIN** — может ли роль подключаться к БД. Без этого атрибута роль ничем не отличается от group.

**SUPERUSER** — обходит все проверки прав. Единственный атрибут, дающий unlimited access. Крайне опасен, используется только для admin.

**CREATEDB** — может создавать новые databases.

**CREATEROLE** — может создавать и управлять другими ролями.

**REPLICATION** — может использоваться для streaming replication.

**PASSWORD** — пароль для аутентификации (если LOGIN).

Простое создание:

```sql
CREATE ROLE app_user WITH LOGIN PASSWORD 'secret123';
CREATE ROLE app_read;  -- group role, без LOGIN
GRANT app_read TO app_user;
```

Теперь app_user наследует все привилегии app_read. Group-based organization: гранты на group, users в group наследуют.

## Привилегии на объекты

Существует три уровня привилегий: **database-level** (подключение к БД), **schema-level** (использование schema), **object-level** (SELECT/INSERT/UPDATE/DELETE на таблицы, EXECUTE на функции).

Правильная организация. Каждое приложение имеет свою роль. Роль имеет минимальные необходимые привилегии — принцип **least privilege**. Никаких SUPERUSER для приложения. Никаких OWNER'ов таблиц у application role.

Типичный setup для КНП-подобного сервиса:

```sql
-- Owner роль для DDL операций и миграций
CREATE ROLE knp_owner WITH LOGIN PASSWORD '<strong>';

-- App роль для runtime
CREATE ROLE knp_app WITH LOGIN PASSWORD '<strong>';

-- Read-only роль для reporting
CREATE ROLE knp_readonly WITH LOGIN PASSWORD '<strong>';

-- Создаём БД от owner'а
CREATE DATABASE knp OWNER knp_owner;

-- Подключаемся как owner, создаём схему
\c knp knp_owner
CREATE SCHEMA app;

-- Даём app роли необходимые права
GRANT USAGE ON SCHEMA app TO knp_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO knp_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA app TO knp_app;

-- Default privileges для новых таблиц (создаваемых потом)
ALTER DEFAULT PRIVILEGES IN SCHEMA app 
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO knp_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA app 
    GRANT USAGE, SELECT ON SEQUENCES TO knp_app;

-- Readonly роль
GRANT USAGE ON SCHEMA app TO knp_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA app TO knp_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA app 
    GRANT SELECT ON TABLES TO knp_readonly;
```

Что критично. Owner роль используется **только для миграций** (Liquibase/Flyway работают под этой ролью). Application код подключается как knp_app — не имеет прав создавать/удалять таблицы, менять структуру. Даже при SQL injection не сможет DROP TABLE или ALTER.

Отдельная readonly роль для BI, reports, аналитиков — они физически не могут ничего изменить.

## Password policies

Пароли для application ролей — не должны быть в коде или git. Хранение через environment variables, secret managers (Vault, AWS Secrets Manager), K8s secrets.

Для PostgreSQL native password authentication методов несколько. Раньше стандарт был `md5` — сохраняется хеш пароля в pg_authid. Уязвим к collision attacks и не позволяет defense-in-depth.

С PostgreSQL 10+ рекомендуется **SCRAM-SHA-256**. Salted challenge-response, значительно надёжнее md5. Настраивается через `password_encryption = scram-sha-256` в конфиге. Все новые пароли хешируются SCRAM, старые md5 постепенно rotate'ятся.

Настройка в pg_hba.conf (файл контроля access):

```
# TYPE  DATABASE  USER       ADDRESS         METHOD
host    knp       knp_app    10.0.0.0/8      scram-sha-256
host    knp       knp_owner  10.0.0.10/32    scram-sha-256
```

Каждая строка — правило: с каких IP разрешено подключение какой роли к какой БД каким методом.

Более сильная альтернатива — **certificate-based auth** через SSL client certificates. Приложение имеет client cert, при подключении PostgreSQL проверяет cert против CA. Не пароль, а cryptographic identity. Настройка сложнее, но надёжнее для критичных систем.

```
hostssl  knp    knp_app    10.0.0.0/8    cert
```

## Ротация паролей

Пароли должны периодически меняться. Проблема: приложение подключается с текущим паролем, во время ротации нельзя допустить downtime.

Правильный процесс с двумя ролями:

```sql
-- Основная роль
CREATE ROLE knp_app_v1 LOGIN PASSWORD '<old>';
GRANT ... TO knp_app_v1;

-- Новая роль на смену
CREATE ROLE knp_app_v2 LOGIN PASSWORD '<new>';
GRANT ... TO knp_app_v2;

-- Приложение постепенно переключается на v2
-- Когда весь трафик на v2:
DROP ROLE knp_app_v1;
```

Или через `ROLLBACK PASSWORD` mechanism с secret managers, где пароль обновляется прозрачно.

Для КНП правильно: пароли rotate'ятся автоматически каждые 90 дней через secret rotation в Vault или AWS Secrets Manager. Приложение получает актуальный пароль при подключении, не хранит константу.

## Row-Level Security (RLS)

Одна из самых мощных фич PostgreSQL для security. Позволяет **на уровне строк** ограничивать, какие данные видит и может изменять каждый пользователь.

Классический use case — multi-tenancy. У тебя SaaS система, много компаний-клиентов, каждая видит только свои данные. Обычно это делают через WHERE clauses в приложении: `SELECT * FROM orders WHERE tenant_id = ?`. Работает, но опасно: одна ошибка разработчика — WHERE забыт — и utente видит чужие данные.

RLS переносит эту защиту на уровень БД. Даже если приложение забудет WHERE, PostgreSQL сам добавит фильтр и не покажет чужие строки.

Настройка. Включаем RLS на таблице:

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
```

Создаём policy:

```sql
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::BIGINT);
```

`USING` — условие которое должны удовлетворять строки чтобы быть видимыми. Здесь: строка видима если её tenant_id совпадает с текущим tenant'ом.

Приложение при подключении устанавливает текущий tenant:

```sql
SET app.tenant_id = '5';
```

Теперь любой `SELECT * FROM orders` неявно превращается в `SELECT * FROM orders WHERE tenant_id = 5`. UPDATE и DELETE тоже фильтруются. Даже INSERT — можно требовать чтобы новые строки удовлетворяли policy (через WITH CHECK):

```sql
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::BIGINT)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::BIGINT);
```

Тогда `INSERT INTO orders VALUES (..., tenant_id=999)` при `SET app.tenant_id = '5'` — падает.

Несколько policies на одной таблице комбинируются: OR по умолчанию (any policy позволяет — доступ), AS RESTRICTIVE — все policies должны позволить.

Использования RLS.

**Multi-tenancy**. Как выше.

**Разделение доступа по департаментам**. `USING (department_id IN (SELECT department_id FROM user_departments WHERE user_id = current_user_id()))`.

**Публичные vs приватные записи**. `USING (public OR user_id = current_user_id())`.

**Compliance requirements**. Регуляторные требования на разделение данных.

Тонкости RLS. Superuser bypass'ит все policies. Owner таблицы тоже (по умолчанию), для его подключения RLS не работает. Установить `ALTER TABLE ... FORCE ROW LEVEL SECURITY` чтобы и owner подчинялся.

Performance impact. Policy — это additional WHERE в каждом запросе. Если policy сложна (JOIN'ит другие таблицы), performance страдает. Хорошая практика: partition key для policy — простые сравнения по индексированной колонке.

## Column-level security

Помимо RLS есть **column-level permissions**. Разные роли видят разные колонки.

```sql
GRANT SELECT (id, name, email) ON users TO regular_role;
GRANT SELECT (id, name, email, salary, ssn) ON users TO hr_role;
```

regular_role видит id, name, email. hr_role видит всё включая salary и ssn. Попытка regular_role сделать `SELECT ssn FROM users` — ошибка permission.

Для сенситивных данных (номера карт, ssn, пароли) — стандартный подход.

## Шифрование: at rest

Данные на диске могут быть украдены физически (диск/сервер) или через unauthorized access к файловой системе. Encryption at rest защищает: без ключа зашифрованные данные нельзя прочитать.

PostgreSQL сам по себе не имеет built-in encryption at rest. Есть несколько способов реализации.

**Filesystem-level encryption**. Linux Full Disk Encryption через LUKS, ZFS encryption. Прозрачно для PostgreSQL. Работает на уровне ОС, минимальный performance overhead с современными CPU (AES-NI hardware acceleration).

Плюсы. Простота — установка на этапе провижнинга сервера, для PostgreSQL полностью transparent. Стандарт compliance для многих регуляций.

Минусы. Не защищает от authorized access к работающей системе — если кто-то получит SSH root, данные видны как обычно. Ключ хранится и загружается на boot, если сервер работает — ключ в памяти.

**Cloud provider encryption**. AWS EBS encryption, GCP persistent disk encryption. Аналог LUKS, но managed. Ключи в KMS. Enable one flag при создании volume. Recommended для managed services.

**pgcrypto для column-level**. Отдельные критически важные колонки шифруются на уровне PostgreSQL:

```sql
CREATE EXTENSION pgcrypto;

INSERT INTO users(email, ssn_encrypted) VALUES (
    'a@b.com', 
    pgp_sym_encrypt('123-45-6789', current_setting('app.encryption_key'))
);

SELECT id, email, pgp_sym_decrypt(ssn_encrypted::bytea, current_setting('app.encryption_key')) 
FROM users;
```

Плюсы. Гранулярно — только для чувствительных полей. Даже DBA с superuser не может прочитать без ключа. Compliance friendly для PCI-DSS, HIPAA.

Минусы. Ключ management — где хранить ключ? Если ключ вместе с данными, encrypted at rest только protects от физической кражи. Настоящая защита требует external key management (KMS). Performance overhead (encryption/decryption на каждый read/write). Нельзя использовать encrypted колонки в WHERE, JOIN, index — они выглядят как случайные bytes.

Практика для КНП. Filesystem encryption обязательно (LUKS для on-prem, EBS для cloud). Column-level encryption для особо критичных полей (пароли — хешированы, финансовые данные — encrypted with pgcrypto + Vault-managed keys).

## Шифрование: in transit

Данные между приложением и БД тоже нужно защищать. Ethernet packets могут быть перехвачены (даже внутри дата-центра — insider threats).

Решение — TLS/SSL для всех подключений. PostgreSQL поддерживает нативно.

Настройка на сервере (postgresql.conf):

```
ssl = on
ssl_cert_file = '/etc/postgresql/server.crt'
ssl_key_file = '/etc/postgresql/server.key'
ssl_ca_file = '/etc/postgresql/root.crt'
```

Force SSL для всех подключений через pg_hba.conf:

```
hostssl  all  all  0.0.0.0/0  scram-sha-256
host     all  all  0.0.0.0/0  reject  # non-SSL — reject
```

Приложение (JDBC):

```
jdbc:postgresql://host:5432/db?ssl=true&sslmode=verify-full&sslrootcert=/path/to/root.crt
```

**sslmode**:
- `disable` — no SSL.
- `prefer` — SSL если возможно, plaintext иначе (небезопасно).
- `require` — SSL обязательно, но server cert не проверяется.
- `verify-ca` — проверить cert против CA.
- `verify-full` — проверить и cert, и hostname. Recommended.

Для prod: `verify-full`. Иначе MITM возможен через self-signed cert.

Certificate-based client authentication (уже упоминали) добавляет второй уровень: не только сервер аутентифицирует себя клиенту, но и клиент — серверу.

## Audit logging

Compliance часто требует audit trail: кто, когда, какие данные читал или изменил.

PostgreSQL из коробки логирует statements (log_statement = 'all' — все SQL команды), но это грубо: тонны логов, performance impact, sensitive data в plaintext логах.

**pgAudit** — extension для structured, конфигурируемого audit logging. Логирует по классам операций (READ, WRITE, DDL, ROLE), только для указанных таблиц или пользователей.

```sql
CREATE EXTENSION pgaudit;
```

В postgresql.conf:

```
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'ddl, role, write'
pgaudit.log_relation = on
pgaudit.log_parameter = on
```

Теперь все DDL, изменения ролей, INSERT/UPDATE/DELETE логируются в structured формате с параметрами.

Session-level или object-level audit:

```sql
-- Аудит всех операций на конкретную таблицу
ALTER TABLE users SECURITY LABEL FOR pgaudit IS 'READ, WRITE';
```

Каждый доступ (SELECT, INSERT, UPDATE) на users будет логироваться независимо от глобальных настроек.

Логи направлять в централизованное хранилище (SIEM, Splunk, Elasticsearch) для анализа и retention. Тонны логов быстро заполнят диск сервера.

Практика. Для КНП обязательно pgAudit для критических таблиц (auth, financial data, personal data). Логи в централизованный audit trail. Alerting на подозрительные patterns (SELECT из чужого tenant, mass DELETE, изменения ролей).

## Row-level security для audit

Иногда нужно логировать не только запросы, но и **кто именно** их сделал. Обычная модель: приложение подключается как knp_app, из БД всегда видно "knp_app сделал запрос" — не полезно для audit.

Один подход — **SET LOCAL** переменные с идентификатором end-user:

```java
// Перед выполнением business запроса
jdbcTemplate.update("SET LOCAL app.current_user_id = ?", userId);
jdbcTemplate.update("...", ...);
```

pgAudit при логировании включает эти переменные, чтобы понять кто именно.

Другой подход — **PostgreSQL SET ROLE**. Приложение подключается как knp_app, при обработке запроса делает `SET ROLE user_role_for_end_user_5`. Все запросы происходят от имени этой роли. Audit чист.

```sql
SET SESSION AUTHORIZATION 'end_user_5';
-- сессия работает как end_user_5
-- все permissions, RLS policies применяются
-- audit log показывает end_user_5
```

Более полная реализация — на уровне БД, все end users имеют свои роли (created dynamically), приложение — просто authorization broker.

## Data masking для non-prod

Проблема. Разработчикам нужны реалистичные данные для тестирования. Копия prod базы в dev environment — стандартная практика. Но prod данные — sensitive (PII, финансовые). Utente на dev имеет доступ ко всем этим данным.

Data masking — процесс замены чувствительных данных на fake, но structurally correct.

Простой подход через SQL после копирования:

```sql
UPDATE users SET 
    email = 'user_' || id || '@test.com',
    phone = '+7 (000) 000-00-' || LPAD(id::text, 2, '0'),
    ssn = LPAD((RANDOM() * 999999999)::int::text, 9, '0'),
    address = 'Fake Address ' || id;

UPDATE payments SET
    card_number = '4111 1111 1111 1111';  -- test Visa
```

Проблема. Может пропустить колонки, легко забыть новые. Real names остаются consistent между таблицами (Bob в users имеет other refs, они всё ещё указывают на Bob) — иногда это OK, иногда нужно randomize.

Более продвинутое — **dynamic data masking** через RLS + views. Прод содержит real data, но специальные views возвращают masked данные для non-privileged users.

**Anonymization tools**: pg_anonymize (PostgreSQL extension), Faker (Python) в скриптах migration. Специализированные commercial: Delphix, Informatica.

Практика для КНП: не давать разработчикам доступ к prod данным напрямую. Automated pipeline — берёт prod snapshot, прогоняет anonymization script, деплоит в dev. Дeвы работают с realistic но fake данными.

## Заключение

Безопасность PostgreSQL — многослойная тема. Каждый уровень защищает от разных threats, и правильная стратегия комбинирует все.

Роли и privileges — principle of least privilege. Application роль без CREATEDB/CREATEROLE/SUPERUSER, минимально необходимые permissions. Owner для миграций отдельно. Readonly для reports. Password rotation через secret managers.

Аутентификация — SCRAM-SHA-256 минимум, certificate-based auth для критичных систем. pg_hba.conf для network-level access control.

Row-Level Security — на уровне БД гарантирует что пользователь видит только свои данные. Multi-tenancy, department separation, публично/приватно. Force RLS даже для owners.

Column-level permissions — разные роли видят разные колонки. Особенно для sensitive fields (salary, ssn).

Encryption at rest — filesystem-level (LUKS, EBS encryption) обязательно. Column-level (pgcrypto) для особо critical fields с внешним key management (Vault, KMS).

Encryption in transit — SSL/TLS all connections, `sslmode=verify-full`. Certificate-based mutual auth для критичного.

Audit logging — pgAudit для structured audit по DDL, ролям, sensitive tables. Централизованное хранилище для retention и analysis. Alerting на подозрительные patterns.

Data masking для non-prod. Никогда не давать разработчикам real prod data. Automated anonymization pipeline.

Для КНП правильный setup: encryption at rest на всех дисках (compliance requirement). SSL/TLS everywhere с verify-full. Application роли с least privilege. RLS для tenant isolation внутри БД. pgAudit для критичных таблиц. Data masking в dev/staging. Regular security audits и penetration tests.

Дальше — практика. Настрой RLS на любой multi-tenant таблице в тестовой БД. Проверь что даже при "забытом" WHERE в запросе, данные других tenant'ов невидимы. Настрой pgcrypto для encrypted column, замерь performance impact. Прогони simple penetration test — что может SQL injection'нутый knp_app? Каждая проверка raises баровку security.
