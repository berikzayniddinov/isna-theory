# 38. Логирование: SLF4J, Logback, MDC, ELK

## Зачем понимать logging глубже println

Разработчик который впервые изучает Java обычно начинает с System.out.println. Это работает в hello-world tutorials, работает для debugging локальных программ, работает для простых утилит. Но когда приложение попадает в production — println становится проблемой. Куда идёт вывод? Как найти конкретное событие среди миллионов строк? Как понять что это было — request от пользователя или scheduled job? Как альтернативные компоненты (например мониторинг) могут узнать что произошла ошибка?

Логи в production выполняют несколько связанных задач которые println не решает. Debugging — найти причину бага когда пользователь сообщает о проблеме несколькими часами позже. Audit — «кто что сделал когда» для compliance и security investigations. Monitoring — «сколько ошибок за минуту» через log-based метрики. Compliance — регуляторные требования могут мандировать конкретное logging (например налоговое законодательство, банковские регуляции). Observability — вместе с метриками и traces образует полную картину поведения системы во времени.

Разница между разработчиком «пишущим логи» и «правильно пишущим логи» огромна и часто не осознаётся. Первый пишет log.info("Processing " + user.toString() + " with data " + data) и не думает. Второй знает что конкатенация string вычисляется всегда независимо от уровня logging (даже если INFO выключен), что String.format дорог, что placeholder syntax {} lazy evaluates и дешевле, что MDC привязан к thread и при @Async теряется, что printStackTrace идёт в stderr и не парсится ELK как error event, что log.debug("Deep info: {}", heavyComputation()) вычислит heavyComputation даже если DEBUG выключен если аргумент не lazy.

В этом файле разберём logging как comprehensive system. Уровни logging и их семантика. Разница API (SLF4J) и implementation (Logback, Log4j). Что actually happens при вызове log.info на уровне library internals. Placeholders и почему они дешевле конкатенации. MDC (Mapped Diagnostic Context) и почему это mission-critical для tracing запросов. Logback конфигурация полностью — appenders, encoders, rolling policies, async processing. Structured logging JSON для machine-parseable output. ELK stack (Elasticsearch, Logstash, Kibana) plus Beats. Специфика Kubernetes logging pipelines. Real-world caveats — что не логировать, как не влиять на performance, как избежать sensitive data leaks. Кейсы КНП где неправильный logging создал реальные production issues.

## Уровни logging и их правильная семантика

Стандартные уровни выстроены от наиболее детального к наиболее критичному.

TRACE это очень детальный уровень — метод-по-методу execution, каждый значимый step алгоритма. Редко включается в production потому что генерирует enormous volume. Полезен для сложных debugging сессий где нужно проследить execution flow буквально каждого действия.

DEBUG детально для отладки — значения переменных, промежуточные шаги алгоритмов, decisions taken. Обычно off в production но включается для конкретных packages при активной investigation. Правильно написанный DEBUG log позволяет реконструировать что происходило без reproducing локально.

INFO важные бизнес события — старт сервиса, успешная обработка значимого запроса, notable state changes. Standard level для production business events. Ключевой критерий — событие важно enough чтобы operators хотели видеть, но happens относительно infrequently (не десятки тысяч в секунду).

WARN что-то подозрительное но не критичная ошибка — retry attempts, использование deprecated API, edge cases требующие attention. Sign что-то нуждается в review но система функционирует.

ERROR ошибка требующая внимания — failure operation, exception при обработке. Обычно triggers alerting в monitoring системах. Правильно используемый ERROR level это «человеку нужно об этом узнать и посмотреть».

FATAL критично, приложение может упасть. Редко используется — обычно ERROR достаточен. Некоторые frameworks (Log4j) поддерживают, другие (Logback) нет — считают что error достаточно.

