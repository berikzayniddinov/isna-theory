# 96. Advanced SQL: CTE, window functions, keyset pagination, аналитика

## Зачем углубляться в SQL

Средний Java-разработчик знает SQL на уровне JOIN, GROUP BY, ORDER BY, простые агрегации. Этого достаточно для 70% задач через ORM. Но остальные 30% — те самые, где производительность реально имеет значение — требуют более глубокого владения SQL. Пейджинация на большой таблице через OFFSET 100000 работает секунды вместо миллисекунд. Отчёт с running total и rank в приложении делается через N+1 запросов вместо одного SQL с window functions. Иерархические данные обходятся рекурсивным Java-кодом вместо recursive CTE.

Знание advanced SQL — это то, что отличает разработчика, который «умеет ORM» от разработчика, который знает как реляционные базы работают. И часто разница между медленным приложением и быстрым — не в архитектуре, а в том, какими запросами написана бизнес-логика.

В этом файле разберём четыре темы, которые дают самую большую отдачу. Common Table Expressions (CTE) — способ структурировать сложные запросы и делать иерархические обходы через recursive CTE. Window functions — аналитические функции без потери детализации, замена сложных self-join'ов и коррелированных подзапросов. Keyset pagination — правильный способ реализации paging'а на больших таблицах вместо OFFSET. Оптимизация агрегатных запросов — pre-computed metrics, incremental aggregation, materialized views, roll-up patterns.

## Common Table Expressions (CTE)

CTE — это способ определить именованный подзапрос в начале большого запроса и потом ссылаться на него по имени. По сути «локальные view» для одного запроса.

Простое использование:

```sql
WITH active_users AS (
    SELECT id, email FROM users WHERE last_login > NOW() - INTERVAL '30 days'
),
recent_orders AS (
    SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days'
)
SELECT au.email, COUNT(ro.id) AS order_count
FROM active_users au
LEFT JOIN recent_orders ro ON ro.user_id = au.id
GROUP BY au.email;
```

Плюс — читаемость. Сложный запрос разбит на логические блоки, каждый именован. Проще понимать, проще менять, проще отлаживать.

Один тонкий момент. До PostgreSQL 12 CTE был **optimization fence** — Optimizer не мог inline CTE в основной запрос, обязательно materializ'ировал результат в промежуточное представление. Это могло быть плохо для performance: если бы CTE был inlined, Optimizer мог бы push down фильтры, объединить операции, использовать индексы. С materialization всё это невозможно.

С PostgreSQL 12 поведение изменилось. По умолчанию Optimizer сам решает — может inline (если CTE referenced один раз и без side effects) или материализовать. Можно явно указать:

```sql
WITH active_users AS MATERIALIZED (
    SELECT id FROM users WHERE ...
)
-- явно materialized
```

или

```sql
WITH active_users AS NOT MATERIALIZED (
    SELECT id FROM users WHERE ...
)
-- явно inlined
```

Практика: для нового кода на PG 12+ можно писать CTE без беспокойства о performance, Optimizer выберет разумное поведение. Для legacy может понадобиться переписать CTE в подзапросы если помешать нужным оптимизациям.

## Recursive CTE

Настоящая мощь CTE проявляется в recursive варианте. Позволяет писать запросы, которые «сами себя вызывают» — обходить иерархические структуры, генерировать последовательности, реализовывать graph traversal.

Классический пример — обход иерархии категорий:

```sql
WITH RECURSIVE category_tree AS (
    -- Anchor: корневые категории
    SELECT id, name, parent_id, 0 AS depth, ARRAY[id] AS path
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive: детей текущего уровня
    SELECT c.id, c.name, c.parent_id, ct.depth + 1, ct.path || c.id
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;
```

Механика. Recursive CTE состоит из двух частей объединённых через UNION ALL. **Anchor** — стартовый запрос (корневые категории, depth=0). **Recursive** — запрос, который использует результаты предыдущей итерации (детей категорий текущего уровня).

PostgreSQL выполняет итеративно: сначала anchor, получает результаты. Потом recursive часть выполняется с этими результатами как input. Получает новые результаты. Recursive выполняется снова с новыми результатами. И так до тех пор, пока результаты не станут пустыми.

Использования recursive CTE.

**Иерархии**. Organizational chart, category tree, files/folders. Найти всех подчинённых, все под-категории, всех предков.

**Bill of materials**. Расчёт составных деталей: изделие → детали → под-детали → материалы.

**Дистанции в графе**. Найти shortest path, friends of friends, connected components. Не так эффективно как Neo4j для больших графов, но работает для небольших.

