# 76. Логи в проде end-to-end: от `log.info()` до Kibana/Grafana

Что происходит с логом когда ты пишешь `log.info("Received request")` в Spring Boot приложении, крутящемся в k8s. Как он оказывается в Kibana/Grafana. Что такое correlation ID и как трейсить один запрос через 5 микросервисов.

---

## 1. Полный путь одной строки лога

```
log.info("...")
    ↓
SLF4J → Logback
    ↓
Console appender (stdout)
    ↓
Docker JSON log driver (файл на ноде)
    ↓
Fluent Bit / Promtail (DaemonSet собирает файлы)
    ↓
Kafka / Redis (буфер, опционально)
    ↓
Logstash / Loki (парсинг, обогащение)
    ↓
Elasticsearch / Loki storage
    ↓
Kibana / Grafana (UI поиска)
```

Каждый этап — отдельная тема, ниже разберём.

---

## 2. В контейнере: пишем в stdout, не в файл

**Правило контейнеризации**: приложение пишет только в stdout/stderr. Файлы на диске = не наша забота, контейнер эфемерный.

Boot-приложение по умолчанию пишет через Logback → `ConsoleAppender` → stdout. В `application.yml`:

```yaml
logging:
  pattern:
    console: '%d{yyyy-MM-dd''T''HH:mm:ss.SSSXXX} %-5level [%thread] %logger{36} - %msg%n'
```

Что видит container runtime:
```
2026-09-12T10:15:23.123+05:00 INFO  [http-nio-8080-exec-1] c.e.MyService - Received request
```

Kubernetes перехватывает stdout контейнера и пишет в файл на ноде:
```
/var/log/pods/<namespace>_<pod-name>_<uid>/<container>/<attempt>.log
```

Формат — **JSON per line** (kubelet добавляет метаданные):
```json
{"log":"2026-09-12T10:15:23... INFO Received request\n","stream":"stdout","time":"2026-09-12T10:15:23.123456789Z"}
```

Важно: kubelet ротирует эти файлы (`log-rotate` политика ноды). Если контейнер упал и pod удалён — логи с ноды теряются. **Поэтому нужен log shipper.**

---

## 3. Log shippers: Fluent Bit vs Promtail vs Filebeat

Log shipper — DaemonSet, запущенный на каждой ноде k8s. Смотрит в `/var/log/pods/` и `/var/log/containers/`, парсит новые строки, отправляет в backend.

**Fluent Bit** (написан на C, легковесный) — стандарт для EFK стека:
- Плагины input (tail file, systemd), filter (parser, kubernetes metadata), output (Elasticsearch, Kafka, Loki, S3).
- Формат конфига: `[SERVICE]`, `[INPUT]`, `[FILTER]`, `[OUTPUT]`.

Пример:
```conf
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            docker
    Tag               kube.*
    Refresh_Interval  5

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Merge_Log           On
    Keep_Log            Off
    K8S-Logging.Parser  On

[OUTPUT]
    Name  es
    Match *
    Host  elasticsearch.logging.svc.cluster.local
    Port  9200
    Index knp-logs
```

Что делает `kubernetes` filter:
- Читает pod-name из имени файла (`myapp-6b8f9-abc.log`).
- Идёт в kube-api, тащит labels/annotations/namespace пода.
- Обогащает лог: добавляет `kubernetes.pod_name`, `kubernetes.namespace_name`, `kubernetes.labels.app`.

Это КЛЮЧ — теперь в Kibana ты фильтруешь `kubernetes.labels.app: isnaknpsync AND level: ERROR`.

**Promtail** — от Grafana Labs, точно так же собирает логи но отправляет в **Loki** (не в ES). Конфиг похож.

**Filebeat** — часть Elastic Stack, более тяжёлый, богаче но и жирнее.

---

## 4. ELK (Elasticsearch + Logstash + Kibana) vs Loki

### 4.1 ELK

- **Elasticsearch** — inverted-index база. Индексирует каждое слово. Поиск full-text — быстрый (`error AND user_id:42`).
- **Logstash** — пайплайн обработки. Может парсить (grok), обогащать, роутить. Тяжёлый (JVM).
- **Kibana** — UI, дашборды, KQL для поиска.

