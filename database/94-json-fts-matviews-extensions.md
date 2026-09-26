# 94. JSON, полнотекстовый поиск, материализованные views и расширения

## Зачем всё это в реляционной базе

PostgreSQL начинал как чисто реляционная система. Строго типизированная схема, нормализация, SQL. Но реальные приложения не всегда хорошо ложатся на этот идеал. Иногда нужно хранить гибкие структуры (пользовательские метаданные разного формата, конфиги, event payloads). Иногда нужен поиск по тексту, не по точному значению. Иногда нужны заранее посчитанные агрегаты, чтобы не пересчитывать каждый раз. И иногда для специфических задач нужны возможности, которых в базовом PostgreSQL нет — но есть в extensions.

В этом файле разберём четыре темы, которые сильно расширяют возможности PostgreSQL за пределы простого реляционного хранилища. **JSON/JSONB** — гибкие данные без жёсткой схемы прямо в реляционной базе, с индексацией и запросами. **Полнотекстовый поиск** — быстрый поиск по textовым полям с учётом языка, ranking, phrase search. **Материализованные представления** — предвычисленные результаты запросов, которые обновляются по расписанию. **Расширения** — модульная система, позволяющая добавить в PostgreSQL всё что угодно от PostGIS до pgvector.

Знание этих возможностей часто определяет разницу между sr-инженером, который «в реляционной базе делает всё через нормализованные таблицы и JOIN'ы», и sr-инженером, который выбирает подходящий инструмент под задачу. Иногда это JSONB вместо шести таблиц. Иногда GIN индекс вместо elasticsearch. Иногда pg_stat_statements вместо самописной телеметрии.

## JSON и JSONB: разница

PostgreSQL поддерживает два JSON-типа: `json` и `jsonb`. Название почти одинаковое, но семантика принципиально разная.

`json` хранит исходный JSON текст как есть. Просто text с валидацией что это корректный JSON. Никакой обработки: пробелы сохраняются, порядок ключей сохраняется, дубликаты ключей допускаются (последнее значение выигрывает при чтении). При каждом запросе PostgreSQL заново парсит текст.

`jsonb` хранит **бинарное разложенное представление** JSON. При INSERT JSON парсится один раз, ключи и значения хранятся в оптимизированной структуре. Порядок ключей не сохраняется (нормализуется), пробелы отбрасываются, дубликаты ключей — только последний. При запросах не нужно парсить.

Практическое следствие: **всегда используй jsonb**. JSON нужен только в специфических случаях, когда важен byte-perfect сохранение оригинального текста (например, для сигнатур/токенов). Для всего остального — jsonb быстрее, поддерживает индексы, имеет больше операторов.

Простое использование:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    metadata JSONB
);

INSERT INTO users (metadata) VALUES ('{"country": "KZ", "preferences": {"theme": "dark"}}');
```

Доступ к полям через операторы:

```sql
-- -> возвращает jsonb (для дальнейшей навигации)
SELECT metadata->'preferences'->'theme' FROM users;
-- Результат: "dark"  (это jsonb с кавычками)

-- ->> возвращает text
SELECT metadata->>'country' FROM users;
-- Результат: KZ  (это text)

-- #> для многоуровневой навигации
SELECT metadata#>'{preferences,theme}' FROM users;

-- #>> для многоуровневой навигации с извлечением text
SELECT metadata#>>'{preferences,theme}' FROM users;
```

Разница `->` и `->>` — критическая. `->` даёт jsonb (можно дальше навигировать), `->>` даёт text (можно сравнивать со строками). Забыл `->>` вместо `->` — получаешь неожиданные результаты сравнения (`"KZ"` не равно `KZ`).

## Операторы поиска в JSONB

`jsonb` имеет богатый набор операторов для поиска.

**Содержит (@>)** — jsonb A содержит jsonb B, если все ключи и значения B есть в A:

```sql
SELECT * FROM users WHERE metadata @> '{"country": "KZ"}';
```

Найдёт все users, где в metadata есть ключ country со значением KZ (независимо от других ключей).

**Содержится в (<@)** — обратно, A содержится в B.

**Ключ существует (?)**:

```sql
SELECT * FROM users WHERE metadata ? 'phone';
-- users, у которых есть ключ phone в metadata
```

**Один из ключей существует (?|)** и **все ключи существуют (?&)**:

```sql
SELECT * FROM users WHERE metadata ?| array['phone', 'email'];
-- есть phone ИЛИ email

