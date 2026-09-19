# 45. Elasticsearch: distributed search и analytics

## Зачем понимать ES глубже basic tutorials

Разработчик впервые encountering Elasticsearch обычно видит его как «JSON-based search engine». Идёт tutorial — POST document, GET search, results returned. Кажется just another database с different API. Но это understanding hides fundamental differences которые определяют когда Elasticsearch правильный выбор а когда нет. И более importantly — determines correctly usage patterns.

Elasticsearch не database в traditional relational sense. Это distributed search plus analytics engine поверх Lucene. Data model fundamentally different — inverted indexes rather than B-trees. Query semantics — full-text search with relevance scoring rather than exact match. Consistency model — eventual (near-real-time refresh) rather than immediate. Understanding эти differences ключ к effective usage.

Разница между разработчиком «использующим Elasticsearch» и «понимающим Elasticsearch» очевидна в architectural decisions. Первый использует ES как general-purpose data store — CRUD operations, joins across indexes, expects immediate consistency. Второй знает что ES optimized для massive read scale plus full-text search — write throughput lower than relational databases, joins between indexes very limited (parent-child, nested — costly), refresh interval means writes not immediately searchable. Знает что shard count критический decision made at index creation time — cannot change without reindex. Знает что text vs keyword field types имеет fundamental implications для how field usable в queries.

В этом файле разберём ES с этой deep perspective. Основные концепты и как отличаются от relational databases. Inverted index — сердце full-text search. Analyzers pipeline transforming text к tokens. Mapping — schema plus type semantics. Shards plus replicas distributing data и обеспечивая HA. Node roles — master, data, coordinating, ingest. Segments — physical storage units и Lucene mechanics. Query DSL с match/term/bool/aggregations. Индексация — bulk API plus refresh policies. ILM для logs retention. Java клиенты. Common problems и best practices.

## Что такое Elasticsearch fundamentally

Elasticsearch это distributed search plus analytics engine. Написан на Java, built on top of Apache Lucene.

Apache Lucene — библиотека для full-text indexing и searching. Reasonably mature — dates back к 1999. Provides fundamentals — inverted indexes, BM25 scoring, text analysis. Elasticsearch adds distribution layer — sharding, replication, cluster coordination, REST API — turning Lucene от library в scalable distributed system.

Data model. JSON documents. Each document self-contained JSON object с fields. Not strict schema by default — can be permissive («dynamic mapping»). Или strict schema через explicit mapping.

Query model. Rich query DSL — full-text match, exact term, range, boolean combinations, aggregations, scoring functions. REST-based API — HTTP requests с JSON bodies.

Distributed by design. Data automatically partitioned across nodes через sharding. Replication for availability и read scaling. Coordination between nodes handled automatically.

Использование в множестве scenarios. Logs (ELK stack) — most common enterprise use. Full-text search на websites, documentation. Analytics dashboards. APM plus metrics (Elastic Observability suite). Security plus SIEM (event data analysis).

Не универсальный replacement для RDBMS. Poor at ACID transactions (нет transactions actually). Joins limited. Write throughput lower than optimized RDBMS. Immediate consistency not default. Правильное usage — как secondary specialized store для search plus analytics workloads.

## Основные концепты

Document — JSON объект. Contains fields как name-value pairs. Self-contained — не references to other documents (только through custom logic).

Index — набор documents. Analogous к table в relational world. Каждый document belongs к one index.

Type — устарел с ES 7. Раньше был «table within index» — single index could have multiple types. Removed для simplification. Каждый index now has one implicit type.

Shard — часть индекса. Данные распределены по shards. Каждый shard — independent Lucene index.

Replica — копия shard на другом node. Для HA plus read scaling.

Node — один экземпляр ES process.

Cluster — набор nodes работающих вместе.

## Inverted index — сердце full-text search

Классический индекс database maps id → value. Enables lookup by id.

Elasticsearch reverses это. Inverted index maps word → list documents where word appears.

Documents:
```
doc1: "Kafka is a distributed streaming platform"
doc2: "Elasticsearch is a distributed search engine"
```