Как хранит: каждый лог — document, все поля indexed. Мощно, но **дорого**: disk, RAM, CPU. Retention 7-30 дней в реальности.

Хорошо для: сложный поиск по контенту, alerting на pattern'ы, security-audit.

### 4.2 Loki

- **Loki** — от Grafana Labs. **Не индексирует контент** логов, только labels (маленький набор ключей).
- Хранит raw логи в chunks (gzip), labels — в отдельной БД (BoltDB/Cassandra).

Как выглядит запрос (LogQL):
```
{app="isnaknpsync", namespace="knp"} |= "ERROR" | json | user_id="42"
```

- Filter по labels — быстрый (index).
- Filter по контенту (`|= "ERROR"`) — линейный поиск в chunks за окно (медленнее ES для широких запросов, но дешевле).

Хорошо для: дешёвого хранения долгого retention, интеграция с Grafana + Prometheus + Tempo.

**Выбор**: если у команды уже ELK — оставайся. Если стартуешь чистый observability стек — Loki обычно дешевле в 5-10 раз по инфре, при тех же кейсах.

---

## 5. Structured logging: почему JSON, а не текст

Обычный текстовый лог:
```
2026-09-12 10:15:23 INFO Received request from user 42 for order 999
```

Плохо для поиска: чтобы найти запросы user_id=42, надо regex-парсить в Kibana. Медленно, ошибочно.

JSON layout:
```json
{
  "timestamp": "2026-09-12T10:15:23.123Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Received request",
  "user_id": 42,
  "order_id": 999,
  "trace_id": "a1b2c3...",
  "span_id": "d4e5f6..."
}
```

Каждое поле — индексируемо. `user_id: 42` — прямой lookup.

**В Boot настраивается через `logstash-logback-encoder`:**

`build.gradle`:
```gradle
implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
```

`logback-spring.xml`:
```xml
<configuration>
    <springProfile name="prod">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeMdcKeyName>trace_id</includeMdcKeyName>
                <includeMdcKeyName>span_id</includeMdcKeyName>
                <includeMdcKeyName>user_id</includeMdcKeyName>
                <customFields>{"app":"isna-knp","env":"prod"}</customFields>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="STDOUT"/>
        </root>
    </springProfile>
    <!-- локально — красивый текстовый -->
    <springProfile name="!prod">
        <include resource="org/springframework/boot/logging/logback/base.xml"/>
    </springProfile>
</configuration>
```

Теперь в проде stdout — JSON. Локально — читаемый текст.

---

## 6. MDC: Mapped Diagnostic Context

MDC — механизм SLF4J для добавления ключ-значений в **контекст текущего потока**. Каждая последующая строка лога в этом потоке автоматически получит эти поля.

```java
import org.slf4j.MDC;

MDC.put("user_id", "42");
MDC.put("order_id", "999");
log.info("Processing"); // → JSON будет содержать user_id=42, order_id=999
// ...
MDC.clear(); // ОБЯЗАТЕЛЬНО в конце
```

**Важно**: MDC привязан к thread. В Spring MVC 1 запрос = 1 thread, значит MDC живёт весь запрос. НО:
- Если ты используешь `@Async` — новый thread, MDC потерян.
- Если reactive (WebFlux) — thread меняется, MDC не работает.
- Если java 21 virtual threads — MDC работает через `ScopedValue` в новых версиях, но не всегда.

**Автоматическая очистка через Filter**:
```java
@Component
public class MdcCleanupFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear();
        }
    }
}
```

---

## 7. Correlation ID: трейсим запрос через сервисы

**Проблема**: запрос ходит `Gateway → UserService → NotificationService → RabbitMQ → SyncService`. В каждом сервисе свои логи. Как понять что все они — про ОДИН запрос?

**Решение**: генерируем UUID в Gateway (`X-Correlation-Id` header), передаём везде, кладём в MDC на входе в каждый сервис.

### 7.1 Gateway генерирует

```java
@Component
public class CorrelationIdFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest http = (HttpServletRequest) req;
        String correlationId = http.getHeader("X-Correlation-Id");
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }
        MDC.put("correlation_id", correlationId);
        ((HttpServletResponse) res).setHeader("X-Correlation-Id", correlationId);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear();
        }
    }
}
```

