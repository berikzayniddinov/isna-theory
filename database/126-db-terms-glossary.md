# 126. Термины БД: словарь простыми словами

Все основные термины PostgreSQL/SQL которые встречаются в работе. С примерами из ISNA.

---

## 1. Основы: что такое БД

### Database (БД)
Место где хранятся данные. Один PostgreSQL сервер может содержать несколько БД.

**Пример из ISNA**: БД `isna_tax_rep`, `isna_nz`, `isna_arm`.

### Schema (схема)
"Папка" внутри БД. Группирует таблицы по смыслу.

**Пример из ISNA**: схема `tax_rep_100` для ФНО 100.00, `tax_rep_328` для ФНО 328.00.

```
    Database: isna_tax_rep
    ├── Schema: public         (общие таблицы)
    ├── Schema: tax_rep_100    (таблицы для 100 формы)
    ├── Schema: tax_rep_328    (таблицы для 328 формы)
    └── Schema: tax_rep_report (общий отчёт)
```

### Table (таблица)
Данные в виде строк и колонок (как Excel).

### Row (строка / запись / кортеж / tuple)
Одна запись в таблице. В PostgreSQL внутри называется **tuple**.

### Column (колонка / поле)
Один атрибут записи (имя, дата, сумма).

### Data type (тип данных)
Что хранится в колонке.

| Тип | Что | Пример |
|-----|-----|--------|
| `INT` / `BIGINT` | Целое число | 42, 1000000 |
| `NUMERIC(15,2)` | Число с точностью | Сумма налога 1234.56 |
| `VARCHAR(20)` | Строка до N символов | ИИН "870101300123" |
| `TEXT` | Строка любой длины | XML документ |
| `TIMESTAMP` | Дата+время | 2026-09-29 18:00:00 |
| `DATE` | Только дата | 2026-09-29 |
| `BOOLEAN` | true/false | is_active |
| `UUID` | Уникальный ID | uuid_generate_v4() |
| `JSONB` | JSON бинарный | payload из ФНО |
| `BYTEA` | Бинарные данные | ЭЦП подпись |

---

## 2. Ключи

### Primary key (PK, первичный ключ)
Уникальный ID строки. Не может повторяться и не NULL.

**Пример**: `id UUID PRIMARY KEY` для декларации.

### Foreign key (FK, внешний ключ)
Ссылка на строку в другой таблице.

**Пример**:
```
tax_report_declaration.taxpayer_id → taxpayer.id
```

Смысл: "эта декларация принадлежит этому налогоплательщику".

### Unique constraint
Значения в колонке не могут повторяться (но могут быть NULL).

**Пример**: `iin` в таблице `taxpayer` должен быть уникальным.

### Composite key (составной ключ)
PK из нескольких колонок.

**Пример**: `PRIMARY KEY (declaration_id, line_number)` — уникальность строки внутри декларации.

### Surrogate key vs Natural key
- **Surrogate** — искусственный ID (UUID, sequence). Не имеет бизнес-смысла.
- **Natural** — реальный атрибут (ИИН, номер декларации).

В ISNA обычно surrogate (UUID) + natural как unique constraint.

---

## 3. Индексы

### Index (индекс)
Отдельная структура для быстрого поиска. Аналог оглавления в книге.

**Без индекса**: `WHERE iin = '870101...'` → PG читает всю таблицу (**Seq Scan**).

**С индексом**: PG идёт по индексу → сразу находит нужную строку (**Index Scan**).

### Типы индексов в PostgreSQL

| Тип | Для чего |
|-----|----------|
| **B-tree** | Default. Для `=`, `<`, `>`, `BETWEEN`, `ORDER BY`. |
| **Hash** | Только `=`. Редко используется. |
| **GIN** | Для массивов, JSONB, full-text search. |
| **GiST** | Геометрия, диапазоны, полнотекстовый поиск. |
| **BRIN** | Для очень больших таблиц с упорядоченными данными (время). |

