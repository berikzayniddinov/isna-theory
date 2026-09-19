# 44. Prometheus + Grafana + метрики

Что такое метрики, как собирать и визуализировать.

---

## 1. Зачем метрики

**Метрики** — числовые показатели, снимаемые периодически. В отличие от логов (текстовые события).

Зачем:
- Мониторинг: «сколько запросов в секунду», «сколько ошибок», «какая latency».
- **Alerting**: «если error rate > 5% — разбудить SRE».
- Capacity planning.
- Отладка performance.
- SLA reporting.

**Observability** = логи + метрики + traces. Метрики — самая дешёвая часть.

---

## 2. Типы метрик

### 2.1 Counter (счётчик)

**Монотонно растёт** (или reset при рестарте). Никогда не уменьшается.

Примеры:
- `http_requests_total` — всего запросов.
- `orders_created_total` — всего созданных ордеров.
- `exceptions_total` — всего exceptions.

Использование:
- Считать rate: **скорость** = derivative.
- В PromQL: `rate(http_requests_total[5m])` = запросов/сек за последние 5 мин.

### 2.2 Gauge (индикатор)

**Может расти и падать**. Snapshot текущего значения.

Примеры:
- `jvm_memory_used_bytes` — сколько занято.
- `active_connections` — сейчас открытых.
- `queue_size` — размер очереди.
- `temperature` — температура.

Использование:
- Прямо смотреть значение.
- Мониторить пороги.

### 2.3 Histogram

**Распределение** значений по buckets. Считает сколько наблюдений попало в каждый bucket.

Примеры:
- `http_request_duration_seconds` — latency распределение.
- `payload_size_bytes` — размер запросов.

Bucket'ы (по умолчанию для http): [0.005, 0.01, 0.025, ..., 10]. Каждый bucket считает `<= X`.

Из histogram можно вычислить percentiles (p50, p95, p99) через **`histogram_quantile`** в PromQL.

### 2.4 Summary

Похож на histogram, но percentiles считаются **на клиенте** (не в Prometheus).

Плюс: точнее.
Минус: нельзя агрегировать по инстансам (percentile от average != average от percentile).

**Правило**: обычно **histogram лучше**, потому что агрегируется.

---

## 3. Prometheus

### 3.1 Что это

**Prometheus** — open-source monitoring system + time-series database.

- Разработан в SoundCloud, теперь в CNCF (как K8s).
- **Pull-based** — Prometheus сам приходит и забирает метрики с targets.
- **TSDB** (Time Series Database) — оптимизировано под метрики.
- **PromQL** — язык запросов.

### 3.2 Архитектура

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

### 3.3 Pull vs push

- **Pull (Prometheus)** — сам приходит, spider'ит `/metrics` endpoints.
- **Push (StatsD, Graphite)** — приложения сами шлют метрики.

Плюсы pull:
- Приложение не знает про мониторинг (уменьшенная связность).
- Prometheus знает какие targets down (нет scrape → alert).
- Легче debug — можно посмотреть `/metrics` руками.

Минус: сложнее для короткоживущих jobs (Push Gateway для них).

### 3.4 Формат метрик

Text-based, простой:
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

Каждая строка — метрика + labels + значение.

### 3.5 Labels

**Ключевая фича**. Метрики + labels = множество measurement.

```
http_requests_total{method="GET", status="200", path="/api/orders"} 1234
http_requests_total{method="POST", status="500", path="/api/orders"} 5
```

Одна метрика, много измерений — фильтруй / агрегируй по labels.

**Кавет: cardinality**. Много labels × много значений = взрыв. Пример: label `user_id` с миллионом значений → миллион отдельных time series → OOM в Prometheus.

**Правило**: labels с **ограниченным** набором значений (status, method, path). Не user_id, request_id, timestamp.

### 3.6 Scrape config