### 7.2 HTTP-клиент передаёт дальше

Для `RestTemplate`:
```java
@Bean
RestTemplate restTemplate() {
    RestTemplate rt = new RestTemplate();
    rt.getInterceptors().add((request, body, execution) -> {
        String cid = MDC.get("correlation_id");
        if (cid != null) request.getHeaders().add("X-Correlation-Id", cid);
        return execution.execute(request, body);
    });
    return rt;
}
```

Для Feign:
```java
@Bean
RequestInterceptor correlationIdInterceptor() {
    return template -> {
        String cid = MDC.get("correlation_id");
        if (cid != null) template.header("X-Correlation-Id", cid);
    };
}
```

### 7.3 RabbitMQ / Kafka — через message headers

```java
// producer
Message msg = MessageBuilder.withBody(payload)
    .setHeader("X-Correlation-Id", MDC.get("correlation_id"))
    .build();
rabbitTemplate.send(exchange, key, msg);

// consumer
@RabbitListener(queues = "orders")
void handle(Message msg) {
    String cid = (String) msg.getMessageProperties().getHeaders().get("X-Correlation-Id");
    MDC.put("correlation_id", cid);
    try {
        // ...
    } finally {
        MDC.clear();
    }
}
```

Теперь в Kibana: `correlation_id: "abc-123"` → вижу все логи по этому запросу через все 5 сервисов. Мощно.

---

## 8. Distributed tracing: OpenTelemetry, Jaeger, Zipkin, Tempo

Correlation ID даёт **факт связи**. Distributed tracing даёт **дерево вызовов + тайминги + структуру**.

### 8.1 Модель: Trace, Span, Context

- **Trace** — весь путь запроса через все сервисы. Один `trace_id`.
- **Span** — одна операция внутри сервиса (HTTP endpoint, DB query, вызов Feign). У каждого span свой `span_id` и ссылка на parent (`parent_span_id`).
- **Context** — trace_id + span_id + baggage (доп поля), передаётся между сервисами через HTTP headers (стандарт W3C Trace Context: `traceparent`).

Пример trace:
```
Trace abc123 (150ms всего)
├── Span 1: Gateway → POST /api/orders (150ms)
│   ├── Span 2: UserService → GET /users/42 (30ms)
│   └── Span 3: OrderService → POST /orders (100ms)
│       ├── Span 4: PostgreSQL INSERT (20ms)
│       └── Span 5: RabbitMQ publish "order.created" (5ms)
└── Span 6: NotificationService (async) SEND email (200ms)
```

Ты видишь **весь граф**, каждый узел с длительностью, включая узкие места.

### 8.2 OpenTelemetry (OTel) — стандарт

OTel — вендор-независимый SDK. Собирает spans в приложении, отправляет в **OTel Collector** (proxy), тот — в бэкенд (Jaeger, Tempo, Zipkin, Datadog, etc.).

Boot 3 интеграция (auto-instrumentation через агент):
```dockerfile
FROM eclipse-temurin:21
COPY app.jar /
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar /agent.jar
ENV JAVA_TOOL_OPTIONS="-javaagent:/agent.jar"
ENV OTEL_SERVICE_NAME=isnaknpsync
ENV OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
ENV OTEL_TRACES_EXPORTER=otlp
CMD java -jar /app.jar
```

Агент авто-инструментирует: HTTP endpoints (Spring MVC), JDBC, HTTP-clients (RestTemplate, WebClient), RabbitMQ. Spans создаются автоматически.

Ручное:
```java
@Autowired Tracer tracer;

void process() {
    Span span = tracer.spanBuilder("process-order").startSpan();
    try (Scope scope = span.makeCurrent()) {
        span.setAttribute("order_id", orderId);
        // ...
    } catch (Exception e) {
        span.recordException(e);
        span.setStatus(StatusCode.ERROR);
        throw e;
    } finally {
        span.end();
    }
}
```

### 8.3 Бэкенды

- **Jaeger** — классика, все-в-одном (UI + storage). Простой старт.
- **Zipkin** — постарше, тоже все-в-одном.
- **Tempo** — Grafana Labs, интегрируется с Loki/Prometheus. Дёшево (S3 storage).
- **Datadog / New Relic** — коммерческие SaaS, дорого но с alerting/AI.