SELECT * FROM users WHERE metadata ?& array['phone', 'email'];
-- есть И phone И email
```

**Путь (jsonpath, PG 12+)** — powerful способ навигации:

```sql
SELECT metadata @@ '$.preferences.theme == "dark"' FROM users;
-- true если path выполнен
```

JsonPath — целый язык запросов для JSON, аналог XPath.

## Индексы на jsonb

Простой B-tree индекс на jsonb колонку работает только для точного равенства всей структуры — редко полезно. Основной тип индекса — **GIN** (Generalized Inverted Index):

```sql
CREATE INDEX ON users USING gin(metadata);
```

Такой GIN индекс поддерживает операторы `@>`, `<@`, `?`, `?|`, `?&`. Все запросы вида «содержит ли jsonb такой-то ключ или структуру» работают быстро.

Есть два варианта GIN индекса:

**Default (jsonb_ops)** — индексирует все ключи и значения. Больший по размеру, но поддерживает все операторы.

**jsonb_path_ops** — индексирует только paths (сочетания ключ+значение). Меньше по размеру, быстрее, но поддерживает только `@>`:

```sql
CREATE INDEX ON users USING gin(metadata jsonb_path_ops);
```

Правило: если запросы почти всегда `@>` — используй jsonb_path_ops. Если нужны `?` и другие — jsonb_ops.

**Специальный индекс на конкретный path** — expression index:

```sql
CREATE INDEX ON users((metadata->>'country'));
```

Это обычный B-tree на выражении. Работает для `WHERE metadata->>'country' = 'KZ'`. Быстрый, компактный, но только для одного пути.

Практика: для полностью гибких запросов по JSONB — GIN. Для specific frequently-used paths — expression index.

## Когда JSONB, когда нормальная схема

JSONB даёт гибкость, но не является заменой нормализованной схеме. Правило выбора.

**JSONB хорош для**: пользовательские настройки (гибкий набор полей), event payloads (разные типы событий, разные схемы), configurations (иерархические, легко меняющиеся), metadata третьих систем (API responses без чёткой схемы), sparse fields (много опциональных полей, где большинство пустые).

**Нормальная схема хороша для**: сильно структурированных данных (email, phone, name — колонки), данных с частыми UPDATE по конкретным полям (JSONB перезаписывает весь блок при UPDATE), данных с сложными joins по вложенным значениям, данных требующих строгих constraints (foreign keys, unique).

Anti-pattern: заменять реляционные модели на «одна таблица с id и jsonb в котором всё». Теряешь всю пользу PostgreSQL: типовую проверку, foreign keys, эффективные joins. Получаешь плохую производительность на любых сложных запросах.

Правильный подход: **hybrid**. Основные структурированные поля — колонки. Опциональные, гибкие, специфичные — в JSONB колонке рядом.

## UPDATE в JSONB

Одна особенность. UPDATE конкретного поля в JSONB переписывает **весь** jsonb значение. PostgreSQL не поддерживает точечное изменение — всегда replace целиком.

Это делается через функции `jsonb_set`, `jsonb_insert`, `-`, `||`:

```sql
-- Заменить поле
UPDATE users 
SET metadata = jsonb_set(metadata, '{country}', '"RU"')
WHERE id = 5;

-- Добавить новое поле
UPDATE users 
SET metadata = metadata || '{"phone": "+7..."}'
WHERE id = 5;