Приложение настраивается на минимальный уровень. INFO показывает INFO, WARN, ERROR и скрывает DEBUG, TRACE:
```yaml
logging:
  level:
    root: INFO
    kz.gov.kgd.isna: DEBUG
    org.hibernate.SQL: DEBUG
```

Правило production — INFO на root plus DEBUG для своих packages при необходимости. Никогда DEBUG на root в production. Залил бы ELK, retention упал бы, файлы взорвались бы от volume. Каждый DEBUG log цена в CPU, memory, disk I/O, ELK storage — умножить на throughput получишь real cost.

Правильное использование levels критично для alerting. ERROR должен означать alert-worthy. WARN — «investigate когда есть время». INFO — событие для understanding но не urgent. Если ERROR используется на любой exception (включая expected validation failures) — alerting fatigue overwhelms real problems. Правильная calibration уровней это discipline.

## SLF4J vs Logback: API против implementation

Java ecosystem имеет несколько logging APIs и implementations. Понимание разницы critical для understanding как configuration работает.

SLF4J (Simple Logging Facade for Java) это APIs standard де-факто. Interfaces которые использует твой code. Компилируется против SLF4J API. При запуске находит implementation на classpath и делегирует. Позволяет менять implementation без recompilation code.

Implementation options. Logback — default в Spring Boot, разработан тем же автором что SLF4J (Ceki Gülcü). Sensible defaults, широко используется, actively maintained. Log4j2 — альтернатива с хорошей performance especially для async logging. Некоторые enterprise projects prefer. Log4j 1.x — устарел, имеет security уязвимости, не использовать. JUL (java.util.logging) — встроен в JDK но редко используется напрямую из-за slow performance и weak features.

Bridge libraries обеспечивают integration когда сторонние библиотеки используют другие logging APIs. jul-to-slf4j редиректит JUL calls в SLF4J. jcl-over-slf4j для Commons Logging. log4j-over-slf4j для Log4j 1.x. Spring Boot автоматически включает эти bridges — вся ecosystem logging goes through unified pipeline.

Схема работы полная:
```
Твой код (import org.slf4j.Logger)
   ↓ compile-time linking
SLF4J API interfaces
   ↓ runtime lookup через ServiceLoader
Logback implementation
   ↓ actual writing
File / Console / Kafka / ELK
```

Log4Shell как historically important security incident. CVE-2021-44228 discovered late 2021. Log4j 2.x до version 2.17 vulnerable к RCE через ${jndi:...} pattern в log messages. Attacker sends specifically crafted string к приложению which logs it. Log4j 2 evaluates JNDI expression в message loading remote code. Полный remote code execution от single innocuous-looking log message.

Все Log4j 2.x installations должны быть updated to 2.17+. Logback не подвержен — не имеет similar JNDI evaluation feature. Historical lesson — even simple-seeming subsystems like logging могут иметь catastrophic vulnerabilities.

## Логика вызова log.info

Разберём что происходит когда developer пишет log.info("Message"). Understanding this deep помогает reasoning about performance implications.

Loggers создаются через LoggerFactory:
```java
private static final Logger log = LoggerFactory.getLogger(OrderService.class);
```

LoggerFactory это SLF4J's factory. Внутри при первом вызове initializes StaticLoggerBinder который binds к actual implementation (Logback). Возвращает org.slf4j.Logger interface. getLogger caches по className — same logger returned на repeated calls.

При log.info("Message") — SLF4J Logger delegates к underlying Logback Logger. Первое что происходит — check level. Каждый Logger имеет configured level. Если log.isInfoEnabled() returns false (INFO disabled для этого logger) — метод returns immediately без further processing.

Если level enabled — actual event created. LoggingEvent object с message, timestamp, thread, logger name, level, throwable if any, MDC contents. Событие passed через configured filters. Если passes — passed to appenders. Each appender processes event по своему — encoder formats event to string/JSON, writer sends output к destination (file, console, network).