### 8.4 Связь logs ↔ traces

OTel Java Agent автоматически инжектит `trace_id` и `span_id` в MDC. Если у тебя structured JSON logs — они уже там. В Grafana/Kibana ты можешь **из лога кликнуть → перейти в trace**. Это observability nirvana.

---

## 9. Retention, sampling, стоимость

Логи — самое дорогое в observability. Реальные объёмы:
- Средний Boot pod с DEBUG на root: 1-10 GB/день/pod.
- Кластер на 50 подов = 500 GB/день. Elasticsearch хранит с replication x2, retention 30 дней = 30 TB диска. Не дёшево.

Тактики:
- **INFO в проде**, не DEBUG. DEBUG только на нужные packages.
- **Sampling** для traces: 100% требований на error, 1-10% на success.
- **Retention**: горячие 7 дней в ES, потом archive в S3.
- **Rate limits**: если pod вдруг зафлудил (bug), Fluent Bit может обрубить.

---

## 10. Пример полного стека для isna-knp

Если бы я строил observability стек для КНП сегодня:

**Логи:**
- Boot: Logback + logstash-encoder, structured JSON.
- MDC: correlation_id, user_id, taxpayer_code.
- DaemonSet: Fluent Bit → Loki.
- UI: Grafana для чтения.

**Метрики:**
- Boot: Micrometer + Prometheus registry.
- Actuator: `/actuator/prometheus`.
- Prometheus scrape каждый pod.
- UI: Grafana дашборды.

**Traces:**
- OTel Java agent на каждый Boot.
- OTel Collector в кластере.
- Backend: Tempo (S3 хранит cheap).
- UI: Grafana Explore, jump from logs → traces.

Все три источника в **одном UI (Grafana)** — самое удобное.

---

## 11. Как читать инцидент в этом стеке

Пример: alert "5xx growing on isnaknpuser". Твои шаги:

1. **Grafana → Prometheus alert firing at 10:15**. Смотришь график rate по коду ответа.
2. **Grafana → Loki**. Query: `{app="isnaknpuser"} |= "ERROR"` за окно 10:10-10:20. Читаешь топ ошибок.
3. Находишь стектрейс: `PSQLException: connection timeout`.
4. **Grafana → Tempo**. Кликаешь на trace_id из лога. Видишь: 90% времени в JDBC query. Postgres был медленный.
5. **Grafana → Prometheus**. Смотришь метрики `pg_stat_activity` — 200 idle-in-transaction connections. Пул хикари забит.
6. Причина: где-то не закрывается транзакция. По trace находишь метод. Fix.

Без стека это заняло бы 3 часа `grep -r` в логах на нодах вручную.

---

## 12. Anti-patterns

- **`log.info("Обработка")`** — бесполезно, ничего не говорит. Пиши `log.info("Processing order", kv("order_id", id))`.
- **`log.error(msg, e)`** без стектрейса — стектрейс проглотится если не так вызвать. В SLF4J: `log.error("Failed", e)` — правильно (e идёт последним аргументом без плейсхолдера).
- **`try { ... } catch (Exception e) { log.error(e.getMessage()); }`** — потеряет стектрейс. Логируй `e` целиком: `log.error("Failed", e)`.
- **DEBUG в проде на root** — залил диск, стек лёг.
- **Sensitive data в логах**: пароли, токены, ПИИ. Отдельная тема compliance.
- **Логировать в каждом методе `Entering foo() ... Exiting foo()`** — шум. Structured events на границах бизнес-операций.

---

## 13. Кратко

- Boot пишет в stdout → containerd/kubelet → файл на ноде.
- DaemonSet (Fluent Bit/Promtail) собирает файлы, обогащает k8s-метаданными, отправляет в backend.
- Backend: ES (дорого, гибко) или Loki (дёшево, простой поиск).
- Structured JSON logs + MDC с correlation_id — трейс запроса через сервисы.
- OpenTelemetry + Jaeger/Tempo — полное дерево вызовов + тайминги.
- Grafana даёт единый UI для logs + metrics + traces.
- В проде: INFO root, structured JSON, correlation_id везде, retention 7-30 дней в hot storage.