-- Удалить поле
UPDATE users 
SET metadata = metadata - 'phone'
WHERE id = 5;
```

Всё это генерирует новую версию jsonb и записывает её как новую версию tuple'а (MVCC). Для JSONB большого размера — существенный overhead. Если UPDATE конкретных полей часты и поле большое — вынести в отдельную таблицу.

## Полнотекстовый поиск

Обычные text-операторы PostgreSQL (LIKE, регексы) не подходят для настоящего text search. `LIKE '%слово%'` не использует B-tree индекс — full scan. Плюс не учитывает морфологию (окончания, времена глаголов), не даёт ranking. Для настоящего text search есть отдельная инфраструктура — full-text search (FTS).

Идея FTS. Текст преобразуется в **tsvector** — структуру, содержащую нормализованные lexemes (базовые формы слов) и их позиции. Запрос преобразуется в **tsquery** — булево выражение из lexemes. Оператор `@@` проверяет, соответствует ли tsvector данному tsquery.

Простой пример:

```sql
SELECT to_tsvector('russian', 'Быстрая коричневая лиса прыгает через ленивую собаку');
-- Результат:
-- 'быстр':1 'коричнев':2 'лен':6 'лис':3 'прыга':4 'собак':7
```

Слова нормализованы (уменьшены до основ), stopwords (артикли, предлоги) убраны, каждое слово имеет позицию.

Запрос:

```sql
SELECT to_tsquery('russian', 'лиса & собака');
-- 'лис' & 'собак'

SELECT to_tsvector('russian', 'быстрая коричневая лиса прыгает через ленивую собаку')
    @@ to_tsquery('russian', 'лиса & собака');
-- true (оба слова есть в тексте)
```

Поиск на таблице:

```sql
SELECT * FROM articles
WHERE to_tsvector('russian', title || ' ' || body) 
      @@ to_tsquery('russian', 'python & производительность');
```

Без индекса это медленно (нужно вычислить tsvector для каждой строки). С индексом — быстро.

## Индекс для FTS

**GIN индекс на tsvector** — стандартный способ:

```sql
CREATE INDEX ON articles USING gin(to_tsvector('russian', title || ' ' || body));
```

Это expression index. Работает если запросы используют такое же выражение. Быстрый (десятки миллисекунд на миллионах документов), но занимает много места (индекс может быть больше самих данных).

Более удобный подход — добавить материализованную колонку tsvector:

```sql
ALTER TABLE articles ADD COLUMN search_vector tsvector;

UPDATE articles SET search_vector = 
    to_tsvector('russian', title || ' ' || body);

CREATE INDEX ON articles USING gin(search_vector);
```

Плюс trigger для автоматического обновления:

```sql
CREATE TRIGGER tsvectorupdate BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION tsvector_update_trigger(
    search_vector, 'pg_catalog.russian', title, body
);
```

Теперь search запросы просто:

```sql
SELECT * FROM articles 
WHERE search_vector @@ to_tsquery('russian', 'python & производительность');
```

С PG 12+ есть **generated columns**, ещё удобнее:

```sql
ALTER TABLE articles ADD COLUMN search_vector tsvector 
    GENERATED ALWAYS AS (to_tsvector('russian', title || ' ' || body)) STORED;
```

## Ranking

Полнотекстовый поиск обычно требует ranking — какой документ более релевантен запросу. PostgreSQL даёт две функции:

**ts_rank** — базовый ranking. Учитывает частоту совпадений и позиции.

**ts_rank_cd** — cover density ranking. Учитывает близость слов в тексте (если запрос «python performance», предпочтёт документ где эти слова рядом).

```sql
SELECT title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('russian', 'python & производительность') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;
```

Веса для разных частей документа:

```sql
UPDATE articles SET search_vector = 
    setweight(to_tsvector('russian', title), 'A') ||
    setweight(to_tsvector('russian', body), 'D');