### Composite index (составной индекс)
Индекс по нескольким колонкам.

**Пример**: `INDEX (taxpayer_id, period)` — быстро найти декларации налогоплательщика за период.

**⚠ Важно**: порядок колонок важен. `INDEX (a, b)` работает для `WHERE a=?` и `WHERE a=? AND b=?`, но НЕ для `WHERE b=?`.

### Partial index (частичный индекс)
Индекс только по части строк.

**Пример**:
```sql
CREATE INDEX ON declaration(status)
WHERE status = 'DRAFT';
```
Индексируем только черновики (их мало), а не все декларации.

### Covering index (покрывающий индекс)
Индекс который содержит ВСЕ нужные колонки — не нужно идти в таблицу.

```sql
CREATE INDEX ON declaration(taxpayer_id) INCLUDE (status, created_at);
```

### Unique index
Индекс + гарантия уникальности. `UNIQUE` constraint автоматически создаёт unique index.

---

## 4. Constraints (ограничения)

### NOT NULL
Значение обязательно.

### CHECK
Условие на значение.

**Пример**: `CHECK (amount >= 0)` — сумма не может быть отрицательной.

### DEFAULT
Значение по умолчанию если не указано.

**Пример**: `created_at TIMESTAMP DEFAULT now()`.

### UNIQUE
Уникальность (см. выше).

### PRIMARY KEY
NOT NULL + UNIQUE + автоматический индекс.

### FOREIGN KEY
Ссылочная целостность. Можно указать поведение при удалении родителя:
- `ON DELETE CASCADE` — удалить дочерние.
- `ON DELETE RESTRICT` — запретить удаление.
- `ON DELETE SET NULL` — обнулить FK.

---

## 5. Специальные объекты

### View (представление)
"Виртуальная таблица" = сохранённый SELECT.

**Пример**:
```sql
CREATE VIEW active_declarations AS
SELECT * FROM declaration WHERE status = 'ACTIVE';
```

Каждый раз при обращении к view выполняется базовый SELECT.

### Materialized view (материализованное представление)
View + физически сохранённые данные. Не пересчитывается автоматически — нужен `REFRESH`.

**Плюс**: очень быстрое чтение.
**Минус**: данные не всегда актуальные.

### Sequence (последовательность)
Генератор чисел (1, 2, 3, ...).

**Пример**:
```sql
CREATE SEQUENCE declaration_id_seq;
SELECT nextval('declaration_id_seq');  -- 1
SELECT nextval('declaration_id_seq');  -- 2
```

В Java используется как `@GeneratedValue(strategy = SEQUENCE)`.

### Trigger (триггер)
Функция которая срабатывает автоматически при INSERT/UPDATE/DELETE.

**Пример**: перед UPDATE — сохранить старую версию в audit таблицу.

### Stored procedure / Function
Функция в БД, вызывается из SQL.

**Пример**:
```sql
CREATE FUNCTION calculate_tax(income NUMERIC) RETURNS NUMERIC AS $$
  RETURN income * 0.1;
$$ LANGUAGE plpgsql;
```

В ISNA используются функции для сложной бизнес-логики (расчёт налога).

---

## 6. Транзакции

### Transaction (транзакция)
Группа операций которые выполняются как ОДНА единица. Или все, или ни одна.

```sql
BEGIN;                                    -- начало
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;                                    -- сохранить
-- или
ROLLBACK;                                  -- откатить
```

В Spring: `@Transactional`.

### ACID
Четыре свойства транзакций:

- **A**tomicity (Атомарность) — всё или ничего.
- **C**onsistency (Консистентность) — БД переходит из одного корректного состояния в другое.
- **I**solation (Изоляция) — параллельные транзакции не мешают друг другу.
- **D**urability (Долговечность) — после COMMIT данные не потеряются.

### Savepoint
Точка внутри транзакции к которой можно откатиться, не откатывая всю транзакцию.

