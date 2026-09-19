# 45. Elasticsearch: индексы, шарды, реплики

Что такое ES, как устроен, зачем в ИСНА (логи + поиск).

---

## 1. Что такое Elasticsearch

**Elasticsearch (ES)** — distributed search + analytics engine.

- Написан на Java, поверх **Apache Lucene**.
- **JSON documents**, schemaless (или strict через mapping).
- **Full-text search** (не только exact match).
- **Aggregations** — sum, avg, group by (аналитика).
- **REST API**.
- **Distributed** — data распределяется по nodes через sharding + replication.

Использование:
- **Логи** (ELK stack).
- **Full-text search** (сайты, документация).
- **Analytics** (dashboards).
- **APM / metrics** (Elastic Observability).
- **Security / SIEM**.

---

## 2. Основные концепты

- **Document** — JSON объект.
- **Index** — набор документов (как таблица в SQL).
- **Type** — устарел с ES 7 (был как «таблица внутри индекса»).
- **Shard** — часть индекса; данные распределены по shards.
- **Replica** — копия shard на другом node.
- **Node** — один экземпляр ES.
- **Cluster** — набор nodes.

---

## 3. Inverted index — основа поиска

Классический индекс БД: id → значение. Elasticsearch наоборот.

**Inverted index**: слово → список документов где встречается.

Документы:
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

Поиск "distributed" → мгновенно даст [doc1, doc2].

Поиск "distributed streaming" → пересечение: [doc1].

Отсюда — ES очень быстрый full-text search.

---

## 4. Analyzers

Как текст превращается в токены (для inverted index).

Analyzer:
1. **Character filter** — предобработка (strip HTML).
2. **Tokenizer** — разбить на токены (по пробелам, N-gram).
3. **Token filters** — lowercase, stop words, stemming, synonyms.

Пример **standard analyzer** для "The Quick Brown Fox":
1. Tokenize: ["The", "Quick", "Brown", "Fox"].
2. Lowercase: ["the", "quick", "brown", "fox"].
3. Stop words: ["quick", "brown", "fox"] (убрали "the").

Индексируются токены. При поиске — тот же analyzer к query.

Custom analyzers: русский язык, синонимы, edge_ngram (для autocomplete).

---

## 5. Mapping — схема

Определяет типы полей.

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

Ключевые типы:
- **`text`** — full-text search, анализируется.
- **`keyword`** — exact match, не анализируется (для filters, aggregations, sort).
- **`long / integer / double`** — числа.
- **`date`** — даты.
- **`boolean`**.
- **`object` / `nested`** — вложенные.
- **`geo_point`** — геопозиция.

**Важно**: `text` для поиска, `keyword` для фильтров. Часто одно поле имеет оба:
```json
"customer": {
  "type": "text",
  "fields": {
    "keyword": { "type": "keyword" }
  }
}
```

Тогда `customer` для search, `customer.keyword` для aggregations.

---

## 6. Shards и replicas

### 6.1 Shard

Каждый индекс делится на N shards. **Каждый shard — независимый Lucene-индекс**.

```
Index "orders" (3 shards)
  ├─ Shard 0
  ├─ Shard 1
  └─ Shard 2
```

При индексации документа — routing (обычно hash(id) % shards) выбирает shard.

При поиске — запрос идёт во **все** shards → каждый ищет → результаты merge.

**Зачем**: масштабирование. Больше shards → распараллеливание.

**Правило**: shard size 10-50 GB оптимально. Много маленьких → overhead. Мало огромных → медленно.

**Кавет**: количество shards **нельзя изменить** после создания индекса. Только через reindex.

### 6.2 Replica

Копия shard. По умолчанию 1 replica.

```
Cluster (3 nodes)
  ├─ Node 1: shard 0 primary, shard 1 replica
  ├─ Node 2: shard 1 primary, shard 2 replica
  └─ Node 3: shard 2 primary, shard 0 replica
```

Плюсы:
- **HA**: node упал → replica становится primary.
- **Read scale**: reads идут и на primary, и на replicas.

Минусы:
- Storage × 2.
- Индексация × 2 (write в primary + replica).

**Правило**: 1 replica для prod (2 — data не критичный).

### 6.3 Yellow / Green / Red

Cluster state:
- **Green** — все primary + replicas распределены.
- **Yellow** — все primary есть, но не все replicas назначены (single-node ES, node down).
- **Red** — некоторые primary не назначены → data loss риск.

---

## 7. Roles nodes

Node может выполнять роли:

### 7.1 Master-eligible

Может быть выбран master. Master управляет cluster state (создание индексов, распределение shards).

Один master в кластере. Остальные master-eligible ждут очереди.

Правило: **3 или 5 master-eligible** (quorum, избежание split-brain).