-- ranking с весами:
SELECT ts_rank(search_vector, query, '{0.1, 0.2, 0.4, 1.0}'::real[])
-- веса для 'D', 'C', 'B', 'A' соответственно
```

Слова из title получат вес A (наивысший), из body — D (наименьший). При ranking совпадения в title дают больший rank.

## Phrase search и другие возможности

**Phrase search (PG 9.6+)**: искать exact последовательность слов.

```sql
SELECT to_tsquery('russian', 'быстрая <-> коричневая <-> лиса');
-- Ищет "быстрая коричневая лиса" именно в такой последовательности
```

`<->` — расстояние 1 (соседние). `<2>` — с одним словом между. `<3>` — с двумя.

**Highlighting**: подсветить совпадения в результате.

```sql
SELECT ts_headline('russian', body, query, 'MaxWords=30, MinWords=10') AS snippet
FROM articles, to_tsquery('russian', 'python') query
WHERE search_vector @@ query;
```

Возвращает snippet текста с выделенными совпадениями (тегами <b>...</b>).

**pg_trgm** — расширение для fuzzy search и similarity. Разбивает текст на триграммы (последовательности из 3 букв), позволяет искать по похожести:

```sql
CREATE EXTENSION pg_trgm;

SELECT * FROM users 
WHERE name % 'Ivnov';  -- операция similarity
-- Найдёт "Иванов" даже с опечаткой

SELECT * FROM articles 
WHERE title ILIKE '%иван%';
CREATE INDEX ON articles USING gin(title gin_trgm_ops);
-- pg_trgm GIN индекс, теперь LIKE через GIN, быстро
```

pg_trgm часто используется параллельно с FTS: FTS для structured word search, trgm для fuzzy и substring.

## Материализованные представления

Обычный VIEW — это сохранённый SQL запрос. Каждый раз при обращении к view PostgreSQL исполняет запрос заново. Материализованное VIEW — результат запроса физически сохранён как таблица, обновляется по команде.

Пример: аналитический запрос для дашборда, который считает продажи по странам:

```sql
CREATE MATERIALIZED VIEW sales_by_country AS
SELECT country, SUM(amount) AS total, COUNT(*) AS orders
FROM orders
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY country;
```

Первое создание — выполняет запрос, сохраняет результат. Дальше `SELECT * FROM sales_by_country` — обычное чтение из таблицы, миллисекунды.

Обновление — вручную:

```sql
REFRESH MATERIALIZED VIEW sales_by_country;
```

Это заново выполнит запрос, перезапишет содержимое view. Пока REFRESH идёт, view **недоступен** (ACCESS EXCLUSIVE lock).

С PG 9.4+ есть `CONCURRENTLY`:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_by_country;
```

Не блокирует чтения. Требует наличия unique index на view.

Когда использовать. Типичные случаи: dashboard queries, где latency критична и данные обновляются раз в день/час; aggregate reports по большим таблицам; денормализованные представления для read-heavy nagriuz.

Когда не использовать: если данные должны быть fresh (real-time); если базовые таблицы часто меняются (REFRESH становится узким местом).

Обновление автоматизируется через `pg_cron` или обычный cron:

```sql
SELECT cron.schedule('refresh-sales', '0 * * * *', 
    'REFRESH MATERIALIZED VIEW CONCURRENTLY sales_by_country');
```

Каждый час обновление в фоне.

## Индексы на materialized views

Materialized view — фактически таблица, можно создавать индексы обычным образом:

```sql
CREATE INDEX ON sales_by_country(country);
CREATE UNIQUE INDEX ON sales_by_country(country);  -- для CONCURRENTLY refresh
```

Индексы делают запросы к view ещё быстрее — то, ради чего view и создавался.

## Расширения PostgreSQL

Extensions — модульная система PostgreSQL. Ты можешь добавить в базу новые функции, типы данных, операторы, индексы. Некоторые встроены и просто включаются, другие — third-party пакеты, устанавливаемые отдельно.

Установка extension:

```sql
CREATE EXTENSION extension_name;
```

Обычно ставится один раз в базу (не в схему). Смотреть какие доступны:

```sql
SELECT * FROM pg_available_extensions;
```

Ключевые расширения, о которых полезно знать.

**pg_stat_statements** — уже упоминали. Собирает статистику по всем выполнявшимся запросам: количество вызовов, общее время, среднее время, IO. Основной инструмент для performance diagnosis в проде. Всегда должен быть включен.

