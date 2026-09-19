# 38. Логирование: SLF4J, Logback, ELK

## Зачем логи

Logging решает несколько связанных задач в production системе. Debugging — найти причину bugа когда пользователь сообщает о проблеме. Audit — «кто что сделал когда» для compliance и security investigations. Monitoring — «сколько ошибок за минуту» через log-based метрики. Compliance — регуляторные требования часто мандируют определённый logging. Observability — вместе с метриками и traces образует полную картину системы.

Фундаментальное правило — логи в production это НЕ println. Это структурированные события с уровнем важности, timestamp, контекстной информацией. Каждый log message должен быть actionable — приносить value для troubleshooting или monitoring.

## Уровни логирования

Стандартные уровни от наименее до наиболее важного.

TRACE это очень детальный уровень — метод-по-методу execution. Редко используется в production потому что генерирует enormous volume. Полезен для сложных debugging сессий.

DEBUG детально для отладки — значения переменных, промежуточные шаги алгоритмов. Обычно off в production но включается для конкретных пакетов при необходимости.

INFO важные события — старт сервиса, успешная обработка запроса, значимые state changes. Standard level для production business events.

WARN что-то подозрительное но не критичная ошибка — retry attempts, использование deprecated API, edge cases требующие attention.

ERROR ошибка требующая внимания — failure operation, exception при обработке. Обычно triggers alerting.

FATAL критично, приложение может упасть. Редко используется — обычно ERROR достаточен.

Приложение настраивается на минимальный уровень. INFO показывает INFO, WARN, ERROR и скрывает DEBUG, TRACE:
```yaml
logging:
  level:
    root: INFO
    kz.gov.kgd.isna: DEBUG
    org.hibernate.SQL: DEBUG
```

Правило production — INFO на root plus DEBUG для своих packages при необходимости. Никогда DEBUG на root в production. Залил бы ELK, retention упал бы, файлы взорвались бы от volume.

## Экосистема Java logging

Разделение на API (фасад) и implementation. Приложение работает через API interfaces не привязываясь к конкретной implementation. Позволяет менять implementation без изменения code.

API options. SLF4J (Simple Logging Facade for Java) это стандарт де-факто, используется в 99 процентов Java проектов. Commons Logging (JCL) старый API, используется через bridge для legacy compatibility. JBoss Logging фасад от JBoss стека.

Implementation options. Logback это default в Spring Boot, разработан тем же автором что SLF4J (Ceki Gülcü). Log4j2 это альтернатива с good performance. Log4j 1.x устарел, имеет security уязвимости, не использовать. JUL (java.util.logging) встроен в JDK но редко используется напрямую из-за slow performance.

Схема работы:
```
Твой код → SLF4J → Logback → File / Console / ELK
```

Bridge libraries обеспечивают integration когда сторонние библиотеки используют другие logging APIs. jul-to-slf4j редиректит JUL calls в SLF4J. jcl-over-slf4j для Commons Logging. log4j-over-slf4j для Log4j 1.x. Spring Boot автоматически включает эти bridges чтобы все logging шло через единый pipeline.

Важная security note — Log4Shell (CVE-2021-44228). Уязвимость в Log4j 2.x до версии 2.17. RCE через ${jndi:...} pattern в log messages. Attacker может отправить specially crafted строку в приложение, оно логирует, Log4j 2 evaluates JNDI expression загружая remote code. Обновиться немедленно если использовался Log4j 2. Logback не подвержен потому что не имеет такой feature.

## Использование SLF4J

Basic usage через LoggerFactory:
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public void save(Order o) {
        log.info("Saving order {}", o.getId());
        try {
            repo.save(o);
            log.debug("Order saved: {}", o);
        } catch (Exception e) {
            log.error("Failed to save order {}", o.getId(), e);
            throw e;
        }
    }
}
```

Через Lombok упрощается до одной аннотации:
```java
@Slf4j
public class OrderService {
    // log переменная генерируется автоматически
}
```

Placeholders через {} критически важны. Использовать placeholder syntax, не string concatenation:
```java
log.debug("Order: " + o.toString());   // toString ВЫЗЫВАЕТСЯ ВСЕГДА, даже если DEBUG выключен