---

## 7. Isolation levels

**Проблема**: две транзакции работают параллельно — как они видят изменения друг друга?

| Уровень | Dirty Read | Non-repeatable Read | Phantom Read |
|---------|-----------|----------------------|--------------|
| Read Uncommitted | возможно | возможно | возможно |
| **Read Committed** (default в PG) | нет | возможно | возможно |
| Repeatable Read | нет | нет | возможно (в PG — нет!) |
| Serializable | нет | нет | нет |

- **Dirty Read** — видишь незакоммиченные данные другой транзакции.
- **Non-repeatable Read** — один и тот же SELECT возвращает разное в одной транзакции.
- **Phantom Read** — новый SELECT возвращает НОВЫЕ строки которых не было.

**Пример из ISNA**: при приёмке декларации — `Repeatable Read` чтобы во время проверок налогоплательщик не поменялся.

---

## 8. Локи (Locks)

### Row lock
Блокировка одной строки. `SELECT ... FOR UPDATE` блокирует строку до конца транзакции.

### Table lock
Блокировка всей таблицы. `LOCK TABLE ... IN ... MODE`. Обычно ставится автоматически при DDL.

### Advisory lock
Ручной "именованный" лок для приложения.

```sql
SELECT pg_advisory_lock(12345);
-- делаем что-то
SELECT pg_advisory_unlock(12345);
```

**Пример**: гарантировать что только один инстанс приложения обрабатывает очередь.

### Deadlock
Две транзакции блокируют друг друга.

```
Tx1: LOCK строка A → ждёт строку B
Tx2: LOCK строка B → ждёт строку A
→ DEADLOCK. PG убивает одну.
```

**Решение**: всегда брать локи в одинаковом порядке.

---

## 9. MVCC (Multi-Version Concurrency Control)

Как PostgreSQL позволяет читать и писать одновременно без блокировки.

**Идея**: при UPDATE не переписываем строку — создаём НОВУЮ версию. Старая живёт пока её видят читатели.

```
Строка id=1, версия 1 (xmin=100, xmax=200) ← старая
Строка id=1, версия 2 (xmin=200, xmax=NULL) ← новая
```

Транзакция 150 видит версию 1 (её xmin=100 < 150, xmax=200 > 150 — валидна).
Транзакция 250 видит версию 2.

**Следствие**: UPDATE = INSERT + пометка старой строки. Старые версии копятся → нужен **VACUUM**.

---

## 10. VACUUM / Autovacuum

### VACUUM
Убирает мёртвые версии строк (созданные из-за MVCC).

**Обычный VACUUM**: помечает место свободным (может переиспользовать), но не отдаёт диску.

**VACUUM FULL**: реально сжимает таблицу. **Блокирует таблицу целиком** — использовать осторожно.

### ANALYZE
Обновляет статистику планировщика (сколько строк, распределение значений).

### Autovacuum
Демон который сам запускает VACUUM+ANALYZE. Настраивается через `autovacuum_vacuum_scale_factor` и т.д.

### Bloat (раздутие)
Таблица занимает больше места чем нужно из-за мёртвых версий. VACUUM решает.

---

## 11. WAL (Write-Ahead Log)

Журнал всех изменений. **Записывается ДО** изменения данных на диске.

**Зачем**: если сервер упал — можем восстановить состояние проигрыванием WAL.

Файлы: `/var/lib/postgresql/data/pg_wal/`.

### Checkpoint
Момент когда все изменения из RAM (shared_buffers) сбрасываются на диск. После checkpoint старый WAL можно удалить.

---

## 12. EXPLAIN / Query planner

### EXPLAIN
Показывает КАК PG собирается выполнить запрос.

```sql
EXPLAIN SELECT * FROM declaration WHERE iin = '870101...';
```

Результат:
```
Index Scan using declaration_iin_idx  (cost=0.43..8.45 rows=1 width=200)
```