```sql
CREATE EXTENSION pg_stat_statements;
-- + shared_preload_libraries = 'pg_stat_statements' в конфиге
```

**pg_repack** — non-blocking VACUUM FULL, обсуждали раньше.

**pg_partman** — автоматическое управление партициями, обсуждали в файле про партиционирование.

**pgcrypto** — криптографические функции. Шифрование колонок, hash, HMAC, PGP.

```sql
CREATE EXTENSION pgcrypto;

-- Bcrypt для паролей:
INSERT INTO users(email, password_hash) 
VALUES ('a@b.com', crypt('secret', gen_salt('bf', 10)));

-- Проверка:
SELECT * FROM users 
WHERE email = 'a@b.com' AND password_hash = crypt('secret', password_hash);
```

**hstore** — key/value store. Существовал до JSONB. Сейчас для новых проектов лучше JSONB, но hstore может встречаться в legacy.

**postgres_fdw** — Foreign Data Wrapper. Позволяет обращаться к таблицам в другой PostgreSQL базе как к локальным.

```sql
CREATE EXTENSION postgres_fdw;

CREATE SERVER other_db FOREIGN DATA WRAPPER postgres_fdw 
    OPTIONS (host 'other-host', dbname 'other');

CREATE USER MAPPING FOR current_user SERVER other_db 
    OPTIONS (user 'x', password 'y');

CREATE FOREIGN TABLE remote_orders (
    id BIGINT, amount NUMERIC
) SERVER other_db OPTIONS (schema_name 'public', table_name 'orders');

SELECT * FROM remote_orders WHERE amount > 1000;
-- Postgres прозрачно проксирует запрос в other-host
```

Полезно для миграций, интеграций, cross-database запросов.

**PostGIS** — geographic information system. Adds типы данных (point, line, polygon), функции (distance, contains, intersects), индексы (GIST для геоданных). Стандарт для приложений с геолокацией. Один из самых зрелых extensions.

**pgvector** — vector similarity search. Adds vector тип для embeddings и индексы (HNSW, IVFFlat) для быстрого nearest neighbor search. Актуально для AI приложений: хранение embeddings текста/изображений и RAG (retrieval augmented generation).

```sql
CREATE EXTENSION vector;

CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)  -- OpenAI ada-002 dimension
);

CREATE INDEX ON documents USING hnsw(embedding vector_cosine_ops);

-- Найти 10 самых похожих на данный embedding:
SELECT * FROM documents 
ORDER BY embedding <=> $1 LIMIT 10;
```

Позволяет использовать PostgreSQL как vector database, конкурируя с Pinecone/Weaviate. Не такой быстрый как специализированные vector базы на billions вопросов, но для миллионов — вполне достаточно.

**TimescaleDB** — extension для time-series. Adds hypertables (автоматически партиционируемые по времени), continuous aggregates (auto-updating materialized views), compression (10-20x сжатие старых данных), retention policies. Специализация для метрик, IoT, финансовых данных.

**pg_cron** — scheduler внутри PostgreSQL. Позволяет запускать SQL-команды по cron-выражениям без внешних инструментов:

```sql
SELECT cron.schedule('daily-cleanup', '0 3 * * *', 
    'DELETE FROM audit_log WHERE created_at < NOW() - INTERVAL ''90 days''');
```

**pg_hint_plan** — hint'ы для Optimizer'а. Позволяет указывать конкретные планы (использовать конкретный индекс, конкретный join type). Мощно, но опасно — использовать только в крайних случаях, когда Optimizer стойко выбирает плохой план.

**pgAudit** — детальный audit logging. Логирует все DDL, DML, roles, при чтении sensitive колонок. Для compliance requirements.

## Мелочи, которые расширяют возможности

Некоторые встроенные фичи, о которых часто забывают.

**Common Table Expressions (CTE)** и **recursive CTE** — способ структурировать сложные запросы или реализовывать иерархические обходы.