log.debug("Order: {}", o);              // toString ТОЛЬКО если DEBUG включён
```

Разница на production. Тысячи toString в секунду on high-load когда логирование disabled равно significant CPU waste. С placeholders — 0 overhead когда level not enabled. SLF4J lazy evaluates placeholders только когда log actually pишется.

Exception как последний аргумент — SLF4J распознаёт и включает stacktrace:
```java
log.error("Failed to save {}", orderId, exception);
// exception выводится как stacktrace, orderId подставляется в {}
```

Anti-pattern — включение exception в message string теряет stacktrace:
```java
log.error("Failed: " + exception.getMessage());   // теряется stacktrace!
```

isDebugEnabled guard для heavy operations:
```java
if (log.isDebugEnabled()) {
    log.debug("Complex state: {}", heavyToString(state));
}
```

Иначе heavyToString вычислится даже если DEBUG выключен. Для simple placeholders guard не нужен — SLF4J handles lazy evaluation.

## Logback конфигурация

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

Ключевые элементы. Appender определяет куда писать — Console, File, Kafka, HTTP. Encoder форматирует output — pattern layout или JSON structured. RollingPolicy управляет rotation файлов по времени или размеру. Logger настраивает уровень для конкретного package. Root default для всего кроме specific loggers.

Pattern layout использует специальные placeholders. %d{format} для timestamp с custom format. %-5level уровень left-aligned до 5 chars. %X{key} для MDC context values. %thread имя потока обрабатывающего request. %logger{36} имя класса сокращённое до 36 chars. %msg сам message. %n newline. %ex exception details (stacktrace).

## Async appenders

Synchronous logging значит каждая log.info блокирует текущий поток пока запись действительно попадёт на disk. На high-load это заметно — thousands calls в секунду каждый плюс несколько миллисекунд.

AsyncAppender решает эту проблему через queue plus background thread. Application thread просто добавляет log event в queue и продолжает работу. Background thread достаёт events из queue и пишет к underlying appender:
```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE"/>
    <queueSize>512</queueSize>
    <discardingThreshold>0</discardingThreshold>   <!-- 0 = не терять -->
    <neverBlock>true</neverBlock>                   <!-- true = при переполнении дропать -->
</appender>
```

Caveats. При JVM crash необлитая queue теряется — recent logs могут пропасть. neverBlock=false — при full queue log.info блокируется что делает async бессмысленным для peaks. neverBlock=true — при full queue events просто dropped, тоже loss.

Правило для production — async plus discardingThreshold=0 (не терять по threshold) plus увеличенный queueSize (например 1024-4096) для handling bursts.

## MDC (Mapped Diagnostic Context)

MDC это thread-local Map для storing context информации доступной для logging. Correlation ID, user ID, request ID — все хранятся в MDC и автоматически включаются в каждый log message.

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

Все logs одного request будут с одинаковым traceId позволяя correlate их в ELK.

MDC caveat с async operations. MDC привязан к потоку через ThreadLocal. При @Async, virtual threads, ExecutorService MDC теряется — новый поток не имеет исходного context.

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

Или использовать Micrometer Tracing который правильно пробрасывает context через async boundaries.

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

Конфигурация appender:
```xml
<appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/app.json</file>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdc>true</includeMdc>
        <customFields>{"app":"isna-knp","env":"prod"}</customFields>
    </encoder>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>logs/app.%d.log.gz</fileNamePattern>
    </rollingPolicy>