**Генерация последовательностей**. Календарь дат, run-length encoding, series expansion.

```sql
-- Генерация всех дат в диапазоне
WITH RECURSIVE dates AS (
    SELECT '2024-01-01'::date AS d
    UNION ALL
    SELECT d + 1 FROM dates WHERE d < '2024-12-31'
)
SELECT * FROM dates;
```

Хотя для этого случая есть проще: `generate_series('2024-01-01'::date, '2024-12-31', INTERVAL '1 day')`.

**Правило: recursive CTE может замедлить систему если не аккуратно**. Проверять limit'ы, включать условия termination, следить за размером промежуточных результатов. По умолчанию `max_stack_depth` защищает от бесконечной рекурсии, но лучше явно ограничивать.

## Window functions

Обычные aggregation функции (COUNT, SUM, AVG) с GROUP BY сжимают строки в группы. `SELECT country, SUM(amount) FROM orders GROUP BY country` возвращает одну строку на страну — детали заказов теряются.

Window functions — аналитические функции которые работают над «окном» строк, но **сохраняют детализацию**. Каждая исходная строка остаётся, но получает дополнительные aggregate значения.

Синтаксис `функция() OVER (окно)`. Окно определяется через PARTITION BY (разбиение) и ORDER BY (сортировка внутри окна).

Простой пример: total amount по каждой стране, но с детализацией по каждому заказу:

```sql
SELECT id, country, amount,
       SUM(amount) OVER (PARTITION BY country) AS country_total,
       amount * 100.0 / SUM(amount) OVER (PARTITION BY country) AS pct_of_country
FROM orders;
```

Каждая строка orders сохраняется, но добавлены два поля: total по её стране и её процент от country_total. Без GROUP BY, без потери деталей.

Другой классический пример — running total:

```sql
SELECT id, created_at, amount,
       SUM(amount) OVER (ORDER BY created_at) AS running_total
FROM orders;
```

Для каждого заказа — сумма amounts всех предыдущих заказов включая текущий. Классика для financial reports.

## Ranking functions

Специальные window functions для ranking:

**ROW_NUMBER()** — уникальный порядковый номер:

```sql
SELECT id, user_id, created_at,
       ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) AS order_num
FROM orders;
```

Каждый заказ user'а получает номер: 1, 2, 3, ... в порядке даты создания.

**RANK()** — ранг с одинаковыми значениями:

```sql
SELECT id, amount,
       RANK() OVER (ORDER BY amount DESC) AS amount_rank
FROM orders;
```

Если два заказа имеют одинаковый amount, они получают одинаковый rank, следующий пропускается (1, 2, 2, 4).

**DENSE_RANK()** — как RANK, но без пропусков (1, 2, 2, 3).

**NTILE(N)** — разбиение на N примерно равных групп. `NTILE(4)` даёт квартили.

Классическая задача — top-3 orders per user:

```sql
WITH ranked AS (
    SELECT id, user_id, amount,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn
    FROM orders
)
SELECT * FROM ranked WHERE rn <= 3;
```

Без window functions это сложный self-join или коррелированный подзапрос. С ними — четыре строки.

## LAG и LEAD

Функции для доступа к соседним строкам без self-join.

**LAG(column, N)** — значение column из строки N позиций назад в окне.

**LEAD(column, N)** — из строки N позиций вперёд.

Пример: показать разницу с предыдущим заказом того же user'а:

```sql
SELECT id, user_id, amount, created_at,
       LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_amount,
       amount - LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS diff
FROM orders;
```

Для первого заказа user'а prev_amount будет NULL. Можно указать default: `LAG(amount, 1, 0)`.

Полезно для computing deltas, detecting changes, time series analysis.

## Frame clause

По умолчанию window включает все строки от начала окна до **текущей строки** (для функций с ORDER BY). Для агрегатов без ORDER BY — все строки окна.

Можно явно управлять frame'ом:

```sql
SELECT id, amount,
       -- Скользящая сумма последних 7 дней:
       SUM(amount) OVER (
           ORDER BY created_at 
           RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
       ) AS last_7_days
FROM orders;
```

`ROWS` и `RANGE` — разные типы frame:
- ROWS — по количеству строк.
- RANGE — по значению ORDER BY колонки.

Комбинации: `ROWS BETWEEN 3 PRECEDING AND CURRENT ROW`, `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, `RANGE BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING`.

Для moving averages и analytical patterns правильный frame критически важен.

## Именованные окна

Если одно и то же окно используется несколько раз, можно определить его один раз:

```sql
SELECT id, user_id, amount,
       ROW_NUMBER() OVER user_win AS order_num,
       SUM(amount) OVER user_win AS running_total,
       AVG(amount) OVER user_win AS running_avg