```yaml
# prometheus.yml
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

Prometheus сам находит поды с annotation `prometheus.io/scrape: "true"` и скрейпит.

### 3.7 Экспортеры

Готовые для стандартных систем:
- **node-exporter** — метрики Linux (CPU, memory, disk).
- **jmx-exporter** — JMX → Prometheus (для Kafka, старой Java).
- **postgres-exporter** — PG статистика.
- **rabbitmq-exporter**.
- **redis-exporter**.

Deploy как sidecar / DaemonSet.

---

## 4. PromQL

Язык запросов. Основы.

### 4.1 Selector

```promql
http_requests_total{method="GET"}
```

Возвращает все time series с указанной метрикой + labels.

### 4.2 Rate (счётчик)

```promql
rate(http_requests_total[5m])
```

Скорость роста counter за 5 минут (запросов в секунду).

### 4.3 Aggregation

```promql
sum(rate(http_requests_total[5m]))                    # общая rate
sum by (method) (rate(http_requests_total[5m]))       # по методам
sum by (status) (rate(http_requests_total{path="/api/orders"}[5m]))
```

Другие операторы: `avg`, `max`, `min`, `count`, `stddev`.

### 4.4 Histogram percentile

```promql
histogram_quantile(0.95,
    sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

p95 latency за последние 5 минут.

`le` (less-equal) — специальный label bucket'а.

### 4.5 Arithmetic

```promql
# ratio ошибок
sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))
```

### 4.6 Alert conditions

```promql
# error rate > 5%
(sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))) > 0.05
```

Если результат = true и > threshold → alert.

---

## 5. Alertmanager

Компонент для алертов.

Prometheus проверяет rules (условия) → если matches → **firing alert** → отправляет Alertmanager.

Alertmanager:
- **Grouping** — объединяет похожие (по labels).
- **Deduplication** — не спамить одним.
- **Silencing** — заглушить на время.
- **Routing** — куда: Slack, PagerDuty, email.

Пример:
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

---

## 6. Grafana

**Grafana** — UI для визуализации метрик (не только Prometheus — поддерживает InfluxDB, Elasticsearch, CloudWatch).

### 6.1 Dashboards

Панели с графиками. Пример:
- HTTP request rate (по endpoint'ам).
- Latency p50/p95/p99.
- Error rate.
- JVM memory / GC.
- DB connections.

### 6.2 Panels

Типы:
- **Graph / Time series** — линейный график.
- **Stat / Single number** — одно значение.
- **Gauge** — «спидометр».
- **Bar chart / Pie**.
- **Table**.
- **Heatmap** — histogram по времени.
- **Alert list**.

### 6.3 Alerts в Grafana

Можно определить alerts прямо в панели. Отправлять через notification channels (Slack, email, PagerDuty).

Многие используют Grafana Alerting вместо Alertmanager (проще UI).

### 6.4 Готовые dashboards

**grafana.com/dashboards** — тысячи готовых. Для Kafka, PG, JVM, K8s.

Импорт по ID.

---

## 7. Micrometer в Spring Boot

**Micrometer** — Java-фасад для метрик (как SLF4J для логов).

```gradle
implementation 'io.micrometer:micrometer-registry-prometheus'
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

Spring Boot автоматически:
- Регистрирует стандартные метрики (JVM, HTTP, HikariCP, JPA, Kafka).
- Экспортирует через `/actuator/prometheus`.

### 7.1 Автометрики

Из коробки:
- **`jvm_*`** — memory, GC, threads, classes.
- **`http_server_requests_seconds`** — все HTTP-запросы.
- **`hikaricp_*`** — pool.
- **`jdbc_*`**.
- **`hibernate_*`** (если enabled).
- **`spring_kafka_*`**.
- **`process_*`** — CPU, uptime.
- **`system_*`** — OS.

### 7.2 Custom метрики

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

        this.activeOrders = Gauge.builder("orders.active", repo, r -> r.countByStatus(ACTIVE))
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

### 7.3 @Timed

```java
@Timed(value = "orders.create", description = "Time to create order")
public Order create(OrderRequest req) { ... }
```

Аннотация автоматически создаёт timer.

Требует `@Bean TimedAspect`.

### 7.4 Actuator config

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

---

## 8. RED и USE методики

Что мониторить.

### 8.1 RED (для сервисов)

- **R**ate — запросы/сек.
- **E**rrors — ошибки/сек (или %).
- **D**uration — latency (p50, p95, p99).

Классический service golden signal.

### 8.2 USE (для ресурсов)

- **U**tilization — % использования (CPU, memory).
- **S**aturation — очередь / ожидание.
- **E**rrors — ошибки.

Пример для CPU:
- Utilization = 80%.
- Saturation = run queue > 1.
- Errors = не бывает у CPU.

### 8.3 Golden signals (Google SRE)

- Traffic.
- Errors.
- Latency.
- Saturation.

Все три подхода похожи. Идея: мониторить **симптомы** (что видит пользователь), не только причины (CPU).

---

## 9. Percentiles vs Averages

**Average (avg)** — среднее.

Проблема: average ЛЖЁТ. Если 99 запросов по 10ms и 1 по 1000ms — average = 20ms. Пользователь который поймал 1000ms — не считает что «сервис быстрый».

**Percentiles** — правильный подход:
- p50 (median) = типичный пользователь.
- p95 = 95% пользователей быстрее.
- p99 = 99% пользователей быстрее (1% страдает).
- p99.9 = fat tail.

**Правило**: alert по p99, не по avg.

---

## 10. Cardinality — важный кавет

Каждая уникальная комбинация labels = отдельный time series.

```
http_requests_total{path="/api/orders/1"} 1
http_requests_total{path="/api/orders/2"} 1
...
http_requests_total{path="/api/orders/1000000"} 1
```

= 1 миллион time series → Prometheus OOM.

**Правило**: не путать пути с параметрами. Использовать template `/api/orders/{id}` как label.

Spring Boot Micrometer автоматически agrupирует по URI template.

---

## 11. ИСНА и мониторинг

Обычная схема:
- Spring Boot exposes `/actuator/prometheus`.
- Prometheus scrape'ит.
- Grafana dashboards.
- Alertmanager → Slack / email.

Метрики что смотреть:
- HTTP: rate, error rate, latency (RED).
- JVM: heap, GC pauses.
- HikariCP: pool active/idle/pending.
- Rabbit: queue depth, consumer count.
- Kafka: consumer lag.
- Consul: healthy services count.
- PostgreSQL: connections, replication lag, slow queries.

---

## 12. Собесные вопросы

1. **Типы метрик?** — Counter (растёт), Gauge (снапшот), Histogram (buckets), Summary (percentiles на клиенте).
2. **Разница histogram и summary?** — Histogram: buckets, percentiles в PromQL (агрегируется); Summary: percentiles на клиенте (точнее, не агрегируется).
3. **Pull vs push?** — Prometheus pull (сам приходит); StatsD push (приложение шлёт).
4. **Что такое PromQL?** — Query language: rate(), sum by, histogram_quantile.
5. **Как вычислить p95 из histogram?** — `histogram_quantile(0.95, sum by (le) (rate(bucket[5m])))`.
6. **Что такое cardinality?** — Уникальных combinations labels; высокая = OOM.
7. **Почему нельзя avg latency?** — Average скрывает выбросы; percentiles показывают реальный опыт пользователей.
8. **RED методика?** — Rate, Errors, Duration.
9. **USE методика?** — Utilization, Saturation, Errors — для ресурсов.
10. **Что такое Micrometer?** — Java-фасад для метрик (как SLF4J для логов).
11. **Как экспортировать метрики Spring Boot?** — Actuator + micrometer-registry-prometheus + `/actuator/prometheus`.
12. **Что такое Alertmanager?** — Компонент для routing/grouping/silencing алертов.
13. **Что такое Grafana?** — UI для визуализации метрик из разных источников.
14. **Что такое node-exporter?** — Prometheus exporter для метрик Linux (CPU, memory).
15. **Как алертить на error rate?** — `sum(rate(errors[5m])) / sum(rate(all[5m])) > 0.05`.

---

## Итог

- **Метрики** = числа во времени.
- **4 типа**: Counter / Gauge / Histogram / Summary.
- **Prometheus** = pull-based TSDB + PromQL.
- **Grafana** = визуализация.
- **Alertmanager** = алерты.
- **Micrometer** = Java-фасад в Spring Boot.
- **RED** для сервисов, **USE** для ресурсов.
- **Percentiles не average**.
- **Cardinality** — главный enemy Prometheus.

Следующий — `45-elasticsearch.md`.