### EXPLAIN ANALYZE
То же, но реально выполняет запрос и показывает фактическое время.

### Планировщик (Query Planner)
Модуль PG который выбирает план выполнения. Использует **статистику** (обновляется через ANALYZE).

### Основные операции в плане

| Операция | Что |
|----------|-----|
| **Seq Scan** | Читает всю таблицу. Медленно на больших. |
| **Index Scan** | Идёт по индексу. Быстро. |
| **Bitmap Index Scan** | Собирает bitmap подходящих строк, потом читает. Для многих строк. |
| **Nested Loop** | JOIN по одному. Для маленьких таблиц. |
| **Hash Join** | Строит hash по одной таблице, ищет по ней. Средние. |
| **Merge Join** | Сортирует обе, идёт параллельно. Большие таблицы. |

---

## 13. JOIN'ы

### INNER JOIN
Только строки где есть совпадение в обеих таблицах.

```
declaration ⨝ taxpayer ON declaration.taxpayer_id = taxpayer.id
```

### LEFT JOIN
Все строки из левой + совпавшие из правой (NULL если нет).

**Пример**: все декларации + их налогоплательщики (если удалили — NULL).

### RIGHT JOIN
Зеркально к LEFT.

### FULL JOIN
Все строки из обеих + NULL где нет совпадения.

### CROSS JOIN
Декартово произведение (все × все). Обычно ошибка.

### LATERAL JOIN
"For each row в левой — выполнить подзапрос".

---

## 14. Продвинутый SQL

### CTE (Common Table Expression / WITH)
Временный именованный результат внутри запроса.

```sql
WITH active_declarations AS (
  SELECT * FROM declaration WHERE status = 'ACTIVE'
)
SELECT * FROM active_declarations WHERE created_at > '2026-01-01';
```

Читабельнее чем вложенные подзапросы.

### Recursive CTE
CTE которая вызывает саму себя. Для иерархий (дерево ФНО, родитель-ребёнок).

### Window functions
Функции над "окном" строк без GROUP BY.

```sql
SELECT
  iin,
  amount,
  SUM(amount) OVER (PARTITION BY iin) AS total_by_iin,
  ROW_NUMBER() OVER (ORDER BY amount DESC) AS rank
FROM declaration;
```

### GROUP BY / HAVING
- `GROUP BY` — группировка.
- `HAVING` — фильтр ПОСЛЕ группировки (`WHERE` — до).

### UNION / INTERSECT / EXCEPT
Операции над множествами.
- `UNION` — объединение (без дубликатов).
- `UNION ALL` — с дубликатами (быстрее).
- `INTERSECT` — пересечение.
- `EXCEPT` — разность.

---

## 15. Категории SQL команд

| Категория | Что | Команды |
|-----------|-----|---------|
| **DDL** | Data Definition | CREATE, ALTER, DROP, TRUNCATE |
| **DML** | Data Manipulation | INSERT, UPDATE, DELETE, MERGE |
| **DQL** | Data Query | SELECT |
| **DCL** | Data Control | GRANT, REVOKE |
| **TCL** | Transaction Control | BEGIN, COMMIT, ROLLBACK, SAVEPOINT |

---

## 16. Репликация / масштабирование

### Replication (репликация)
Копия БД на другом сервере.

- **Master** (Primary) — куда пишем.
- **Standby** (Replica) — куда данные копируются.

### Streaming replication
Master шлёт WAL на standby в реальном времени.

### Read replica
Standby можно использовать для READ запросов (снять нагрузку с master).

### Physical vs Logical replication
- **Physical** — копирует файлы БД байт в байт (стандартная).
- **Logical** — копирует "изменения" (можно фильтровать таблицы).

### Sharding (шардирование)
Разделение данных по СЕРВЕРАМ (не по таблицам).

**Пример**: декларации 2024 года на сервере A, 2025 на сервере B.

### Partitioning (партиционирование)
Разделение одной таблицы на несколько по ключу.