Actually blocking behavior важен. Default appenders synchronous — log.info blocks until output written. Для FileAppender это может mean disk I/O time — миллисекунды under load. Multiplied на high throughput — measurable performance impact.

Async appender solution — обсудим ниже. Но understanding basic flow important — синхронный synchronous logging может быть bottleneck.

## Placeholders и почему они важны

Ключевое performance правило SLF4J — использовать placeholder syntax not concatenation:
```java
log.debug("Order: " + o.toString());
log.debug("Order: {}", o);
```

Difference on runtime critical. First version — concatenation evaluated always. o.toString() called always. Result string constructed always. Только then log.debug called. If DEBUG disabled — event dropped immediately after level check, но CPU work already wasted.

Second version с placeholder — args passed as Object array. SLF4J internally checks level first. Only if enabled — invokes toString on args and substitutes into message template. Wasted work только when level enabled and message actually written.

Impact on production. High-throughput service processing 1000 requests/sec. Каждый request может иметь несколько log.debug calls "for future debugging". If concatenation used — 1000 × N × toString() calls per second even when DEBUG disabled. Complex objects with heavy toString (deep hierarchies) может cost seconds of CPU. With placeholders — zero cost when disabled.

Exception logging pattern:
```java
log.error("Failed to save {}", orderId, exception);
```

SLF4J recognizes when last argument is Throwable — treats it specially. Message formatted with orderId substituted. Exception logged with full stack trace separately. Correct pattern preserved through formatting.

Anti-pattern — including exception in message:
```java
log.error("Failed: " + exception.getMessage());
```

getMessage returns только localized message без stack trace. Stack trace lost. Debugging becomes much harder — you know что failed но not where.

isDebugEnabled guard для heavy operations:
```java
if (log.isDebugEnabled()) {
    log.debug("Complex state: {}", heavyToString(state));
}
```

Иначе heavyToString(state) вычислится еще до вызова log.debug — args evaluated eagerly в Java даже с placeholder syntax. Guard explicit проверяет level перед expensive computation.

Для simple placeholders (existing variables) guard не нужен — минимальный overhead от argument passing.

## MDC (Mapped Diagnostic Context)

MDC это thread-local Map для storing context информации доступной для logging. Correlation ID, user ID, request ID — все хранятся в MDC и автоматически включаются в каждый log message без явной передачи в каждый вызов.

Implementation через ThreadLocal<Map<String, String>>. Каждый thread имеет свою map. Values stored через MDC.put(key, value). Retrieved automatically pattern layout через %X{key}.

Установка:
```java
MDC.put("traceId", UUID.randomUUID().toString());
MDC.put("userId", currentUser.getId());
try {
    // работа
} finally {
    MDC.clear();
}
```

В pattern использование через %X{key}:
```
%X{traceId} %X{userId}
```

Результат в логе:
```
2026-09-05 10:15 [abc123, berik] INFO OrderService - Saving order 42
```

MDC plus filter установка в начале каждого HTTP request:
```java
@Component
class MdcFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) {
        try {
            String traceId = ((HttpServletRequest) req).getHeader("X-Trace-Id");
            if (traceId == null) traceId = UUID.randomUUID().toString();
            MDC.put("traceId", traceId);
            chain.doFilter(req, resp);
        } finally {
            MDC.clear();
        }
    }
}
```

Все logs одного request будут с одинаковым traceId позволяя correlate их в ELK. Powerful debugging tool — search by traceId shows все logs всего request.

MDC caveat с async operations. MDC привязан к потоку через ThreadLocal. При @Async, virtual threads, ExecutorService MDC теряется — новый thread не имеет исходного context. Reading MDC returns null.