FROM orders
WINDOW user_win AS (PARTITION BY user_id ORDER BY created_at);
```

Читабельнее, короче. Одно определение окна — три функции над ним.

## Keyset pagination

Классическая проблема. У тебя большая таблица (миллионы строк), нужно постранично отдавать данные (100 строк per page). Классический подход:

```sql
SELECT * FROM orders 
ORDER BY created_at DESC 
LIMIT 100 OFFSET 5000;
```

Работает для маленьких offset. Но `OFFSET 100000` — катастрофа. PostgreSQL должен прочитать первые 100100 строк (в отсортированном порядке), первые 100000 отбросить, оставшиеся 100 вернуть. Чем больше offset, тем медленнее. На большой таблице `OFFSET 1000000` — секунды.

Правильное решение — **keyset pagination** (иногда называют cursor-based pagination). Вместо пропуска строк — используем значение конкретной колонки как cursor.

```sql
-- Первая страница
SELECT * FROM orders 
ORDER BY created_at DESC, id DESC 
LIMIT 100;

-- Последняя строка полученного результата имеет created_at=X, id=Y
-- Следующая страница:
SELECT * FROM orders 
WHERE (created_at, id) < ('2024-01-15 10:30:00', 12345)
ORDER BY created_at DESC, id DESC 
LIMIT 100;
```

Использование **row-value comparison** `(a, b) < (X, Y)` эквивалентно `a < X OR (a = X AND b < Y)`. Работает идеально с составным индексом на `(created_at DESC, id DESC)`.

При таком запросе PostgreSQL идёт напрямую к нужной позиции в индексе (log N), читает 100 строк, возвращает. Мгновенно независимо от того, страница 1 или 10000.

Плюсы keyset pagination:
- Constant time для любой страницы.
- Стабильно при вставке новых строк (offset может пропускать/дублировать при concurrent inserts).

Минусы:
- Нельзя прыгать на конкретную страницу N (только «следующая», «предыдущая»).
- Требует stable ordering — колонки должны формировать unique tuple (обычно ORDER BY по бизнес-колонке + id как tiebreaker).

Для UI с бесконечной прокруткой (Twitter, Facebook feed) — идеально. Для traditional table с номерами страниц — не подходит, там всё равно OFFSET.

## Оптимизация агрегатов

Другая частая проблема — тяжёлые агрегатные запросы. `SELECT country, SUM(amount) FROM orders GROUP BY country` на 100M-строчной таблице — минуты работы. Читать 20 GB, вычислять сумму, группировать.

Подходы к оптимизации.

**Индексы для агрегатов**. PostgreSQL может использовать index-only scan для агрегатов, если все нужные колонки в индексе:

```sql
CREATE INDEX ON orders (country, amount);

SELECT country, SUM(amount) FROM orders GROUP BY country;
```

Планировщик может пройти по индексу, читая только country и amount. Это быстрее чем читать весь heap. Проверить через EXPLAIN — Index Only Scan вместо Seq Scan.

**Партиционирование**. Если данные партиционированы, каждая партиция агрегирается независимо, потом объединяются. Плюс `enable_partitionwise_aggregate = on` даёт parallel aggregation.

**Materialized views**. Для frequently запрашиваемых aggregates — pre-compute:

```sql
CREATE MATERIALIZED VIEW sales_by_country AS
SELECT country, SUM(amount) AS total, COUNT(*) AS orders
FROM orders GROUP BY country;

REFRESH MATERIALIZED VIEW CONCURRENTLY sales_by_country;
```

Обсуждали в 94.

**Incremental aggregation**. Часто нужна свежая агрегация. Полный REFRESH каждый раз — дорого. Incremental — обновлять только что изменилось.

Простейший вариант — trigger на INSERT/UPDATE/DELETE обновляет счётчик:

```sql
CREATE TABLE sales_counters (
    country VARCHAR(2) PRIMARY KEY,
    total NUMERIC DEFAULT 0,
    orders INT DEFAULT 0
);

