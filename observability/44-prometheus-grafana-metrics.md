# 44. Prometheus, Grafana, метрики: observability для production

## Зачем нужны метрики отдельно от логов

Разработчик который знаком только с logging обычно думает — «логи содержат всё, зачем ещё metrics?». Тактика — при инциденте искать в логах, найти root cause, fix. Работает для development и небольших систем. Но при production scale — миллионы log entries per hour — эта стратегия breaks. Как понять что error rate растёт до того как customer complaints coming? Как определить среднюю latency без processing всех logs? Как настроить alerts на «too many errors» когда «too many» это relative к normal rate?

Metrics решают задачи которые logs решить не могут efficiently. Aggregation по time — «сколько запросов в minute». Statistical analysis — percentiles latency, error rates, saturation. Alerting на trends — «error rate exceeded 5% for 5 minutes». Long-term retention efficient — metrics 100x cheaper than logs per unit of information. Real-time dashboards showing current state без searching logs.

Разница между разработчиком «использующим logs» и «понимающим observability» проявляется в incident response. Первый greps logs after alert — «why did this fire?». Второй знает что alert based на p99 latency, correlates с metric «database_connection_pool_saturation», sees что pool exhausted 5 minutes before, checks что long-running transaction started at that time, identifies specific transaction in APM traces. Metrics answered «what», tracing answered «where», logs answered «what specifically». Complete observability requires all three.

В этом файле разберём observability comprehensively. Fundamental types metrics — counter, gauge, histogram, summary — каждый с specific use cases. Prometheus как pull-based TSDB — architecture, scrape configs, query language PromQL. Grafana visualization. Alertmanager routing plus deduplication. Micrometer как Java facade abstraction. RED plus USE methodologies. Percentiles vs averages why matters. Cardinality — fundamental Prometheus limit. Practical setup в Spring Boot.

## Четыре типа метрик и их semantics

Counter это монотонно растущий number. Never decreases (кроме reset при process restart). Represents count of events over time.

Examples. http_requests_total — total HTTP requests since startup. orders_created_total — total orders processed. exceptions_total — total exceptions occurred.

Usage не value itself но rate change. rate(counter[5m]) computes derivative — events per second averaged over 5 minutes. Meaningful metric — actual traffic rate.

Prometheus predicts value at query time если counter reset detected (process restart). Algorithm assumes 0 starting point, computes rate accordingly. Reset detection through decrease detection.

Gauge это snapshot value at moment. Can go up or down freely. Represents state.

Examples. jvm_memory_used_bytes — current memory usage. active_connections — currently open connections. queue_size — current queue depth. temperature — current temperature reading.

Usage — direct reading value. Alerts на thresholds — «memory > 80% для 5 minutes». Trends over time show growth patterns.

Histogram это distribution values по buckets. Counts observations в each bucket.

Buckets определяются заранее. Default для HTTP latency в Micrometer — [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]. Каждый bucket counts observations <= X seconds.

Example. http_request_duration_seconds histogram. If request took 0.03 seconds — increments buckets 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10. Cumulative — each bucket contains count of observations <= its upper limit.

Из histogram можно вычислить percentiles через histogram_quantile function в PromQL. p95 latency shows «95% requests faster than X». Meaningful business metric.

Summary похож на histogram но percentiles вычисляются на клиенте не в Prometheus. Client-side computation more accurate (exact percentiles vs bucket-based approximation) but not aggregatable.

Key difference. Histogram — buckets aggregatable — можно compute p95 across all instances of service. Summary — client-side quantiles — cannot be aggregated (percentile of average != average of percentiles).

Правило — обычно histogram лучше потому что агрегируется. Slight loss precision worth aggregatability для service running на multiple instances.

## Prometheus architecture

Prometheus это open-source monitoring system plus time-series database.

Разработан в SoundCloud circa 2012. Now в CNCF (Cloud Native Computing Foundation) alongside Kubernetes. Industry standard для cloud-native monitoring.