Решение через TaskDecorator копирующий MDC при передаче задачи в executor:
```java
public class MdcTaskDecorator implements TaskDecorator {
    public Runnable decorate(Runnable runnable) {
        Map<String, String> ctx = MDC.getCopyOfContextMap();
        return () -> {
            try {
                MDC.setContextMap(ctx);
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

Captures MDC state at submission time. Restores в worker thread before running task. Clears afterwards to avoid leaking context в pooled threads.

Или использовать Micrometer Tracing which correctly propagates context через async boundaries automatically. Более cleanly integrated solution.

## Logback конфигурация полностью

Файл logback-spring.xml в src/main/resources стандартный location. Spring Boot автоматически обнаруживает и использует. Пример полной production конфигурации:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <property name="LOG_PATTERN"
        value="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%X{traceId:-},%X{spanId:-}] [%thread] %logger{36} - %msg%n"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="FILE"/>
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <neverBlock>true</neverBlock>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="ASYNC_FILE"/>
    </root>

    <logger name="kz.gov.kgd.isna" level="DEBUG"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
</configuration>
```

Ключевые элементы конфигурации. Appender определяет куда писать — Console (stdout), File, Kafka, HTTP endpoint, etc. Encoder форматирует output — pattern layout или JSON structured. RollingPolicy управляет rotation файлов по времени или размеру предотвращая единый файл от unbounded growth. Logger настраивает уровень для конкретного package overriding root. Root default для всего кроме specific loggers.

Pattern layout использует специальные placeholders. %d{format} для timestamp с custom format. %-5level уровень left-aligned до 5 chars. %X{key} для MDC context values. %thread имя потока обрабатывающего request. %logger{36} имя класса сокращённое до 36 chars. %msg сам message. %n newline. %ex exception details (stack trace).

Async appenders для performance. Synchronous logging значит каждая log.info блокирует текущий поток пока запись действительно попадёт на disk. На high-load это заметно.

AsyncAppender решает эту проблему через queue plus background thread. Application thread просто добавляет log event в queue и продолжает работу. Background thread достаёт events из queue и пишет к underlying appender.

Caveats. При JVM crash необлитая queue теряется — recent logs могут пропасть. neverBlock=false — при full queue log.info блокируется что делает async бессмысленным для peaks. neverBlock=true — при full queue events просто dropped, тоже loss.

Правило для production — async plus discardingThreshold=0 (не терять по threshold) plus увеличенный queueSize (например 1024-4096) для handling bursts.

## Structured logging JSON для ELK

Обычные text logs удобны для humans но не идеальны для machines. ELK может парсить но не всегда точно. Structured logging через JSON формат делает parsing тривиальным:
```
2026-09-05 10:15:30 INFO OrderService - Saving order 42
```

vs JSON:
```json
{
  "@timestamp": "2026-09-05T10:15:30.123Z",
  "level": "INFO",
  "thread": "http-nio-8080-exec-1",
  "logger": "kz.gov.kgd.isna.knp.OrderService",
  "message": "Saving order 42",
  "traceId": "abc123",
  "userId": "berik",
  "app": "isna-knp",
  "env": "prod"
}
```

Использование logstash-logback-encoder library:
```gradle
implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
```

Конфигурация appender с LogstashEncoder. ELK парсит JSON легко и точно. Все fields доступны для filtering, aggregation, dashboards в Kibana.

kv arguments для structured data. Логика — вместо embedding data в message string, ставить как separate fields:
```java
log.info("Order saved", kv("orderId", o.getId()), kv("status", o.getStatus()));
```

В Kibana можно filter orderId:42 напрямую вместо regex по message. Более efficient search и aggregation. Data types preserved — числа как numbers, dates as dates.

## ELK stack

ELK это Elasticsearch plus Logstash plus Kibana. Standard стек для centralized logging.

Elasticsearch хранит и индексирует логи. Full-text search plus structured field queries. Distributed для horizontal scaling. Обсуждается детально в файле 45.

Logstash это pipeline для обработки логов. Читает из различных sources (Filebeat, Kafka), parses и трансформирует, пишет в Elasticsearch. Написан на JRuby. Тяжелый по CPU и memory. Часто заменяется на легковесные alternatives.

