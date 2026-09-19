# 95. Когда что выбрать: PostgreSQL vs NoSQL, on-prem vs cloud

## Постановка проблемы

Мир баз данных перестал быть простым «SQL или не SQL». В 2010-х произошёл взрыв разных типов хранилищ — MongoDB, Cassandra, Redis, DynamoDB, Neo4j, Elasticsearch, ClickHouse, BigQuery, Snowflake. Каждая позиционировала себя как решение конкретных проблем, для которых реляционные базы «плохо подходят». Через десять лет картина стала яснее: NoSQL не заменил SQL, но дополнил. Для разных задач разные инструменты работают лучше.

Senior-инженер должен понимать этот landscape. Не для того, чтобы использовать все технологии сразу — обычно это плохая идея (полиглотическая архитектура — знаменитая ловушка). А чтобы принимать осознанные решения: для конкретной задачи PostgreSQL хорош, для другой лучше Redis, для третьей MongoDB не даёт преимуществ и лучше остаться на PostgreSQL. Понимать trade-off'ы, а не выбирать по моде.

Параллельная тема — где эту базу разворачивать. On-premise (свой сервер, свой дата-центр) даёт полный контроль и потенциально дешевле, но требует команды сисадминов и DBA. Managed cloud services (AWS RDS, Aurora, Google Cloud SQL, AlloyDB, Azure Database) — берут на себя всю операционку, но за это платят подписку, и есть ограничения. Serverless databases (Aurora Serverless, PlanetScale, Neon) идут дальше — платишь только за использование, автоскейлинг. У каждого варианта свои плюсы и минусы, и правильный выбор зависит от масштаба, команды и требований.

В этом файле разберём типы NoSQL баз, их сильные и слабые стороны, конкретные use cases, где они действительно нужны, а где PostgreSQL справляется не хуже. Потом пройдёмся по managed cloud databases: AWS, GCP, Azure, отдельные игроки; сравним стоимость, функциональность, ограничения. Дадим фреймворк выбора для реальных ситуаций.

## Как классифицировать NoSQL

Термин NoSQL — маркетинговый, объединяющий разнородные технологии. Реально это семейство разных типов баз:

**Document databases** — MongoDB, CouchDB, Firestore. Хранят JSON-подобные документы, каждый со своей схемой. Хороши для полу-структурированных данных.

**Key-value stores** — Redis, DynamoDB, Riak. Хранят пары ключ-значение с быстрым lookup по ключу. Хороши для кэша, сессий, простых lookup'ов.

**Wide-column stores** — Cassandra, ScyllaDB, HBase. Хранят таблицы, но с гибкой structure — каждая строка может иметь свой набор колонок. Оптимизированы для write-heavy нагрузок и горизонтального масштабирования.

**Graph databases** — Neo4j, ArangoDB. Хранят узлы и связи между ними, оптимизированы для запросов по графам (найти всех друзей друзей, пути между узлами).

**Search engines** — Elasticsearch, Solr, OpenSearch. Специализированы для полнотекстового поиска и аналитических запросов.

**Column stores для аналитики** — ClickHouse, Vertica, BigQuery, Redshift, Snowflake. Как обсуждали в 87, оптимизированы для OLAP.

**Time-series databases** — InfluxDB, TimescaleDB, Prometheus. Специализация для временных рядов.

**Vector databases** — Pinecone, Weaviate, Milvus, pgvector. Для embeddings и similarity search в AI приложениях.

Каждый тип решает специфический класс задач лучше чем реляционные базы. Но реляционные базы, особенно PostgreSQL, покрывают достаточно широкий спектр — и часто лучше выбрать одну хорошую базу с некоторыми компромиссами, чем десять специализированных с operational overhead.

## MongoDB и document databases

MongoDB — самая известная NoSQL база, часто первая, о которой думают при упоминании «нам нужна гибкость». Хранит документы в BSON (Binary JSON), поддерживает embedded documents (вложенные структуры), массивы, разные типы данных в одном поле разных документов.

Пример документа:

```json
{
  "_id": ObjectId("..."),
  "username": "berik",
  "email": "b@example.com",
  "addresses": [
    { "type": "home", "city": "Astana", "street": "..." },
    { "type": "work", "city": "Almaty", "street": "..." }
  ],
  "metadata": {
    "signup_source": "google_ads",
    "referral": null
  },
  "last_login": ISODate("...")
}
```

Плюсы MongoDB. Схема гибкая — можно менять структуру документов без миграций. Хорошо ложится на объектную модель приложений (документ = агрегат в DDD). Горизонтальное масштабирование (auto-sharding). Хорошая производительность для CRUD по _id.

Минусы. Отсутствие полноценных JOIN'ов — приходится либо денормализовывать (дублировать данные в разных документах), либо делать multiple queries. Транзакции многодокументные появились относительно недавно (4.0+), но с ограничениями. Consistency — по умолчанию eventual, strong требует специальных настроек. Не так хорошо оптимизирована для сложных запросов и агрегаций как SQL.

Реальные use cases для MongoDB.

**Хорошо**: content management systems где документы имеют разные структуры (blog posts, articles), каталоги товаров с разными attribute sets, event logs с гибким payload, user profiles с гибкими полями, real-time analytics dashboards с pre-aggregated data.

**Плохо**: transactional systems с сложной бизнес-логикой, финансовые данные требующие ACID гарантий, отчёты с сложными JOIN'ами, любые задачи где нужна строгая консистентность.

Часто MongoDB выбирают из-за flexibility, а потом жалуются что нельзя нормально сделать JOIN. Это ложный аргумент — PostgreSQL JSONB даёт ту же гибкость плюс SQL для сложных запросов. Для 90% случаев, где рассматривают MongoDB, PostgreSQL + JSONB — лучший выбор.

MongoDB имеет смысл если:
- Реально не знаешь заранее structure данных.
- Ожидаешь horizontal scaling до сотен нод (redko достигается в enterprise).
- Приложение полностью document-oriented (нет relational-heavy queries).
- Команда уже опытна в MongoDB.

## Redis

Redis — in-memory key-value store. Хранит всё в RAM, что делает его невероятно быстрым — миллионы операций в секунду с sub-millisecond latency. Опционально persist на диск (RDB snapshots или AOF log), но основной use case — кэш.

Помимо простых key-value, Redis поддерживает много structures: strings, hashes, lists, sets, sorted sets, streams, pub/sub, geographical indices, HyperLogLog для approximate counting. Это делает его универсальным инструментом для многих задач.

Классические use cases Redis.

**Кэширование**. Самый распространённый. Приложение хранит часто читаемые данные (сессии пользователей, результаты дорогих запросов) в Redis. TTL для автоматической expiration. Cache invalidation при изменениях.

**Session storage**. Сессии HTTP приложений часто хранят в Redis. Быстро, распределяется между инстансами приложения.

**Rate limiting**. Redis-based rate limiter — считает количество запросов в скользящем окне, incremental counters, atomic operations.

**Distributed locks**. Redis-based mutex для координации между несколькими процессами.

**Queues**. Через lists (LPUSH/BRPOP) или streams. Не столь надёжно как RabbitMQ/Kafka, но проще и быстро.

**Pub/Sub**. Publish-subscribe между приложениями.

**Leaderboards**. Sorted sets с score позволяют efficient top-N queries.

Плюсы. Невероятная скорость. Много useful structures. Проверенная в продакшене (используется Twitter, GitHub, Stack Overflow).

Минусы. Данные в RAM — ограничение памяти. Persistence слабее чем у реляционных баз (можно потерять последние секунды при crash). Не подходит для primary storage больших данных.

Redis почти всегда используется **вместе** с реляционной базой, не вместо неё. PostgreSQL — source of truth, Redis — cache и specialized structures. Правильная роль Redis — оптимизация специфических случаев, не хранение бизнес-данных.

Redis Cluster — horizontal scaling для больших deployments. Redis Sentinel — HA для master-slave setup.