CREATE FUNCTION update_sales_counters() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO sales_counters(country, total, orders)
        VALUES (NEW.country, NEW.amount, 1)
        ON CONFLICT (country) DO UPDATE
        SET total = sales_counters.total + NEW.amount,
            orders = sales_counters.orders + 1;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE sales_counters 
        SET total = total - OLD.amount, orders = orders - 1
        WHERE country = OLD.country;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_counter 
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION update_sales_counters();
```

Плюсы: sales_counters всегда актуальна, query мгновенный (одна строка).

Минусы: overhead на каждый INSERT/UPDATE/DELETE, потенциальный contention если много concurrent writes в одну country (hot row).

Для hot row можно использовать **counter sharding** — вместо одной строки на country, N строк, INSERT в случайную, при чтении SUM всех:

```sql
CREATE TABLE sales_counters_sharded (
    country VARCHAR(2),
    shard SMALLINT,
    total NUMERIC,
    orders INT,
    PRIMARY KEY (country, shard)
);
```

Каждый INSERT идёт в `shard = random(0..99)`. Reads: `SELECT country, SUM(total) FROM sales_counters_sharded GROUP BY country`. Contention распределяется.

**TimescaleDB continuous aggregates**. Для time-series — специальный механизм. Автоматически обновляемые materialized views с incremental refresh:

```sql
CREATE MATERIALIZED VIEW hourly_sales
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', created_at) AS hour, 
       country, SUM(amount) AS total
FROM orders
GROUP BY hour, country;

SELECT add_continuous_aggregate_policy('hourly_sales',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

TimescaleDB автоматически обновляет только новые часы, не пересчитывая старые.

## Аналитические паттерны

Несколько типовых задач и их решения.

**Top-N per group**. Уже показали через ROW_NUMBER.

**Running total и cumulative distributions**. Window function с SUM ORDER BY.

**Percentiles**. `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount)` — медиана. Percentile_disc для discrete percentile.

**Skewed values (outliers)**. Через statistical functions: `stddev(amount)`, `variance(amount)`, поиск строк за пределами N sigma.

**Cohort analysis**. GROUP BY по signup month + активность в последующих месяцах. Часто через generate_series и join.

**Funnel analysis**. Пользователь проходит steps A → B → C. Считать конверсию на каждом шаге. Часто через lateral join и window functions.

**Retention curves**. Для каждого cohort — процент активных пользователей через N дней после регистрации. Классическая задача product analytics.

Эти запросы сложные, но с window functions и generate_series разумной длины и производительности. Без — Java-код с миллионами roundtrip'ов.

## FILTER clause

Полезная штука — inline фильтр в aggregate функциях.

```sql
SELECT country,
       COUNT(*) AS total_orders,
       COUNT(*) FILTER (WHERE status = 'completed') AS completed,
       COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled,
       SUM(amount) FILTER (WHERE status = 'completed') AS revenue
FROM orders
GROUP BY country;
```

За один проход по данным получить несколько условных агрегатов. Раньше делалось через CASE WHEN — работает, но verbose.

## Заключение

Advanced SQL — большая тема, дающая колоссальную отдачу когда владеешь. CTE делают сложные запросы читаемыми, recursive CTE открывают иерархические обходы. Window functions заменяют десятки строк Java-кода одним SQL. Keyset pagination спасает от катастрофического OFFSET на больших таблицах. Оптимизация агрегатов через индексы, materialized views, incremental через триггеры или TimescaleDB — переводит запросы из минут в миллисекунды.

CTE: с PG 12+ по умолчанию inlining, можно явно MATERIALIZED или NOT MATERIALIZED. Recursive для иерархий, графов, генерации последовательностей. Всегда termination condition, следить за производительностью.

Window functions: OVER (PARTITION BY ... ORDER BY ...). Основные функции: ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD, SUM/AVG/COUNT OVER, PERCENTILE_CONT. Frame clause для скользящих окон. Именованные окна для читаемости.

Keyset pagination: (col1, col2) < (X, Y) вместо OFFSET. Требует stable ordering и составной индекс. Constant time для любой страницы. Не подходит для UI с прыжками на страницу N.

Aggregate optimization: индексы для index-only scan, партиционирование для parallel aggregate, materialized views для pre-compute, triggers или TimescaleDB для incremental refresh. FILTER clause для условных агрегатов.

Практика: возьми любой отчёт в приложении, который пишется через несколько roundtrip'ов или сложную Java-логику. Попробуй переписать через CTE + window functions в один SQL. Замерь до/после. Часто разница — порядки.

Дальше — практика реально важна. Прочитать эту тему сложно на память понять; она заходит только когда решишь задачу через window function, увидишь как чисто и быстро получается вместо привычного nested loop. Возьми SQL problem set (например HackerRank или Leetcode SQL) и пройди сложные задачи через advanced features. Каждая решённая — новый инструмент в арсенале.