Beats это семейство легковесных shippers. Filebeat читает файлы, отправляет в Elasticsearch или Logstash. Metricbeat собирает metrics. Auditbeat для audit logs. Написаны на Go — быстрые и легковесные.

Kibana это UI для Elasticsearch. Search, dashboards, alerts, visualization. Стандартный way для interacting с ELK data.

Типичная pipeline:
```
Приложение
    │  пишет JSON лог в файл
    ▼
[app.json]
    │  Filebeat читает
    ▼
[Filebeat] ─────────► [Elasticsearch]
                            │
                            ▼
                         [Kibana]  ← ты смотришь
```

Альтернативная pipeline через message broker для reliability:
```
Приложение → Logback appender → Kafka → Logstash → Elasticsearch
```

В КНП своя ELK infrastructure называется isna-elk. Логи через kubectl exec в pod Elasticsearch для historical queries. Memory кейс knp-prod-historical-logs-elk напоминает — kubectl logs даёт только current pod logs, для history надо ELK.

Retention policies. Логи не хранятся вечно — expensive storage. Hot indices последние 7 дней быстрый доступ. Warm indices 30 дней медленнее. Cold 90+ дней архив. Delete старше. ILM (Index Lifecycle Management) в Elasticsearch автоматизирует эти transitions.

## Kubernetes logging pipelines

Приложения в контейнерах пишут в stdout и stderr. Docker или kubelet перехватывает, пишет в файлы на node (/var/log/pods/...).

Access через kubectl:
```
kubectl logs <pod>            # текущий container
kubectl logs <pod> --previous # предыдущий instance (после restart)
```

Ротация kubelet сохраняет ограниченное количество history. По умолчанию 10 MB per file plus N старых files. Старше суток обычно уже нет — memory кейс knp-prod-historical-logs-elk именно об этом.

Fluentd или Filebeat как DaemonSet на каждой worker node. Читает pod логи из /var/log/pods, отправляет в ELK. Стандартная K8s logging architecture.

## Что НЕ логировать

Секреты absolutely never. Пароли, tokens, keys — все sensitive credentials. Личные данные (PII) — имена, email addresses, phone numbers, addresses когда protected regulations. Кредитные карты — PCI compliance требования.

Фильтрация через regex в encoder:
```xml
<encoder>
    <pattern>%replace(%msg){'Bearer [A-Za-z0-9._-]+', 'Bearer ***'}</pattern>
</encoder>
```

Заменяет любой Bearer token на marker в log output. Prevents accidental logging tokens когда developer забыл.

Огромные objects. log.info("Response: {}", hugeJson) где hugeJson 10 MB — это 10 MB в лог per request. Смерть ELK при significant volume. Truncate или skip:
```java
log.info("Response: {}", StringUtils.left(hugeJson, 500));
```

Every SQL в production через org.hibernate.SQL: DEBUG — тысячи queries в секунду в лог. Only для debugging специфических issues. Regular production logging бы захлебнула storage.

## printStackTrace как классическая проблема

Никогда не использовать e.printStackTrace(). Common anti-pattern в legacy code plus copy-paste tutorials:
```java
} catch (Exception e) {
    e.printStackTrace();  // WRONG!
}
```

Проблемы. Output goes to stderr not through SLF4J. ELK typically parses stdout — stderr treated separately или ignored. Structured logging bypassed — no traceId, no MDC context. Not machine-parseable — alerting can't detect based на these. Development artifact leaking в production.

Правильно:
```java
} catch (Exception e) {
    log.error("Operation failed for id={}", orderId, e);
}
```

Structured message plus context plus exception through SLF4J.

Реальный кейс из КНП. Memory knp-fo-sync-notification-bugs — printStackTrace создало «ELK-слепую зону». Ошибки не visible в ELK потому что stderr не indexed. 6 багов копились months потому что error monitoring не seeing их. Real cost of небольшой convenience shortcut.