## Cassandra и wide-column

Cassandra — распределённая база созданная в Facebook, потом развитая как open-source. Дизайн заточен под massive horizontal scale и write-heavy workloads. Пишешь с сотней тысяч нод, каждая делает миллион writes в секунду. Read более медленный чем PostgreSQL, потому что нужно merge из multiple SSTables (LSM tree).

Модель данных — wide-column: таблицы, но каждая row может иметь свой набор колонок. Primary key разделён на partition key (определяет ноду) и clustering columns (определяют порядок внутри partition).

Плюсы. Linear scalability write throughput. High availability (multi-master, no single point of failure). Tunable consistency (можно выбирать между strong и eventual для каждого запроса).

Минусы. Плохо для read-heavy с complex queries. Нет JOIN'ов вообще. Нет ACID транзакций (обещают в новых версиях, но пока условно). Сложная эксплуатация — Cassandra требует настоящих экспертов.

Реальные use cases. Time-series data with high write rate (metrics, logs, IoT). Message logs (Discord использует Cassandra для истории сообщений). Recommendation engines. Fraud detection.

Для типичных enterprise систем Cassandra — overkill. PostgreSQL с партиционированием справится с гораздо большими нагрузками, чем можно ожидать. Cassandra оправдана только когда действительно нужен write throughput за пределами возможностей одной PostgreSQL машины.

## Neo4j и graph databases

Graph databases специализируются на хранении и запросах в графах — узлах и связях. Оптимизированы для queries вроде «найти друзей друзей», «кратчайший путь», «сообщества в графе».