```sql
-- Обычный CTE
WITH recent_orders AS (
    SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days'
)
SELECT user_id, COUNT(*) FROM recent_orders GROUP BY user_id;

-- Recursive CTE для иерархий
WITH RECURSIVE tree AS (
    SELECT id, parent_id, name, 0 AS level 
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.parent_id, c.name, t.level + 1
    FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree ORDER BY level, name;
```

С PG 12+ обычные CTE (не recursive) не являются optimization fence — Optimizer может inline их в основной запрос, что часто даёт лучший план. До 12 CTE всегда materialized.

**Window functions** — окно-функции для аналитики без GROUP BY.

```sql
SELECT order_id, amount,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) AS order_num,
    SUM(amount) OVER (PARTITION BY user_id) AS total_by_user,
    LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_amount
FROM orders;
```

Мощнейший инструмент для аналитики. Позволяет заменить многие коррелированные подзапросы. Стоит знать OVER, PARTITION BY, ORDER BY внутри window, ROWS/RANGE frames.

**JSON aggregation** — построение JSON из запроса:

```sql
SELECT jsonb_agg(row_to_json(u)) 
FROM users u WHERE country = 'KZ';
-- Возвращает JSON массив с полями каждого user
```

Часто используется для API responses без extra transformation в приложении.

**LATERAL join** — join, где правая сторона может ссылаться на левую. Полезно для «top N per group» запросов:

```sql
SELECT u.id, o.*
FROM users u
LEFT JOIN LATERAL (
    SELECT * FROM orders 
    WHERE orders.user_id = u.id 
    ORDER BY created_at DESC 
    LIMIT 3
) o ON true;
-- Для каждого user'а — его 3 последних заказа
```

Без LATERAL это делается через window functions + фильтр, но LATERAL часто чище.

## Заключение

PostgreSQL — не просто SQL база. Он комбинирует классическое реляционное хранение с гибкими JSON, полнотекстовым поиском, материализованными views и extensible архитектурой через extensions.

JSONB даёт хранение полу-структурированных данных прямо в базе. Богатые операторы поиска, GIN индексы (с вариантами jsonb_ops и jsonb_path_ops), expression indexes для конкретных paths. Правильное использование — hybrid: основные поля колонками, гибкие в JSONB. Не заменять реляционную модель на «одна таблица с id и JSONB».

Полнотекстовый поиск — встроенное решение для text search с поддержкой языков, морфологии, ranking, phrase search. tsvector + tsquery + GIN index. Материализованная колонка + trigger для автообновления. pg_trgm как дополнение для fuzzy и substring search.

Материализованные views — предвычисленные результаты для дорогих аналитических запросов. REFRESH CONCURRENTLY не блокирует чтения. Автообновление через pg_cron. Индексы на views работают как на таблицах. Идеально для dashboard queries, aggregate reports.

Extensions — модульная архитектура, добавляющая огромные возможности. pg_stat_statements обязателен в проде. pg_repack для non-blocking maintenance. pgcrypto для crypto operations. postgres_fdw для cross-database. PostGIS для геоданных. pgvector для AI/embeddings. TimescaleDB для time-series. pg_cron для schedule inside базы. pgAudit для compliance.

Плюс встроенные CTE, window functions, LATERAL join, jsonb_agg — вещи, знание которых существенно расширяет возможности при написании запросов.

Для КНП практично: pg_stat_statements включён; jsonb колонки для extensible форм данных; полнотекстовый поиск для search по обращениям и документам; pg_repack для регулярного обслуживания больших таблиц; postgres_fdw для интеграции между базами; PostGIS если есть геолокация; pg_cron для scheduled tasks вместо внешнего Quartz.

Дальше — практика. Создай в своей базе jsonb колонку, поиграй с операторами. Добавь полнотекстовый поиск на любую textовую колонку. Настрой materialized view для существующего слабого dashboard-запроса. Установи и попробуй pgvector, посчитай похожесть между текстами через OpenAI embeddings. Каждый такой эксперимент открывает новые возможности PostgreSQL, о которых в стандартной работе даже не подозревают.