</appender>
```

ELK парсит JSON легко и точно. Все fields доступны для filtering, aggregation, dashboards в Kibana.

kv arguments для structured data:
```java
log.info("Order saved", kv("orderId", o.getId()), kv("status", o.getStatus()));
```

Через StructuredArguments из logstash-encoder:
```java
log.info("Order saved: {}", value("orderId", o.getId()));
```

Плюс — в Kibana можно filter orderId:42 напрямую вместо regex по message. Более efficient search и aggregation.

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

## Kubernetes и логи

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

Заменяет любой Bearer token на marker в log output. Predотвращает случайное logging tokens когда developer забыл.

Огромные objects. log.info Response: {}, hugeJson где hugeJson 10 MB — это 10 MB в лог per request. Смерть ELK при значительной нагрузке. Truncate или skip:
```java
log.info("Response: {}", StringUtils.left(hugeJson, 500));
```

Every SQL в production через `org.hibernate.SQL: DEBUG` — тысячи queries в секунду в лог. Only для debugging специфических issues. Regular production logging бы захлебнула storage.

## Правила использования

Structured plus не printStackTrace. Никогда:
```java
} catch (Exception e) {
    e.printStackTrace();       // stderr, ELK не парсит как error
}
```

Правильно:
```java
} catch (Exception e) {
    log.error("Operation failed for id={}", orderId, e);
}
```

Реальный кейс КНП knp-fo-sync-notification-bugs — printStackTrace создавало «ELK-слепую зону». Ошибки не видны в мониторинге. 6 багов копились месяцами потому что error monitoring not seeing их.

Уровни правильно. ERROR — что-то надо делать (alert, incident). WARN — проверить когда есть время (deprecated API, retry). INFO — важные события пользователя. DEBUG — детали для отладки, off в prod.

Не пиши log.error на любой exception — если это ожидаемая (validation) то WARN или INFO. Правильный уровень critical для alerting effectiveness. Log.error должно означать «investigate это».

Correlation ID везде. Приходит в HTTP header X-Request-Id или генерируется если отсутствует. MDC plus traceId в pattern. Пробрасывается в downstream через Feign interceptor или HTTP client interceptor. Пробрасывается в Rabbit или Kafka message headers.

Потом в Kibana search traceId:abc123 показывает всю chain через все сервисы. Distributed tracing без full distributed tracing tool.

Не логировать в hot path. Метод дёргается 100000 раз в секунду — каждый log call это disk write, смерть системы. Используй метрики (counter, gauge) для такого monitoring, а лог только для аномалий требующих investigation.

Sampling для partial logging когда полное невозможно:
```java
if (ThreadLocalRandom.current().nextInt(100) == 0) {
    log.info("Sampled request: {}", req);
}
```

Логирует 1 процент requests. Statistical sample достаточен для understanding patterns без volume overhead. Или через Micrometer Tracing sampling для automatic scheme.

## Best-practice logger для сервиса

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

Комбинация logging plus метрики. INFO для важных бизнес events. DEBUG для деталей. ERROR plus stacktrace на неожиданное. Metrics counter и timer параллельно с logs для aggregate monitoring.

## Реальные кейсы КНП

Memory knp-fo-sync-notification-bugs — printStackTrace создало «ELK-слепую зону». 6 багов копились месяцами потому что error monitoring не видел их. Urok — structured logging обязательно, printStackTrace никогда не использовать.

Memory knp-prod-historical-logs-elk — kubectl logs показывает только current, историческое через ES queries. kubectl exec в pod ES для complex historical searches. Standard practice знать где искать historical logs.

Fno328 регенерация — важен log каждой обработанной entity чтобы отследить прогресс job. Long-running batch operations без progress logging сложно troubleshoot когда что-то идёт wrong.

## Правила для команды

Никакого printStackTrace. Всегда log.error с exception как last argument.

Structured logging JSON для production. Text logs только для local development.

MDC plus traceId в каждом request. Correlation across services.

Метрики параллельно с логами. Log для events plus metrics для counters/timers.

Уровни правильно. ERROR означает alert-worthy. Не логировать expected exceptions как ERROR.

## Итоги

SLF4J API plus Logback implementation default в Spring Boot. Правильно использовать через SLF4J API для decoupling.

{} placeholders никогда конкатенация. Lazy evaluation экономит CPU когда level not enabled.

Уровни — DEBUG для development, INFO для важных events, WARN для подозрительного, ERROR для alert-worthy.

MDC для correlation ID и context. Thread-local через ThreadLocal — теряется на async boundaries без proper handling.

Structured JSON для ELK. logstash-logback-encoder library. Custom fields plus MDC.

AsyncAppender для performance. discardingThreshold=0 чтобы не терять. Increased queueSize для bursts.

ELK stack (Elasticsearch plus Logstash или Beats plus Kibana) стандарт для centralized logging. Retention через ILM.

Kubernetes stdout/stderr перехватывается kubelet. Filebeat DaemonSet шипает в ELK.

Никогда логировать секреты, PII, huge objects, every SQL в prod.

Никогда printStackTrace — используй log.error с exception. Реальный urok из КНП.

Correlation ID через MDC plus HTTP headers plus message headers для distributed traceability.

Метрики параллельно с логами. Observability = logs + metrics + traces.

Дальше — Kafka базовые concepts как основа для understanding streaming platform.