В PostgreSQL можно моделировать графы через таблицы (nodes и edges), но queries сложные (multiple self-joins для многих hop'ов) и медленные. Neo4j хранит edges как first-class objects с pointers на nodes, что делает traversal быстрым.

Cypher query language:

```cypher
MATCH (p:Person)-[:FRIEND]->()-[:FRIEND]->(fof:Person)
WHERE p.name = 'Berik'
RETURN DISTINCT fof.name
```

«Друзья друзей Berik'а».

Плюсы. Отлично для graph queries: social networks, recommendation engines, fraud detection (обнаружение сетей связанных мошенников), knowledge graphs.

Минусы. Не подходит для не-графовых данных. Меньшая экосистема чем реляционные базы. Requires specialized knowledge.

Для типичных enterprise систем graph database редко нужна как primary. Иногда — как secondary для конкретной feature (recommendation engine).

## Elasticsearch и search

Elasticsearch — распределённый search engine на базе Lucene. Специализирован для full-text search, log analytics, сложных фильтров и агрегаций на текстовых данных.

Основной use case — search-heavy applications: e-commerce поиск товаров, поиск документов, анализ логов через ELK stack.

Плюсы. Мощный full-text search (лучше чем PostgreSQL). Быстрая аналитика (aggregations по большим объёмам). Distributed by design. Kibana для визуализации.

Минусы. Не source-of-truth (это search index, не БД). Consistency — eventual. Данные duplicate из primary storage. Operational overhead значительный.

Правильная роль. Primary storage — PostgreSQL. Elasticsearch — search index, синхронизированный через Debezium или periodic sync jobs. Пользователь ищет в Elasticsearch, получает IDs, загружает full data из PostgreSQL.

Для КНП простой search: PostgreSQL FTS достаточно. Если объёмы и требования растут — Elasticsearch как дополнение.

## Column stores для аналитики

Уже обсуждали в 87 (NSM/DSM/PAX). ClickHouse, Vertica, BigQuery, Snowflake, Redshift — все они column-oriented, специализированы для OLAP.

Реальные use cases. Business intelligence dashboards. Marketing analytics. Financial reporting. Log analytics на терабайты.

Плюсы. Невероятная скорость на аналитических запросах (SUM/AVG по миллиардам строк за секунды). Отличное сжатие данных. Часто SQL интерфейс, знакомый разработчикам.

Минусы. Ужасны для transactional workload (нет обновлений строк без огромной пере-записи). Данные duplicate из OLTP source. Дороже per-storage чем реляционные базы.

Правильная архитектура. PostgreSQL как OLTP. Периодический ETL в column store (ClickHouse или BigQuery) для analytics. Разработчики бизнес-логики работают с PostgreSQL, аналитики — с column store.

Для КНП это классический pattern: PostgreSQL для operational, отдельная analytics база (можно ClickHouse on-prem или BigQuery в облаке) для reporting.

## Time-series databases

TimescaleDB — самый интересный вариант, потому что это extension поверх PostgreSQL. Ты получаешь SQL, PostgreSQL надёжность, ecosystem — плюс специальные фичи для time-series: hypertables (auto-partition по времени), continuous aggregates (auto-updating materialized views), compression (10-20x сжатие старых данных), retention policies.

Для КНП это hot pick — если есть time-series нагрузка (метрики, аудит, IoT), TimescaleDB даст лучшее из обоих миров: PostgreSQL под капотом плюс time-series оптимизации.

Prometheus — отдельная лига, специализирован для service metrics. Использует свой pull-model, RRD storage. Стандарт для мониторинга Kubernetes.

InfluxDB — покольник в time-series space, но потерял почву последние годы. Не рекомендуется для новых проектов.

## Vector databases для AI

С развитием LLM появилась новая категория. Задача: хранить embeddings (векторы, репрезентации текстов/изображений) и быстро находить nearest neighbors для similarity search. Основа RAG (retrieval augmented generation) — LLM отвечает на вопросы, находя релевантные документы через vector search.

Специализированные: Pinecone, Weaviate, Milvus, Qdrant. Pgvector — PostgreSQL extension. Elasticsearch тоже добавил vector search.

Плюсы специализированных. Оптимизированы под миллиарды векторов. Продвинутые индексы (HNSW, IVF-PQ). Managed hosting в большинстве случаев.

Плюсы pgvector. PostgreSQL стек, SQL, транзакции, JOIN'ы с текстом. Достаточно для миллионов документов. Дешевле специализированных.

Для КНП, если приложение использует AI features (умный search, recommendations, chatbot с RAG), pgvector внутри PostgreSQL — правильный первый шаг. Если объёмы вырастут за пределы миллионов — можно смотреть на специализированные.

## PostgreSQL как universal database

Если сложить все use cases, PostgreSQL умеет почти всё:

- OLTP — свой хлеб.
- JSON documents — через JSONB.
- Full-text search — встроенный, tsvector.
- Key-value — можно хранить в hstore или JSONB, plus unlogged tables для быстрого доступа.
- Time-series — через TimescaleDB extension.
- Vector search — через pgvector.
- Geo — через PostGIS.
- Аналитика — не так быстро как ClickHouse, но partial coverage через партиционирование и parallel query.

Это делает PostgreSQL хорошим default выбором для большинства проектов. Одна база, одна команда, один стек, одни backup процедуры. Стартап на PostgreSQL может расти несколько лет до момента, когда специализированные решения становятся оправданными.

Правило: **начинай с PostgreSQL, добавляй специализированное когда реально нужно**. Не наоборот. Не «нам нужна MongoDB потому что современно» — а «PostgreSQL плохо решает эту конкретную задачу, вот причины, используем X».

## Managed cloud databases

Переходим к теме hosting'а. Cloud managed сервисы взяли большую часть рынка, потому что убирают operational burden. Управляют backup'ами, patches, replication, scaling, monitoring — команда фокусируется на приложении.

**AWS RDS**. Самый старый managed PostgreSQL (и MySQL, SQL Server, MariaDB, Oracle). Классическая single-instance модель с option для read replicas и Multi-AZ (auto-failover). Простая и понятная. Дешевле аналогов до определённого размера.

**AWS Aurora**. Custom storage engine, разработанный AWS. Отделяет compute от storage — storage автоматически replicated 6-way между availability zones, автоматически backup, автоматически репликация. Compute (writer + reader) может scaling'ться независимо. Совместимость с PostgreSQL (или MySQL) — драйверы, SQL, extensions работают.

Плюсы Aurora. Автоматическое storage scaling до 128 TB. Быстрая репликация (единый storage). Read replicas почти без replication lag. Aurora Serverless — pay per usage вместо inst types.

Минусы. Дороже RDS для маленьких инстансов. Vendor lock-in (Aurora API не 100% PostgreSQL compatible в edge cases). Некоторые extensions недоступны.

**Google Cloud SQL**. Аналог RDS от Google. Meta-simpler чем Aurora, но чуть меньше фич.

**Google AlloyDB**. Ответ Google на Aurora. PostgreSQL-совместимый, distributed storage, columnar engine для analytical queries. Молодой продукт, растёт.

**Azure Database for PostgreSQL**. Аналог от Microsoft, две модели: Flexible Server (ближе к RDS) и Hyperscale (partitioning based на Citus, sharded).

**Managed services от других игроков**. Neon (serverless PostgreSQL с branching как в git), Supabase (PostgreSQL + auth + realtime + APIs), Timescale Cloud (managed TimescaleDB), Crunchy Data (enterprise PostgreSQL).

## Стоимость и compare

Стоимость сильно зависит от требований, но общие правила.

**Small (до сотни GB, стандартная нагрузка)**: RDS или Cloud SQL — самое дешёвое. Aurora overkill.

**Medium (сотни GB, значительная нагрузка)**: Aurora показывает преимущества (лучше performance для той же цены, лучше HA).

**Large (терабайты, высокая нагрузка)**: Aurora, AlloyDB, специализированные. Стоимость становится проблемой, надо оптимизировать (right-sizing instances, использование reserved capacity, right storage type).

**Variable load**: Aurora Serverless V2, Neon — pay per usage, отлично для dev/staging и spiky production.

Cloud managed databases обычно **в 2-3 раза дороже** self-hosted на VMs. Разница — плата за operational simplicity. Для маленькой команды без DBA это оправдано. Для большой enterprise с командой DBA — self-hosted может быть выгоднее.

## Ограничения managed сервисов

Managed услуги имеют ограничения, которых нет в self-hosted.

**Ограниченный доступ**. Нет root доступа к OS, нет прямого доступа к файлам, нет полного суперпользователя в PostgreSQL. Некоторые операции недоступны.

**Ограниченные extensions**. Не все extensions установлены. Обычно есть pg_stat_statements, pgcrypto, PostGIS. Специфические (pg_repack, pgAudit некоторых версий) могут отсутствовать.

**Ограниченные настройки**. Некоторые postgres.conf параметры недоступны или доступны через специальный API. Debug options часто отключены.

**Vendor lock-in**. Особенно Aurora — миграция обратно на обычный PostgreSQL требует работы.

**Backup ограничения**. Backup сохраняется в проприетарном формате, извлечь и unstruct'ить может быть сложно.

**Compliance**. Для строгих compliance (data sovereignty, конкретные certifications) managed сервисы могут не подходить.

Для КНП государственная система с data sovereignty требованиями — managed cloud вряд ли применимо, on-prem или private cloud единственный вариант.

## Serverless databases

Новая категория — базы, где ты не думаешь про инстансы вообще. Платишь только за использованное хранение и запросы.

**Aurora Serverless V2** — auto-scales от 0.5 до 128 ACU (Aurora Capacity Units). Скалирование за секунды. Хорош для variable workloads.

**Neon** — PostgreSQL с separation of storage and compute, branching (создать копию базы для feature branch, как в git), scale to zero. Молодой, для новых проектов.

**PlanetScale** — MySQL-совместимый (Vitess based). Branching, non-blocking schema changes. Не для PostgreSQL мира, но интересный дизайн.

**Cloudflare D1** — SQLite as service, всё простой но масштабирует для edge use cases.

**CockroachDB Serverless** — distributed SQL, PostgreSQL wire compatible, distributed transactions.

Serverless хорош для: startups (не платишь за idle), dev/staging (много окружений, редко используются), unpredictable traffic (пики без provisioning). Плохо для: predictable high-load production (dedicated instance дешевле), when consistent latency критична.

## Фреймворк выбора

Как реально выбирать. Начни с вопросов.

**Каков масштаб?** Стартап с 10 пользователей vs enterprise с миллионами — совершенно разные ответы. Маленький масштаб — начни с самого простого (PostgreSQL на RDS или self-hosted). Большой — рассматривай специализированные решения.

**Что за workload?** OLTP (транзакции) — реляционка. Analytics — column store или TimescaleDB. Search — Elasticsearch или PG FTS. Кэш — Redis. Graph queries — сначала PostgreSQL с recursive CTE, если не хватит — Neo4j.

**Есть ли уже expertise?** Команда с опытом MongoDB — использовать MongoDB может быть быстрее, даже если PostgreSQL «лучше». Экспертиза важнее theoretical fitness.

**Есть ли DBA?** Есть — self-hosted OK. Нет — managed cloud, чтобы не тонуть в operations.

**Compliance и regulatory?** Data sovereignty — on-prem. Standard cloud — managed OK.

**Cost sensitivity?** Startup с runway — cheapest option (обычно self-hosted или serverless). Enterprise с бюджетом — managed для reduced risk.

**Специализированные требования?** Real-time analytics — ClickHouse. AI/RAG — pgvector или Pinecone. Massive time-series — TimescaleDB. Общие — PostgreSQL.

Правило одно: **начинай с PostgreSQL для 90% случаев**, добавляй специализированное когда есть конкретная причина. Не «модно потому что». Причина должна формулироваться конкретно: «PostgreSQL плохо решает эту задачу, вот EXPLAIN, вот метрики, вот compared с X, X лучше на N».

## Заключение

NoSQL — не замена SQL, а комплементарная категория. Разные базы решают разные задачи. MongoDB для document-heavy, Redis для in-memory кэша и specialized structures, Cassandra для massive write throughput, Elasticsearch для search, ClickHouse для аналитики, pgvector для AI. Каждая имеет свою сферу применения.

PostgreSQL — de facto универсальный default. JSONB для гибкости, FTS для search, TimescaleDB extension для time-series, pgvector для vectors, PostGIS для гео. Одна база покрывает 90% случаев. Специализированные решения — только когда PostgreSQL реально плохо справляется и это доказуемо.

Cloud managed services убрали operational burden. RDS/Cloud SQL — простые, дешевле. Aurora/AlloyDB — продвинутые, distributed storage, лучше performance/HA. Serverless (Aurora Serverless V2, Neon) — pay per usage, для variable workload. Ограничения managed: нет root, ограниченные extensions, vendor lock-in.

Стоимость managed — в 2-3 раза дороже self-hosted, но экономия на команде DBA обычно оправдывает. Для больших enterprise с командой DBA self-hosted выгоднее. Для startup и mid-size — managed правильный выбор.

Выбор технологии зависит от масштаба, workload, expertise, compliance, cost. Нет правильного ответа в вакууме. Всегда начинай с самого простого (PostgreSQL на RDS для 90% случаев), добавляй сложность только когда есть конкретная причина.

Для КНП правильный выбор: PostgreSQL как primary для всего OLTP, вероятно on-prem из-за compliance, Redis для кэша и сессий, отдельная analytics база (ClickHouse или BigQuery) для reporting если объёмы растут, Elasticsearch если нужен продвинутый search. Не пять NoSQL баз ради ради «современной архитектуры», а обоснованные добавления там, где реально нужны.

Дальше — практика. Попробуй развернуть PostgreSQL на разных managed сервисах (RDS, Aurora, Cloud SQL), замерь стоимость и performance для типичной нагрузки. Установи локально MongoDB, Redis, Elasticsearch, реши какую-то простую задачу на каждой, сравни ergonomics с PostgreSQL. Каждый эксперимент даёт больше понимания, чем читать блог-посты сравнений.