Inverted index:
```
"kafka"          → [doc1]
"is"             → [doc1, doc2]
"a"              → [doc1, doc2]
"distributed"    → [doc1, doc2]
"streaming"      → [doc1]
"platform"       → [doc1]
"elasticsearch"  → [doc2]
"search"         → [doc2]
"engine"         → [doc2]
```

Поиск "distributed" — instantly returns [doc1, doc2]. No scanning documents. Direct lookup в index.

Поиск "distributed streaming" — intersection of postings lists. distributed → [doc1, doc2]. streaming → [doc1]. Intersection — [doc1].

Отсюда ES extremely fast для full-text search. Complexity O(log N) для term lookup plus O(k) для intersection where k is postings list length. Massively faster than scanning all documents.

Cost — indexing time. Adding document requires updating multiple postings lists (one per term in document). Slower writes than key-value stores. But queries dramatically faster.

## Analyzers pipeline

How text превращается в tokens (для inverted index). Multi-stage pipeline:

Character filter — preprocessing raw text. Strip HTML tags. Replace patterns. Например «<h1>Title</h1>» becomes «Title».

Tokenizer — разбить на tokens. Standard tokenizer splits on whitespace plus punctuation. N-gram tokenizer creates overlapping sequences для partial matching. Others for specific languages or scenarios.

Token filters — process tokens. Lowercase — «QUICK» becomes «quick». Stop words removal — remove «the», «a», «is». Stemming — «running» becomes «run». Synonyms — «car» matches «automobile».

Пример standard analyzer для «The Quick Brown Fox»:
1. Tokenize: ["The", "Quick", "Brown", "Fox"].
2. Lowercase: ["the", "quick", "brown", "fox"].
3. Stop words: ["quick", "brown", "fox"] (убрали "the").

Индексируются tokens. При поиске — тот же analyzer applied к query terms. Query «Quick Fox» tokenizes to ["quick", "fox"] matching indexed tokens.

Custom analyzers для specific languages, domain-specific patterns, autocomplete (edge_ngram), synonym expansion. Russian analyzer handles Russian language morphology. Custom synonym filter maps «USA» to «United States» to «America».

Analyzer choice affects search behavior fundamentally. Wrong analyzer — searches don't match expected documents. Right analyzer — natural language queries work intuitively.

## Mapping — schema

Определяет типы полей в index. Может быть explicit (recommended для production) или dynamic (inferred from first document — convenient но dangerous).

Explicit mapping:
```json
PUT /orders
{
  "mappings": {
    "properties": {
      "id": { "type": "long" },
      "customer": { "type": "keyword" },
      "description": { "type": "text", "analyzer": "russian" },
      "amount": { "type": "double" },
      "created_at": { "type": "date" },
      "tags": { "type": "keyword" }
    }
  }
}
```

Ключевые types.

text — full-text search field. Analyzed при indexing и querying. Used для description, content, comments — anything semantic search desired. Can't be used efficiently для sort или exact aggregation because tokenized.

keyword — exact match field. Not analyzed — stored as-is. Used для filters, aggregations, sort. Identifiers, status codes, tags — anywhere exact match needed.

long/integer/double — numeric types. Standard numeric semantics. Efficient range queries.

date — dates plus timestamps. Multiple formats accepted. Range queries work naturally. Automatic parsing common formats plus custom formats configurable.

boolean — true/false.

object — nested JSON. Fields accessed через dot notation. Not preserved as unit — each field independent.

nested — arrays of objects preserving relationships. When array elements need to be searched as atomic units. Более costly than object но preserves semantics.

geo_point — geographical location. Enables geo-spatial queries — points near a location, within bounding box.

Critical distinction text vs keyword. Same field можно have both через multi-fields:
```json
"customer": {
  "type": "text",
  "fields": {
    "keyword": { "type": "keyword" }
  }
}
```

Тогда customer используется для search (analyzed tokens), customer.keyword для aggregations plus filters (exact). Common pattern для fields нуждающихся в both usage patterns.