**Пример**: `declaration_2024`, `declaration_2025` внутри одного PG.

Виды:
- **Range** — по диапазону (даты).
- **List** — по списку значений (регион).
- **Hash** — по хешу от ключа.

---

## 17. Connection pool

### Pool (пул соединений)
Готовые открытые соединения с БД. Приложение берёт из пула, возвращает после запроса.

**Зачем**: открыть соединение = дорого (TCP, auth, forking процесса в PG).

### PgBouncer
Внешний pool перед PG. Держит N соединений с PG, обслуживает M соединений от приложения (M >> N).

**Пример из ISNA**: PgBouncer стоит перед всеми БД.

### HikariCP
Java pool на стороне приложения (default в Spring Boot).

---

## 18. Backup / Recovery

### pg_dump / pg_restore
Логический бэкап (SQL команды).

```
pg_dump isna_tax_rep > backup.sql
pg_restore -d isna_tax_rep backup.sql
```

### pg_basebackup
Физический бэкап (файлы БД целиком).

### Point-in-Time Recovery (PITR)
Восстановление на конкретный момент времени. Нужен базовый бэкап + все WAL с того момента.

### WAL archiving
Копирование WAL в архив (S3, NFS) для PITR.

---

## 19. Миграции

### Migration
Изменение схемы БД + версионирование.

### Liquibase / Flyway
Инструменты для миграций.

**Liquibase** (используется в ISNA):
- XML/YAML/SQL changelog.
- Таблица `databasechangelog` — история применённых миграций.
- Каждый changeset применяется 1 раз.

### DDL и блокировки
`ALTER TABLE ADD COLUMN` — быстрая (если без DEFAULT).
`ALTER TABLE ADD COLUMN ... DEFAULT ...` — переписывает всю таблицу, блокирует.

**Правило**: миграции должны быть safe для больших таблиц. См. **106** и **92**.

---

## 20. Роли и права

### Role
Пользователь или группа. В PG нет отдельно users/groups — только roles.

```sql
CREATE ROLE isna_app WITH LOGIN PASSWORD 'xxx';
GRANT SELECT, INSERT ON declaration TO isna_app;
```

### GRANT / REVOKE
Дать / забрать права.

### RLS (Row-Level Security)
Фильтрация строк на уровне БД.

```sql
CREATE POLICY declaration_by_taxpayer ON declaration
  USING (taxpayer_id = current_setting('app.taxpayer_id')::UUID);
```

Каждый пользователь видит только свои декларации.

---

## Что запомнить

**Основа**: Database → Schema → Table → Row → Column.

**Ключи**: PK (уникальный ID), FK (ссылка), UNIQUE (без повторов).

**Индексы**: делают поиск быстрым. B-tree (default), GIN (JSONB), partial (по условию).

**Транзакции**: ACID + isolation levels (в PG default = Read Committed).

**MVCC**: PG не переписывает строки — создаёт новые версии. Отсюда VACUUM.

**WAL**: журнал изменений. Пишется ДО данных. Основа для PITR и репликации.

**EXPLAIN**: покажет план запроса. Ищи `Seq Scan` на большой таблице — обычно это проблема.

**Pool** (PgBouncer + HikariCP): экономит на открытии соединений.

**Миграции** (Liquibase): версионирование схемы. Осторожно с DDL на больших таблицах.

---

Смотри также:
- **122** — PostgreSQL визуально: физическая архитектура.
- **28** — PostgreSQL internals.
- **87** — Pages / IO / storage.
- **88** — Locks и EXPLAIN детально.
- **89** — VACUUM / статистика.
- **90** — Репликация и HA.
- **91** — Partitioning / Sharding.
- **92** — Backup / recovery.
- **96** — CTE / Window functions.
- **103** — Index Scan vs Seq Scan.
- **106** — ALTER TABLE lock queue.
- **108** — Партиционирование.
- **109** — Индексы: когда нужны, когда нет.