Ключевые characteristics. Pull-based — Prometheus сам приходит и забирает метрики с targets. Different from many older systems (StatsD, Graphite) that push metrics. TSDB (Time Series Database) optimized для metrics data patterns. PromQL — expressive query language.

Architecture:
```
┌─── App 1 ────┐    ┌─── App 2 ────┐    ┌─── Node exporter ──┐
│ /metrics    │    │ /metrics     │    │ /metrics           │
└──────┬──────┘    └──────┬───────┘    └──────┬─────────────┘
       │                  │                    │
       │ scrape (каждые 15s)                  │
       │                  │                    │
       └──────────────┬───┴────────────────────┘
                      │
                      ▼
              ┌───────────────┐
              │ Prometheus    │
              │  TSDB         │
              │  Scraper      │
              │  PromQL       │
              └───┬───────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
   ┌──────────┐        ┌──────────────┐
   │  Grafana │        │ Alertmanager │
   │ (UI)     │        │  (алерты)    │
   └──────────┘        └──────────────┘
```

Applications expose /metrics endpoint returning current values. Prometheus periodically scrapes (обычно 15s interval). Stores time series в TSDB. Queries через PromQL. Sends alerts к Alertmanager. Visualizes через Grafana.

Pull vs push trade-offs. Pull advantages — target agnostic (не нужно know monitoring existence), Prometheus knows what's down (missing scrapes = alert candidate), debugging easier (curl /metrics manually). Pull disadvantages — problematic для short-lived jobs (may finish before scrape), Push Gateway workaround exists.

Push advantages — easier для ephemeral jobs, no scrape delay, works через firewalls (outbound only). Push disadvantages — coupling между application и monitoring, difficult to detect missing metrics, security concerns (auth push endpoint).

## Формат метрик

Text-based, simple:
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234
http_requests_total{method="POST",status="200"} 567
http_requests_total{method="GET",status="500"} 12