Dynamic mapping — если field appears в document без explicit mapping, ES infers type. String — text plus keyword multi-field default. Numbers — long или double. Dates — date if matches recognized pattern.

Problems с dynamic. First document determines type. Later document with different value type causes conflict — document rejected. Explicit mapping для important indexes prevents these issues.

## Shards и replicas распределяющие data

Shard это часть index. Каждый index divided в N shards. Каждый shard — независимый Lucene index physically stored на some node.

```
Index "orders" (3 shards)
  ├─ Shard 0
  ├─ Shard 1
  └─ Shard 2
```

При индексации document — routing (обычно hash(_id) modulo shard count) выбирает shard. Same document ID always goes к same shard.

При поиске — query executed against ALL shards в parallel. Each shard searches its subset. Results merged coordinated node. Response returned client.

Purpose sharding — scalability. Больше shards → более parallelism → higher throughput possible. Но each shard has overhead — memory, file handles, coordination costs.

Правило shard size 10-50 GB оптимально. Много small shards → high overhead per shard. Мало huge shards → slow queries because не enough parallelism.

Critical caveat — количество shards НЕЛЬЗЯ изменить after index creation. Only через reindex — creating new index с different shard count и copying data. Plan carefully.

Estimation формула. Expected total data size / target shard size = shard count. Для 300 GB data plus 20 GB per shard → 15 shards. Round up к preserve headroom.

Replica это копия shard. По умолчанию 1 replica (primary plus one copy):
```
Cluster (3 nodes)
  ├─ Node 1: shard 0 primary, shard 1 replica
  ├─ Node 2: shard 1 primary, shard 2 replica
  └─ Node 3: shard 2 primary, shard 0 replica
```

ES automatically distributes primaries plus replicas ensuring no node has both primary и replica same shard (that would defeat HA purpose).

Плюсы replicas. HA — при node down replica takes over как primary. Read scaling — read queries могут go к primary или replicas.

Минусы. Storage × (1 + replica count). Indexing overhead × (1 + replica count) because writes replicated.

Правило — 1 replica для prod default. 2 replicas для extremely critical data. 0 для non-critical или single-node dev setups.

Cluster state — Green/Yellow/Red. Green — все primary plus replicas assigned. Yellow — все primary есть но не все replicas assigned (single-node ES, node down). Red — некоторые primary shards missing → data unavailable → data loss risk.

Yellow с single-node deployments common — replicas cannot be allocated because there's no second node. Solution — either add second node или set number_of_replicas=0 для index.

## Node roles

Node может выполнять roles. Multiple roles могут combined на one instance. Или dedicated nodes с single role.

Master-eligible node — может быть выбран master. Master manages cluster state — index creation/deletion, shard allocation, node membership tracking. Один master в cluster в any time. Others eligible ждут election.

Правило — 3 или 5 master-eligible для HA plus split-brain prevention. Election через voting configuration requires quorum. Even numbers problematic — 2 requires unanimous, 4 requires 3 (same as 3-node с less redundancy).

Data node — хранит shards, обслуживает queries. Actual work of storing и querying happens here. Scalable horizontally.

Coordinating node — принимает client requests, forwards к data nodes, merges results. Every node can act as coordinating by default. Dedicated coordinating nodes offload this work when cluster large.

Ingest node — pre-processing документов через ingest pipelines. Optional stage before indexing — parse, transform, enrich data.

Configuration в yml:
```yaml
node.roles: [master, data]     # master плюс data
node.roles: [data]              # только data
node.roles: [master]            # только master
node.roles: [ingest]
node.roles: []                  # только coordinating
```

Для small clusters — combined roles common. Для larger production — separated roles recommended для performance isolation.

## Segments и Lucene mechanics

Внутри shard — набор Lucene segments (immutable файлы). Each segment self-contained mini-index.

Indexation flow:
1. Document arrives.
2. Placed в in-memory buffer.
3. Each refresh_interval (default 1 sec) — new segment created, written to disk, becomes searchable.
4. Периодически segments merge — small segments combined into larger.