## Правила использования

Структурированный approach vs printStackTrace. Всегда SLF4J logging с exception как last arg.

Уровни правильно calibrated. ERROR alert-worthy. WARN — investigate когда есть время. INFO — важные business events. DEBUG — details for debugging, off in production. Не log.error на любую exception — если это expected (validation) то WARN или INFO.

Correlation ID везде. HTTP header X-Request-Id или generated. MDC plus %X{traceId} в pattern. Пробрасывается downstream через Feign interceptor или HTTP client interceptor. Через RabbitMQ или Kafka message headers.

Search traceId:abc123 в Kibana shows всю chain через все services. Distributed tracing without full distributed tracing tool.

Не logging в hot path. Метод дёргается 100000 раз в секунду — каждый log call is disk write, смерть системы. Используй метрики (counter, gauge) для monitoring, log только для аномалий требующих investigation.

Sampling для partial logging когда полное overkill:
```java
if (ThreadLocalRandom.current().nextInt(100) == 0) {
    log.info("Sampled request: {}", req);
}
```

Логирует 1 процент requests. Statistical sample достаточен для understanding patterns без volume overhead.

## Best-practice logger

Собранный воедино правильный approach:
```java
@Slf4j
@Service
public class OrderService {

    private final OrderRepository repo;
    private final Counter savedCounter;
    private final Timer saveTimer;

    public OrderService(OrderRepository repo, MeterRegistry registry) {
        this.repo = repo;
        this.savedCounter = registry.counter("orders.saved");
        this.saveTimer = registry.timer("orders.save.time");
    }

    public Order save(OrderRequest req) {
        return saveTimer.record(() -> {
            log.info("Saving order for customerId={}", req.customerId());

            try {
                Order o = new Order(req);
                repo.save(o);
                savedCounter.increment();
                log.debug("Order saved: {}", o.getId());
                return o;
            } catch (DataAccessException e) {
                log.error("DB error saving order for customerId={}", req.customerId(), e);
                throw new BusinessException("db_error", e);
            }
        });
    }
}
```

Комбинация logging plus метрики. INFO для важных бизнес events. DEBUG для деталей. ERROR plus stack trace на неожиданное. Metrics counter и timer параллельно с logs для aggregate monitoring без heavy log volume.

## Итоги

SLF4J API plus Logback implementation default в Spring Boot. Использовать через SLF4J API для decoupling.

Placeholder syntax `{}` не concatenation. Lazy evaluation экономит CPU когда level not enabled.

Уровни — DEBUG для development, INFO для важных events, WARN для подозрительного, ERROR для alert-worthy. Правильная calibration критична для alerting.

MDC для correlation ID и context. Thread-local через ThreadLocal — теряется на async boundaries без proper handling (TaskDecorator plus MDC copying).

Structured JSON для ELK. logstash-logback-encoder library. Custom fields plus MDC. Machine-parseable, precise filtering в Kibana.

AsyncAppender для performance. discardingThreshold=0 чтобы не терять. Increased queueSize для bursts.

ELK stack (Elasticsearch plus Logstash или Beats plus Kibana) стандарт для centralized logging. Retention через ILM.

Kubernetes stdout/stderr перехватывается kubelet. Filebeat DaemonSet шипает в ELK. kubectl logs только для current, historical через ELK.

Никогда логировать секреты, PII, huge objects, every SQL в prod. Regex masking в encoder patterns.

Никогда printStackTrace — используй log.error с exception. Реальный урок из КНП где attentioning ELK-blind zone позволил bugs копиться месяцами.

Correlation ID через MDC plus HTTP headers plus message headers для distributed traceability.

Метрики параллельно с логами. Observability = logs + metrics + traces. Each provides different perspective, combined give complete picture.

Дальше — Kafka базовые concepts как основа для understanding streaming platform.