# HELP jvm_memory_used_bytes JVM memory used
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="heap"} 268435456
jvm_memory_used_bytes{area="nonheap"} 67108864
```

Каждая строка — метрика name plus labels plus value. Labels in curly braces provide dimensions.

Labels это ключевая feature Prometheus. Метрики plus labels = множество measurement. Same metric can have many combinations of label values, each becoming separate time series.

```
http_requests_total{method="GET", status="200", path="/api/orders"} 1234
http_requests_total{method="POST", status="500", path="/api/orders"} 5
```

Одна метрика http_requests_total, множество измерений. Filter или aggregate по labels в queries.

Critical caveat — cardinality. Много labels × много values = взрыв memory usage. Пример disaster — label user_id с миллионом values → миллион отдельных time series → Prometheus OOM.

Правило. Labels с ограниченным набором values — status codes (< 20), HTTP methods (< 10), path templates (< 100). Не user_id, request_id, timestamp — unbounded values.

## Scrape configuration

Prometheus knows what to scrape через configuration:
```yaml
scrape_configs:
  - job_name: 'isna-knp'
    scrape_interval: 15s
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ['isna-knp:8080']

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: 'true'
```

Job — logical grouping targets. scrape_interval — периодичность. metrics_path — endpoint для scraping (default /metrics).

Static configs списывают fixed target list. Simple но не dynamic — не handles auto-scaling groups.

Service discovery configurations для dynamic environments. Kubernetes SD auto-discovers pods matching labels. Consul SD queries Consul registry. EC2 SD queries AWS. Каждый provider has specific discovery config.

Relabel configs manipulate metadata before scraping. Filter targets (keep only pods with specific annotation). Transform labels (rename __meta_kubernetes_pod_label_app to app). Powerful mechanism customizing behavior.

Exporters для systems без native Prometheus support. node-exporter — Linux metrics (CPU, memory, disk). jmx-exporter — JMX beans через HTTP. postgres-exporter — PostgreSQL statistics. rabbitmq-exporter — RabbitMQ metrics. redis-exporter — Redis stats. Deployed as sidecars, DaemonSets, standalone processes.

## PromQL основы

Язык queries. Строится вокруг manipulation time series.

Selector picks time series:
```promql
http_requests_total{method="GET"}
```

Returns all time series matching metric name plus label constraints. Может быть multiple series одновременно если matched by multiple label combinations.

Rate для counters:
```promql
rate(http_requests_total[5m])
```

Скорость роста counter за 5 minutes averaged. Result в units per second. Standard way to interpret counter data.

Aggregation combines multiple series:
```promql
sum(rate(http_requests_total[5m]))                    # общая rate
sum by (method) (rate(http_requests_total[5m]))       # по methods
sum by (status) (rate(http_requests_total{path="/api/orders"}[5m]))
```

Aggregation operators — sum, avg, max, min, count, stddev, stdvar. by clause groups results by specific labels. without clause groups by everything except specified.

Histogram percentile:
```promql
histogram_quantile(0.95,
    sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

p95 latency за 5 minutes. le (less-equal) special label indicating bucket upper bound.

Arithmetic operations:
```promql
# ratio ошибок
sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))
```

Compute error rate as percentage of total requests. Regex status=~"5.." matches all 5xx errors.

Alert conditions same syntax:
```promql
# error rate > 5%
(sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))) > 0.05
```

Если результат = true и above threshold — alert triggered.

## Alertmanager

Component для alerts management. Prometheus проверяет rules (conditions) — если matches — firing alert — отправляет Alertmanager.

Alertmanager responsibilities. Grouping — объединяет похожие alerts by labels (например все alerts от same service into one notification). Deduplication — не спамить дубликатами. Silencing — заглушить на time period (during maintenance). Routing — куда отправить: Slack, PagerDuty, email, custom webhooks.

Configuration:
```yaml
receivers:
  - name: 'slack-critical'
    slack_configs:
      - api_url: 'https://hooks.slack.com/...'
        channel: '#alerts'

route:
  receiver: 'slack-critical'
  group_by: ['alertname', 'service']
  group_wait: 30s
  repeat_interval: 4h
```

route matches alerts к receivers based на labels. group_by controls grouping. group_wait wait period перед first notification (allow related alerts to arrive). repeat_interval how often to remind about unresolved alerts.

Nested routes для different alert types к different receivers. Critical alerts к PagerDuty (paging). Warnings к Slack (informational). Info-level к email digest.

Silencing через web UI или API. Temporarily suppress alerts matching labels. Common when doing maintenance operations knowing alerts will fire but no action needed.

## Grafana

UI для visualization метрик. Не только Prometheus — supports InfluxDB, Elasticsearch, CloudWatch, множество datasources.

Dashboards это collections panels showing different aspects system. Common panels для Spring Boot service. HTTP request rate по endpoint's — see which are hot. Latency p50/p95/p99 — user experience. Error rate — reliability signal. JVM memory plus GC — resource health. DB connections — infrastructure saturation.

Panel types. Graph/Time series — line chart над time. Stat/Single number — current value как big number. Gauge — «speedometer» style visualization. Bar chart/Pie — categorical breakdown. Table — tabular data. Heatmap — histogram over time (great для latency distributions). Alert list — currently firing alerts.

Alerts в Grafana. Можно определить alerts прямо в panel — trigger когда metric matches condition. Отправлять через notification channels. Многие prefer Grafana Alerting UI vs Alertmanager для simplicity.

Готовые dashboards на grafana.com/dashboards. Тысячи community dashboards для common systems — Kafka, PostgreSQL, JVM, Kubernetes, RabbitMQ. Import через ID — instant working dashboard. Customize для specific setup.

## Micrometer в Spring Boot

Micrometer это Java-фасад для метрик — как SLF4J для логов. Application code writes через Micrometer API. Backend adapter (Prometheus, StatsD, CloudWatch, etc) selected через dependency:
```gradle
implementation 'io.micrometer:micrometer-registry-prometheus'
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

Spring Boot автоматически. Registers standard metrics — JVM (memory, GC, threads, classes), HTTP (server requests, client requests), HikariCP (pool), JPA (queries), Kafka (producer/consumer stats). Exposes через /actuator/prometheus в Prometheus format.

Auto-instrumented metrics. jvm_* — memory areas, GC pauses, thread counts, loaded classes. http_server_requests_seconds — histogram latency, count. hikaricp_* — pool active/idle/pending. jdbc_* — connection acquire times. hibernate_* — если enabled — query stats. spring_kafka_* — Kafka client metrics. process_* — CPU time, uptime, file descriptors. system_* — OS-level metrics.

Custom metrics через MeterRegistry:
```java
@Service
class OrderService {
    private final Counter ordersCreated;
    private final Timer orderProcessingTime;
    private final Gauge activeOrders;

    public OrderService(MeterRegistry registry, OrderRepository repo) {
        this.ordersCreated = registry.counter("orders.created",
            Tags.of("service", "isna-knp"));

        this.orderProcessingTime = registry.timer("orders.processing.time");

        this.activeOrders = Gauge.builder("orders.active", repo, 
            r -> r.countByStatus(ACTIVE))
            .register(registry);
    }

    public void create(OrderRequest req) {
        Timer.Sample sample = Timer.start();
        try {
            Order o = new Order(req);
            repo.save(o);
            ordersCreated.increment();
        } finally {
            sample.stop(orderProcessingTime);
        }
    }
}
```

Counter increment на success. Timer sample plus stop для duration measurement. Gauge через supplier function evaluated on-demand.

@Timed annotation autoматически создаёт timer:
```java
@Timed(value = "orders.create", description = "Time to create order")
public Order create(OrderRequest req) { ... }
```

Requires @Bean TimedAspect для AOP interception. Cleaner чем manual instrumentation для simple cases.

Actuator configuration:
```yaml
management:
  endpoints:
    web.exposure.include: health,info,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
      env: ${SPRING_PROFILES_ACTIVE:default}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
```

Global tags added к every metric. application plus env identify source service. distribution config controls histogram plus percentile emission.

## RED и USE методики

RED (Rate, Errors, Duration) — методология для services. Introduced Tom Wilkie. Три метрики per service.

R — Rate — запросы/сек. How much traffic getting.

E — Errors — ошибки/сек или percentage. What fraction failing.

D — Duration — latency (p50, p95, p99). How long таким.

Classical service golden signals. If R suddenly changes — traffic pattern shift. If E rises — reliability issue. If D increases — performance degradation.

USE (Utilization, Saturation, Errors) — методология для resources. Introduced Brendan Gregg. Три метрики per resource.

U — Utilization — % использования. How busy.

S — Saturation — queue length or wait time. Backlog forming.

E — Errors — resource-level errors.

Пример для CPU. Utilization = 80%. Saturation = run queue > 1. Errors = не бывает для CPU обычно.

Пример для disk. Utilization = 60%. Saturation = IO wait queue длина. Errors = read/write failures.

Golden signals от Google SRE Book — combining ideas. Traffic (like Rate). Errors. Latency (like Duration). Saturation. Similar concept — monitor symptoms user sees not just causes.

Common thread всех three approaches — monitor symptoms что видит пользователь, не только causes (CPU high). CPU high может быть normal for compute-heavy service. But user-facing latency increase always indicative.

## Percentiles vs averages

Average (avg) — среднее по values. Common but misleading.

Проблема — average LIES about user experience. Если 99 запросов по 10ms и 1 по 1000ms — average = 20ms. Sounds fine. But пользователь который поймал 1000ms request — не считает сервис быстрым. Actually 1% users experience terrible latency, average hides это.

Percentiles правильный подход. p50 (median) = типичный пользователь. p95 = 95% пользователей faster. p99 = 99% faster (1% страдает). p99.9 = 99.9% faster (0.1% страдает — fat tail).

Правило — alert по p99 не по avg. Avg passes threshold когда обычно fine but occasional bad experience. p99 catches when significant fraction actually suffering.

Distribution shapes matter. If distribution normal — avg approximates. If skewed — avg misses tail. Real-world traffic almost always skewed — occasional slow queries, GC pauses, transient issues create outliers. Percentiles honest measurement.

Micrometer configuration для percentile emission:
```yaml
management.metrics.distribution:
  percentiles-histogram:
    http.server.requests: true
  percentiles:
    http.server.requests: 0.5, 0.95, 0.99, 0.999
```

Emits histogram buckets plus computed percentiles both. Buckets enable server-side percentile computation aggregating across instances. Client-side percentiles included для quick reference но not aggregatable.

## Cardinality как критический constraint

Каждая уникальная комбинация labels = отдельный time series. Prometheus memory usage scales с number of active time series.

Простой example. Metric http_requests_total с label path. 100 endpoints — 100 time series. Ok.

Плохой example. Same metric с label path включающим ID:
```
http_requests_total{path="/api/orders/1"} 1
http_requests_total{path="/api/orders/2"} 1
...
http_requests_total{path="/api/orders/1000000"} 1
```

1 миллион time series. Prometheus OOM. Query performance dramatically degraded.

Правило critical — labels с ограниченным множеством values. Path templates like «/api/orders/{id}» вместо concrete IDs. HTTP status codes. Method verbs. Instance identifiers if bounded.

Never labels с unbounded values. user_id (unless bounded users). request_id (each unique). timestamp (each different). Anywhere с identifier-like semantic.

Spring Boot Micrometer автоматически uses URI template как label — /api/orders/{id} not /api/orders/42. Careful when custom instrumentation — check не логируешь unbounded IDs.

Investigation cardinality issues через Prometheus itself:
```promql
count({__name__=~".+"}) by (__name__)
```

Shows count time series per metric name. Anomalously high counts indicate cardinality problem.

## Setup в КНП

Обычная схема наблюдения в enterprise:
- Spring Boot exposes /actuator/prometheus.
- Prometheus scrape'ит.
- Grafana dashboards.
- Alertmanager → Slack / email.

Метрики что смотреть. HTTP: rate, error rate, latency (RED). JVM: heap, GC pauses. HikariCP: pool active/idle/pending. Rabbit: queue depth, consumer count. Kafka: consumer lag. Consul: healthy services count. PostgreSQL: connections, replication lag, slow queries.

Alerts критичные для production awareness. Consumer lag > threshold. UnderReplicatedPartitions > 0 (Kafka). Broker/database down. Disk full approaching. High p99 latency indicating performance issues. Elevated error rate.

Alert calibration important. Too sensitive — alert fatigue, real issues ignored. Too permissive — real issues not surfaced timely. Balance based на operational experience.

## Итоги

Метрики решают задачи которые logs не могут efficiently. Aggregation, statistical analysis, alerting on trends, dashboards.

4 типа метрик. Counter monotonically increasing — use rate. Gauge snapshot value. Histogram distribution через buckets — use histogram_quantile для percentiles. Summary client-side percentiles — не aggregatable.

Prometheus pull-based TSDB. Scrapes /metrics endpoints periodically. Stores time series. PromQL для queries.

Labels dimensions для metrics. Filter, aggregate. Cardinality critical constraint — bounded label values only.

Alertmanager routing, grouping, deduplication, silencing alerts. Prometheus fires — Alertmanager processes plus notifies.

Grafana UI для visualizations. Multiple datasources. Community dashboards available. Alerts могут быть defined в Grafana OR Alertmanager.

Micrometer Java facade для metrics. Spring Boot автоматически instruments common concerns. Custom metrics через Counter, Timer, Gauge. @Timed annotation для declarative timing.

RED (Rate, Errors, Duration) для services. USE (Utilization, Saturation, Errors) для resources. Golden signals из Google SRE. Monitor symptoms user sees.

Percentiles not averages. Avg lies about tail experience. p95/p99 honest metrics.

Cardinality critical constraint. Bounded label values only. Path templates not IDs. Investigation через Prometheus queries showing series counts per metric.

Дальше — Elasticsearch как основа для search, аналитика, ELK stack для логов.