Refresh vs flush operations. Refresh (1 sec default) — new segment появляется как searchable. Не пишет на disk actually (только memory plus translog reference). Near real-time search — writes searchable in ~1 second.

Flush (~30 minutes или when translog full) — actually writes segments to disk plus fsync plus clear translog. Durable persistent form.

Translog — WAL-like для durability. Записывается на диск synchronously per operation (или async depending settings). Provides recovery если crash between refreshes. Similar concept к WAL в databases.

Near real-time (NRT) — между index и searchable ~1 second. Not immediate as in RDBMS transactions. Trade-off для write throughput.

Для bulk-load часто оптимизация — увеличить refresh_interval к 30s или даже -1 (disable during load). Then explicitly refresh после completion. Reduces overhead частого segment creation.

## Query DSL

REST API plus JSON queries. Rich language для expressing complex searches.

Match query для full-text search:
```json
POST /orders/_search
{
  "query": {
    "match": {
      "description": "быстрая доставка"
    }
  }
}
```

Analyzer разбирает "быстрая доставка" — токены [быстр, доставк] после stemming — searches для any documents where these tokens appear. Fuzzy semantic search.

Term query для exact match:
```json
{
  "query": {
    "term": {
      "customer.keyword": "berik"
    }
  }
}
```

Точное совпадение. Не analyzed. Requires keyword field. Использование для identifiers, status codes.

Bool query combines multiple:
```json
{
  "query": {
    "bool": {
      "must":     [ { "match": { "description": "заказ" } } ],
      "filter":   [ { "term":  { "status": "active" } } ],
      "should":   [ { "match": { "priority": "high" } } ],
      "must_not": [ { "term":  { "deleted": true } } ]
    }
  }
}
```

Ключевое difference between must и filter. must — обязательно plus влияет на score (contributes к relevance). filter — обязательно, не влияет на score (кэшируется — efficient для repeated identical filters). should — по возможности (boost score если matches). must_not — исключить documents.

Filter versus must performance. Filter cached automatically. Repeated filter queries hit cache. Must recomputed each query. Правило — use filter для exact match constraints, must для scoring-relevant conditions.

Range query для numeric/date ranges:
```json
{
  "query": {
    "range": {
      "created_at": {
        "gte": "2026-01-01",
        "lte": "2026-12-31"
      }
    }
  }
}
```

Aggregations для analytics:
```json
{
  "aggs": {
    "orders_by_status": {
      "terms": { "field": "status.keyword" }
    },
    "total_amount": {
      "sum": { "field": "amount" }
    },
    "amount_stats": {
      "stats": { "field": "amount" }
    }
  }
}
```

Analog SQL GROUP BY plus aggregate functions. terms — group by field value. sum, avg, min, max, stats (all combined). Nested aggregations для sub-groupings.

Aggregations powerful но expensive. Especially cardinality-based (unique counts). Aggregation performance depends много from index design и field types.

Scoring через BM25 (default). Учитывает term frequency (TF — сколько раз token в document), inverse document frequency (IDF — редкий token has higher weight), length normalization (short document с match has higher score than long).

Custom scoring через function_score или script_score для business-specific relevance rules.

## Индексация

Один документ:
```
POST /orders/_doc
{
  "id": 1,
  "customer": "berik",
  "amount": 100.50,
  "created_at": "2026-09-05T10:00:00Z"
}
```

Bulk indexing для performance:
```
POST /_bulk
{ "index": { "_index": "orders", "_id": "1" } }
{ "customer": "berik", "amount": 100.50 }
{ "index": { "_index": "orders", "_id": "2" } }
{ "customer": "alice", "amount": 200.00 }
```

Тысячи документов одним запросом. Обязательно для performance при bulk loads. Order of magnitude faster than individual indexing.

Refresh policy control:

wait_for — ждать refresh перед возврат. Гарантирует indexed doc immediately searchable. Slow для high-volume но provides read-after-write consistency.

true — immediate refresh. Extremely slow. Только для tests или debugging.