### 7.2 Data node

Хранит shards, обслуживает queries.

### 7.3 Coordinating node

Принимает запросы клиентов, форвардит на нужные data nodes, merge results.

Все nodes могут быть coordinating по умолчанию. Специализированные — только coordinating (без data / master).

### 7.4 Ingest node

Pre-processing документов через **ingest pipelines** (parse, transform, enrich).

### 7.5 Роли в yml

```yaml
node.roles: [master, data]     # только master + data
node.roles: [data]              # только data
node.roles: [master]            # только master
node.roles: [ingest]
node.roles: []                  # только coordinating
```

Для больших cluster — разделять роли.

---

## 8. Segments

Внутри shard — набор Lucene segments (immutable файлы).

Индексация:
1. Документ в **in-memory buffer**.
2. Каждые `refresh_interval` (default 1 сек) → segment создаётся + записывается на диск → доступен для поиска.
3. Периодически segments **merge**ятся (маленькие → большие).

**Refresh vs flush**:
- **Refresh** (1 сек) — новый segment, стал searchable. Не пишет на диск (только memory + translog).
- **Flush** — actually на диск + fsync + очистка translog.

**Translog** — WAL-like. Пишется на диск синхронно, для durability между flush.

**Near real-time (NRT)**: между index и searchable = ~1 сек (refresh interval).

Для bulk-load — увеличить `refresh_interval=30s` → быстрее индексация.

---

## 9. Query DSL

REST API + JSON queries.

### 9.1 Match query (full-text)

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

Analyzer разбирает "быстрая доставка" → токены [быстр, доставк] → ищет документы где встречается любой.

### 9.2 Term query (exact)

```json
{
  "query": {
    "term": {
      "customer.keyword": "berik"
    }
  }
}
```

Точное совпадение. Не анализируется.

### 9.3 Bool query

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

- `must` — обязательно + влияет на score.
- `filter` — обязательно, не влияет на score (кэшируется).
- `should` — по возможности (boost score).
- `must_not` — исключить.

### 9.4 Range

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

### 9.5 Aggregations

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

Аналог SQL GROUP BY + aggregates.

### 9.6 Scoring (BM25)

По умолчанию — **BM25**. Учитывает:
- **TF (Term Frequency)** — сколько раз токен в документе.
- **IDF (Inverse Document Frequency)** — редкий токен = вес выше.
- **Length norm** — короткий документ с match = вес выше.

Custom scoring: `function_score`, `script_score`.

---

## 10. Индексация

### 10.1 Один документ

```
POST /orders/_doc
{
  "id": 1,
  "customer": "berik",
  "amount": 100.50,
  "created_at": "2026-09-05T10:00:00Z"
}
```

### 10.2 Bulk

```
POST /_bulk
{ "index": { "_index": "orders", "_id": "1" } }
{ "customer": "berik", "amount": 100.50 }
{ "index": { "_index": "orders", "_id": "2" } }
{ "customer": "alice", "amount": 200.00 }
```

Тысячи документов одним запросом. Обязательно для performance.

### 10.3 Refresh policy

- **`wait_for`** — ждать refresh (для read-after-write consistency).
- **`true`** — immediate refresh (медленно, только для тестов).
- **`false`** — не форсировать (default, refresh_interval).

---

## 11. ILM (Index Lifecycle Management)

Автоматически перемещает индексы между фазами:
- **Hot** — новые, active, на быстрых SSD.
- **Warm** — старее, read-mostly, на медленных дисках.
- **Cold** — старые, редко читаемые.
- **Delete** — удалить.

Классика для логов:
- Индексы **daily** (`logs-2026.09.05`).
- Hot: 7 дней.
- Warm: 30 дней.
- Delete: 90 дней.

---

## 12. Java клиент

### 12.1 REST Client (basic, обычный)

```java
RestClient client = RestClient.builder(new HttpHost("es", 9200)).build();
Request req = new Request("GET", "/orders/_search");
req.setJsonEntity("{...}");
Response resp = client.performRequest(req);
```

### 12.2 Elasticsearch Java Client (новый, 8+)

```java
ElasticsearchClient client = new ElasticsearchClient(transport);

SearchResponse<Order> resp = client.search(s -> s
    .index("orders")
    .query(q -> q.match(m -> m.field("description").query("заказ"))),
    Order.class);
```

Type-safe.

**RestHighLevelClient** (устарел в 8+) заменён на новый.

### 12.3 Spring Data Elasticsearch

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

Аналог Spring Data JPA но для ES.

---

## 13. Kibana

UI для ES:
- **Discover** — поиск и фильтрация логов.
- **Visualizations** — графики.
- **Dashboards** — панели.
- **Dev Tools** — REST console для запросов.
- **Alerts**.
- **ML** — anomaly detection.