false — не форсировать (default). Doc searchable eventually через regular refresh interval. Best performance.

## ILM (Index Lifecycle Management)

Автоматически перемещает indices между phases based on age или size. Critical для logs retention.

Phases стандартные. Hot — новые, active, читаются часто, на fast SSD. Warm — старее, read-mostly, на cheaper storage. Cold — старые, редко читаемые, could use S3 archive tier. Delete — удалить.

Классическая configuration для logs. Indices daily (logs-2026.09.05, logs-2026.09.06). Hot: последние 7 days. Warm: 30 дней. Delete: 90 дней. Rollover — new index created когда current достигает size или age threshold.

ILM automatically transitions indices between phases plus deletes when scheduled. Reduces operational overhead manually managing log retention.

## Java клиенты

REST Client basic:
```java
RestClient client = RestClient.builder(new HttpHost("es", 9200)).build();
Request req = new Request("GET", "/orders/_search");
req.setJsonEntity("{...}");
Response resp = client.performRequest(req);
```

Low-level. Direct HTTP calls. Manual JSON handling. Не type-safe. Useful only для custom scenarios.

Elasticsearch Java Client (новый, 8+):
```java
ElasticsearchClient client = new ElasticsearchClient(transport);

SearchResponse<Order> resp = client.search(s -> s
    .index("orders")
    .query(q -> q.match(m -> m.field("description").query("заказ"))),
    Order.class);
```

Type-safe API. Fluent builder pattern. Preferred для new projects в ES 8+.

RestHighLevelClient устарел в 8+. Long time was standard. Replaced by new Elasticsearch Java Client. Migration требуется для upgrading к ES 8.

Spring Data Elasticsearch аналог Spring Data JPA но для ES:
```java
@Document(indexName = "orders")
class Order {
    @Id String id;
    @Field(type = FieldType.Keyword) String customer;
    @Field(type = FieldType.Double) double amount;
}

interface OrderRepository extends ElasticsearchRepository<Order, String> {
    List<Order> findByCustomer(String customer);
}
```

Declarative repositories. Automatic query generation from method names. Convenient для standard CRUD но complex queries требуют explicit query building.

## Kibana

UI для ES. Разработан Elastic вместе с Elasticsearch. Standard visualization layer.

Возможности. Discover — full-text search plus filtering documents (типичное logs viewing). Visualizations — charts, graphs, maps. Dashboards — collections visualizations. Dev Tools — REST console для direct API queries. Alerts — condition-based notifications. Machine Learning — anomaly detection в data.

Analog Grafana но специализирован под ES data. Grafana более general (multiple datasources including ES) но Kibana имеет deeper ES integration.

Enterprise features (Elastic Stack Basic License plus paid tiers) — machine learning, alerting, security, cross-cluster search. Free Basic license adequate для most needs.

## ELK stack

ELK = Elasticsearch plus Logstash plus Kibana. Стандарт для centralized logging.

С добавлением Beats — «Elastic Stack». Beats это family lightweight shippers replacing Logstash для simple use cases.

Typical flow:
```
App → Filebeat (reads log files)
        │
        ▼
   [Logstash pipeline]  ← парсинг, enrichment
        │
        ▼
   [Elasticsearch]
        │
        ▼
     [Kibana]  ← ты смотришь
```

Или проще: Filebeat → Elasticsearch → Kibana (без Logstash для простых cases). Filebeat достаточно если no transformation needed.

Logstash tends to be heavy (JRuby-based). Filebeat lightweight (Go-based). Trend — прefer Filebeat direct к Elasticsearch, используя Elasticsearch's ingest pipelines для transformations вместо Logstash.

В КНП. Memory knp-prod-historical-logs-elk — свой ELK кластер (isna-elk). Логи по-старому через kubectl exec в pod ES для historical queries. kubectl logs = только сегодня (rotation), за вчера — только через ES.

Логи в поле log — match_phrase по дефисным кодам плохо работает потому что analyzer tokenizes on hyphens. Нужно искать по токенам plus использовать keyword field для exact matches.