Аналог Grafana, но специализирован под ES.

---

## 14. ELK stack

**ELK** = Elasticsearch + Logstash + Kibana. Стандарт для логов.

С добавлением Beats — **Elastic Stack**.

Flow (типовой):
```
App → Filebeat (реад лог-файлов)
        │
        ▼
   [Logstash pipeline]  ← парсинг, обогащение
        │
        ▼
   [Elasticsearch]
        │
        ▼
     [Kibana]  ← ты смотришь
```

Или проще: **Filebeat → Elasticsearch → Kibana** (без Logstash для простых случаев).

### 14.1 В ИСНА

Из memory `knp-prod-historical-logs-elk`: свой ELK-кластер (`isna-elk`), логи по-старому через kubectl exec в под ES.

`kubectl logs` = только сегодня (ротация); за вчера идти в ES.

Логи в поле `log`, `match_phrase` по дефисным кодам плохо работает → искать по токенам.

---

## 15. Проблемы и типовые кейсы

### 15.1 Yellow cluster

- Single-node ES + replicas=1 → всегда yellow.
- Node down → shards без replicas → yellow.

Fix: `number_of_replicas=0` для single-node или добавить node.

### 15.2 Red cluster

Некоторые primary shards недоступны. Data loss риск.

Fix: восстановить nodes, snapshot restore.

### 15.3 Медленный поиск

Возможные причины:
- Слишком большой shard.
- Слишком много shards (overhead).
- Нет фильтров → полный scan.
- Слабое железо.
- Sorted query по text field (плохо, только по keyword).

Профайлинг: `?pretty&profile=true`.

### 15.4 Cardinality на aggregations

Aggregation по high-cardinality field (userId) = много buckets → OOM.

Fix: `terms.size` ограничение, `composite` aggregation для пагинации.

### 15.5 Mapping конфликт

Индексируешь string в поле, потом same field как number → conflict, документ отклонён.

Правило: **explicit mapping** для важных индексов, не полагаться на dynamic mapping.

### 15.6 Slow bulk

- Уменьшить refresh_interval (`30s` или `-1`).
- Больше batch size.
- Не через один coordinating node — round-robin.

---

## 16. Best practices

1. **Explicit mapping** для важных индексов.
2. **`text` для search, `keyword` для filter/sort/aggregate**.
3. **Shard size 10-50 GB**.
4. **`number_of_shards`** = продумать заранее.
5. **1 replica** default.
6. **Bulk** для indexing.
7. **ILM** для logs.
8. **Snapshots** к S3/blob для backup.
9. **3 или 5 master-eligible** nodes.
10. **Monitoring** (JVM, indexing rate, search latency).

---

## 17. Собесные вопросы

1. **Что такое Elasticsearch?** — Distributed search + analytics engine, поверх Lucene, JSON documents.
2. **Что такое inverted index?** — Слово → список документов где встречается; основа full-text search.
3. **Разница text и keyword?** — text анализируется (для search); keyword точный (для filter/sort/aggregate).
4. **Что такое shard и replica?** — Shard — часть индекса (для масштаба); replica — копия (для HA).
5. **Как ES решает в какой shard записать?** — Обычно `hash(_id) % number_of_shards`.
6. **Green / Yellow / Red — что?** — Green: всё ок; Yellow: replicas не назначены; Red: primary недоступны (data loss).
7. **Роли nodes?** — Master-eligible, data, coordinating, ingest.
8. **Почему 3 master-eligible?** — Quorum (2 из 3), избежание split-brain.
9. **Что такое segment?** — Immutable Lucene файл; много segment'ов в shard; merge периодически.
10. **Refresh vs flush?** — Refresh (1 сек) — searchable, ещё в memory + translog; flush — на диск.
11. **Что такое analyzer?** — Пайплайн: char filter + tokenizer + token filters.
12. **Разница must и filter в bool?** — must влияет на score; filter не влияет, кэшируется.
13. **Что такое BM25?** — Алгоритм scoring в ES (TF + IDF + norm).
14. **ILM — зачем?** — Автоматически перемещать индексы hot → warm → cold → delete.
15. **Что такое ELK stack?** — Elasticsearch + Logstash + Kibana; для логов и analytics.

---

## Итог

- **ES** = distributed search + analytics.
- **Inverted index** — сердце full-text.
- **Shard** для масштаба, **replica** для HA.
- **text vs keyword** — критично для правильного mapping.
- **Bulk + ILM** для logs.
- **Kibana** для просмотра.
- **ELK** — стандарт логов; в ИСНА `isna-elk`.

Следующий — `46-nginx.md`.