## Проблемы и типовые кейсы

Yellow cluster — replicas не назначены. Single-node ES plus replicas=1 → yellow всегда потому что нет второго node. Solution — number_of_replicas=0 для single-node deployments. Или добавить node.

Red cluster — некоторые primary shards недоступны. Data loss риск. Recovery — restore nodes hosting failed shards, snapshot restore если necessary.

Медленный поиск — множество possible причин. Слишком большой shard (single shard scanning takes long). Слишком много shards (parallelization overhead exceeds benefits). Нет filters — full scan indexes. Слабое железо не keeping up with load. Sort на text field (не имеет doc_values by default, expensive) — use keyword variant для sort.

Profiling через ?pretty&profile=true shows exact time breakdown query execution. Identifies slow parts.

Cardinality на aggregations. Aggregation по high-cardinality field (userId) — many buckets — OOM potential. Fix через terms.size ограничение (limit к top N), composite aggregation для pagination through all buckets.

Mapping конфликт. Индексируешь field one type, потом another document has different type для same field → conflict → document rejected. Solution — explicit mapping для важных indexes, не полагаться на dynamic mapping для production.

Slow bulk indexing. Optimizations. Уменьшить refresh_interval к 30s или -1 during bulk load. Больше batch size (10-50 MB per bulk request). Distribute across coordinating nodes — не через один. После bulk — reset refresh_interval к 1s, force merge если needed.

## Best practices

Правила проверенные production experience.

Explicit mapping для важных индексов. Не полагаться на dynamic mapping. Определить types upfront.

text для search, keyword для filter/sort/aggregate. Часто multi-fields (text plus keyword одновременно) для fields нуждающихся в both.

Shard size 10-50 GB. Estimate upfront based на expected total data plus growth.

number_of_shards продумать заранее. Cannot change without reindex. Overprovision слегка allows growth.

1 replica default. Balance HA plus storage cost.

Bulk indexing для performance. Individual document indexing extremely slow at scale.

ILM для logs plus other time-based data. Automatic tier transitions и retention.

Snapshots к S3 или blob storage для backup. Regular snapshots critical для disaster recovery.

3 или 5 master-eligible nodes. Quorum-based election.

Monitoring standard metrics — JVM, indexing rate, search latency, shard health, disk usage. Alerting on anomalies.

## Итоги

Elasticsearch это distributed search plus analytics engine поверх Lucene. Не general-purpose database — optimized для search и analytics.

JSON documents. Rich mapping определяет field types plus behaviors.

Inverted index — сердце full-text search. Maps word → documents. O(log N) lookup дramatically faster than table scans.

Analyzers pipeline — character filters, tokenizer, token filters — transform text к indexed tokens. Language-specific analyzers plus custom for domain needs.

Mapping schema. text для search analyzed. keyword для exact match, sort, aggregate. Multi-fields часто combining both.

Sharding distributes data. Cannot change after index creation. Shard size 10-50 GB target.

Replicas для HA plus read scaling. 1 replica default production.

Green/Yellow/Red cluster states indicating health.

Node roles — master (management), data (storage plus query), coordinating (request routing), ingest (pre-processing). Combined in small clusters, separated в large production.

Segments — immutable Lucene files. Refresh creates new searchable segment. Flush persists к disk. Merge combines small segments. Near real-time — refresh interval ~1s determines search visibility.

Query DSL — match для full-text, term для exact, bool для combinations, aggregations для analytics. must vs filter — filter cached plus не affecting scoring.

Bulk indexing обязательно для performance. Refresh policies balance consistency и throughput.

ILM для automatic index lifecycle. Hot/warm/cold/delete phases.

Java client (new one для ES 8+), Spring Data Elasticsearch для declarative repositories.

Kibana как UI. ELK stack (или Elastic Stack с Beats) — standard для logs.

Best practices — explicit mapping, right field types, appropriate shard sizing, replicas, bulk indexing, ILM, monitoring.

Дальше — nginx как reverse proxy, load balancer, web server, критический компонент infrastructure.
