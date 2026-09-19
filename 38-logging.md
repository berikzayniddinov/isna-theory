# 38. Логирование: SLF4J, Logback, ELK

Зачем логировать, чем, как правильно, как отправить в ELK.

---

## 1. Зачем логи

- **Debugging** — найти причину бага.
- **Audit** — «кто что сделал когда».
- **Monitoring** — «сколько ошибок за минуту».
- **Compliance** — регуляторные требования.
- **Observability** — вместе с метриками + traces = полная картина.

**Правило**: логи в проде — это НЕ println. Это структурированные события с уровнем, timestamp, контекстом.

---

## 2. Уровни

Стандарт (снизу вверх):

- **TRACE** — очень детально, редко в проде (метод-по-методу).
- **DEBUG** — детально для отладки (значения переменных, промежуточные шаги).
- **INFO** — важные события (старт сервиса, успешная обработка).
- **WARN** — что-то подозрительное но не ошибка (retry, deprecated API).
- **ERROR** — ошибка, требующая внимания.
- **FATAL** — критично, приложение падает (редко используется).

Приложение настраивается на уровень: `INFO` — покажет INFO, WARN, ERROR, скроет DEBUG и TRACE.

```yaml
logging:
  level:
    root: INFO
    kz.gov.kgd.isna: DEBUG
    org.hibernate.SQL: DEBUG
```

**Правило прода**: `INFO` root + `DEBUG` для своих пакетов при необходимости.

**Никогда** DEBUG на root в проде — залил ELK, ретеншн упал, файлы взорвались.

---

## 3. Экосистема логирования в Java

### 3.1 API vs implementation

**API (фасад)** — интерфейс который использует твой код:
- **SLF4J (Simple Logging Facade for Java)** — стандарт де-факто.
- **Commons Logging (JCL)** — старый, use JCL bridge для совместимости.
- **JBoss Logging** — фасад JBoss.

**Implementation** — что реально пишет:
- **Logback** — default в Spring Boot.
- **Log4j2** — альтернатива.
- **Log4j 1.x** — устарел, уязвимости, не использовать.
- **JUL (java.util.logging)** — встроенный в JDK, редко напрямую.

Схема:
```
Твой код → SLF4J → Logback → File / Console / ELK
```

### 3.2 Bridge

Если библиотеки используют разные фасады — bridges переадресуют в SLF4J:
- `jul-to-slf4j` — JUL → SLF4J.
- `jcl-over-slf4j` — Commons → SLF4J.
- `log4j-over-slf4j` — Log4j 1.x → SLF4J.

Spring Boot делает это автоматически.

### 3.3 Log4Shell (CVE-2021-44228)

Уязвимость Log4j 2.x до 2.17. RCE через `${jndi:...}` в log message. Обновиться немедленно.

Logback не подвержен (нет такой фичи).

---

## 4. Использование SLF4J

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

Через Lombok:
```java
@Slf4j
public class OrderService {
    // log — static final Logger автоматом
}
```

### 4.1 Плейсхолдеры `{}`

**ВСЕГДА использовать `{}`**, не конкатенацию:
```java
log.debug("Order: " + o.toString());   // ← toString вызывается ВСЕГДА, даже если DEBUG выключен

log.debug("Order: {}", o);              // ← toString только если DEBUG включён
```

Разница на прод-нагрузке — тысячи toString в секунду vs 0.

### 4.2 Exception как последний аргумент

```java
log.error("Failed to save {}", orderId, exception);
// exception распакуется как stacktrace, orderId подставится в {}
```

Не бросай в текст:
```java
log.error("Failed: " + exception.getMessage());   // теряется stacktrace!
```

### 4.3 isDebugEnabled guard

Для тяжёлых вычислений в log:
```java
if (log.isDebugEnabled()) {
    log.debug("Complex state: {}", heavyToString(state));
}
```

Иначе `heavyToString` вызовется даже если DEBUG выключен.

Для простых `{}` — guard не нужен.

---

## 5. Logback конфигурация

Файл `src/main/resources/logback-spring.xml`:

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

    <!-- Async для performance -->
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

### 5.1 Ключевые элементы

- **`<appender>`** — куда писать (Console, File, Kafka, HTTP).
- **`<encoder>`** — формат (pattern или JSON).
- **`<rollingPolicy>`** — ротация файлов (по времени / размеру).
- **`<logger>`** — уровень для конкретного пакета.
- **`<root>`** — по умолчанию для всего.

### 5.2 Pattern layout

`%d{yyyy-MM-dd HH:mm:ss.SSS}` — timestamp.
`%-5level` — уровень (left-aligned, 5 chars).
`%X{key}` — MDC (см. §7).
`%thread` — имя потока.
`%logger{36}` — имя класса (сокращено до 36 chars).
`%msg` — сообщение.
`%n` — newline.
`%ex` — exception.

---

## 6. Async appenders

Синхронный лог = каждая `log.info` блокирует поток пока запишется на диск. На high-load это заметно.

**AsyncAppender** — очередь, background thread пишет.

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE"/>
    <queueSize>512</queueSize>
    <discardingThreshold>0</discardingThreshold>   <!-- 0 = не терять; иначе % от queue -->
    <neverBlock>true</neverBlock>                   <!-- true = при переполнении дропать -->
</appender>
```

Кавет:
- При падении JVM — необлитая очередь теряется.
- `neverBlock=false` — при полной очереди log.info блокируется.

Правило для прода: async + `discardingThreshold=0` (не терять) + `queueSize` побольше.

---

## 7. MDC (Mapped Diagnostic Context)

Thread-local Map для контекста запроса. Отсюда:
- Correlation ID.
- User ID.
- Request ID.

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

В pattern:
```
%X{traceId} %X{userId}
```

Результат:
```
2026-09-05 10:15 [abc123, berik] INFO OrderService - Saving order 42
```

### 7.1 MDC + фильтры

В web приложении установить MDC в фильтре:
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

Все логи запроса будут с одним traceId.

### 7.2 MDC + async

MDC привязан к потоку. При `@Async` или virtual threads MDC **теряется**.

Решение:
- Spring `TaskDecorator` — копирует MDC при передаче задачи в executor.
- Micrometer Tracing — правильно пробрасывает через методы.

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

---

## 8. Structured logging (JSON) для ELK

Обычный log:
```
2026-09-05 10:15:30 INFO OrderService - Saving order 42
```

ELK может парсить, но не всегда точно. Лучше писать в JSON.

### 8.1 logstash-encoder

```gradle
implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
```

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

Результат:
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

ELK парсит легко и точно.

### 8.2 kv arguments

Ещё лучше — вынести значения в отдельные поля:
```java
log.info("Order saved", kv("orderId", o.getId()), kv("status", o.getStatus()));
```

Или через **StructuredArguments** (Logstash-encoder):
```java
log.info("Order saved: {}", value("orderId", o.getId()));
```

Плюс: можно фильтровать в Kibana `orderId:42` вместо regex по message.

---

## 9. ELK stack

**ELK** = Elasticsearch + Logstash + Kibana. Плюс Beats.

### 9.1 Elasticsearch

Хранит и индексирует логи. Full-text search + структурированные поля. См. `45-elasticsearch.md`.

### 9.2 Logstash

Pipeline для обработки логов:
- Читает из источников (Filebeat, Kafka, ...).
- Парсит / трансформирует.
- Пишет в Elasticsearch.

Тяжелый (Java). Часто заменяется на:

### 9.3 Beats

Легковесные шипперы:
- **Filebeat** — читает файлы, шлёт в Elasticsearch/Logstash.
- **Metricbeat** — метрики.
- **Auditbeat** — аудит.

### 9.4 Kibana

UI для Elasticsearch. Dashboards, поиск, alerts.

### 9.5 Типичная цепочка

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

Альтернатива:
```
Приложение → Logback appender → Kafka → Logstash → Elasticsearch
```

В ИСНА: свой ELK (`isna-elk`), логи через kubectl exec в под ES (memory `knp-prod-historical-logs-elk`).

### 9.6 Ретеншн

Логи не хранятся вечно (дорого):
- Hot indices — последние 7 дней, быстрый доступ.
- Warm indices — 30 дней, медленнее.
- Cold — 90+ дней, архив.
- Удалить старше.

**ILM (Index Lifecycle Management)** — автоматизирует.

---

## 10. Kubernetes и логи

### 10.1 stdout/stderr

Приложение в контейнере пишет в **stdout/stderr**. Docker/kubelet перехватывает, пишет в файлы на ноде (`/var/log/pods/...`).

```
kubectl logs <pod>            # текущий контейнер
kubectl logs <pod> --previous # предыдущий инстанс
```

### 10.2 Ротация kubelet

kubelet ротирует логи по размеру (обычно 10 MB) и держит N старых.

**Кавет**: старше суток обычно уже нет — memory `knp-prod-historical-logs-elk`. Надо через ELK.

### 10.3 Fluentd / Filebeat как DaemonSet

Один DaemonSet шиппера на каждой ноде читает pod-логи → шлёт в ELK.

---

## 11. Что НЕ логировать

### 11.1 Секреты

- Пароли, токены, ключи.
- Личные данные (PII).
- Кредитные карты.

Фильтрация:
```xml
<encoder>
    <pattern>%replace(%msg){'Bearer [A-Za-z0-9._-]+', 'Bearer ***'}</pattern>
</encoder>
```

Или через custom converter.

### 11.2 Огромные объекты

`log.info("Response: {}", hugeJson)` — 10 MB в лог. Смерть ELK.

Truncate:
```java
log.info("Response: {}", StringUtils.left(hugeJson, 500));
```

### 11.3 Every SQL в проде

`org.hibernate.SQL: DEBUG` — тысячи запросов в секунду в лог. Только для отладки.

---

## 12. Правила использования

### 12.1 Structured, не printStackTrace

**НИКОГДА**:
```java
} catch (Exception e) {
    e.printStackTrace();       // идёт в stderr, ELK не парсит как error
}
```

**Правильно**:
```java
} catch (Exception e) {
    log.error("Operation failed for id={}", orderId, e);
}
```

Реальный ИСНА-кейс `knp-fo-sync-notification-bugs` — printStackTrace → «ELK-слепая зона», ошибки не видны в мониторинге.

### 12.2 Уровни правильно

- **ERROR** — надо что-то делать (alert, инцидент).
- **WARN** — проверить когда есть время (deprecated API, retry).
- **INFO** — важные события пользователя.
- **DEBUG** — для отладки, off в проде.

Не пиши `log.error` на любую exception — если это ожидаемая (validation) → WARN или INFO.

### 12.3 Correlation ID везде

- Приходит в HTTP header `X-Request-Id` или генерируется.
- MDC + traceId в pattern.
- Пробрасывается в downstream (Feign interceptor).
- Пробрасывается в Rabbit/Kafka message headers.

Потом в Kibana ищешь `traceId:abc123` — видишь всю цепочку через все сервисы.

### 12.4 Не логировать в hot path

Метод дёргается 100000 раз в секунду → каждый log = disk write = смерть.

Используй метрики (counter, gauge) для таких мест, а лог только для аномалий.

### 12.5 Sampling

Иногда логировать только 1% запросов:
```java
if (ThreadLocalRandom.current().nextInt(100) == 0) {
    log.info("Sampled request: {}", req);
}
```

Или через Micrometer Tracing sampling.

---

## 13. Best-practice логер для сервиса

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

- INFO — важные события.
- DEBUG — детали.
- ERROR + stacktrace на неожиданное.
- Метрики параллельно с логами.

---

## 14. Реальные кейсы ИСНА

- **`knp-fo-sync-notification-bugs`**: `printStackTrace` → «ELK-слепая зона», не видно ошибок → 6 багов копились месяцами.
- **`knp-prod-historical-logs-elk`**: `kubectl logs` = только сегодня; исторические через ES (kubectl exec в pod ES).
- Fno328 регенерация — важен log каждой обработанной, чтобы отследить прогресс job.

Правила для команды:
1. Никакого `printStackTrace()`.
2. Structured logging (JSON).
3. MDC + traceId.
4. Метрики параллельно.
5. Уровни правильно (ERROR = alert-worthy).

---

## 15. Собесные вопросы

1. **SLF4J vs Logback?** — SLF4J фасад (API), Logback реализация.
2. **Уровни логов?** — TRACE, DEBUG, INFO, WARN, ERROR (+ FATAL редко).
3. **Почему `log.info("... " + x)` плохо?** — Конкатенация вычисляется всегда, даже если INFO выключен.
4. **Как правильно логировать exception?** — `log.error("msg with {}", context, exception)` — exception последним.
5. **Что такое MDC?** — Thread-local map для контекста (traceId, userId); попадает в log через `%X{key}`.
6. **AsyncAppender — за и против?** — Быстрее (не блокирует); при падении JVM теряет необлитую очередь.
7. **Что такое structured logging?** — Формат JSON с типизированными полями; легко парсится ELK.
8. **Что такое ELK?** — Elasticsearch + Logstash + Kibana; стек для логов.
9. **Разница Filebeat и Logstash?** — Filebeat легковесный шиппер; Logstash тяжёлый pipeline с трансформациями.
10. **Как правильно писать логи в K8s?** — В stdout/stderr; kubelet перехватит; DaemonSet Filebeat/Fluentd → ELK.
11. **Что такое Log4Shell?** — Log4j 2 RCE (CVE-2021-44228) через `${jndi:...}`; обновиться.
12. **Как маскировать секреты в логах?** — regex replace в encoder или custom converter.
13. **Correlation ID — зачем?** — Следовать за одним запросом через все сервисы (в MDC + HTTP header).
14. **printStackTrace — почему нельзя?** — Идёт в stderr; ELK не парсит как error; нет structured данных.
15. **Уровень логов в проде?** — INFO root; DEBUG только для конкретных пакетов при необходимости.

---

## Итог

- **SLF4J** = API; **Logback** = default реализация.
- **`{}` placeholders**, не конкатенация.
- **Уровни**: DEBUG (dev) / INFO (важные) / WARN (подозрительное) / ERROR (alert-worthy).
- **MDC** для correlation ID.
- **JSON structured** для ELK.
- **AsyncAppender** + async encoders для performance.
- **kubectl logs** = только сегодня; исторические через ELK.
- **НИКОГДА** printStackTrace, секреты в log, конкатенация.
- **Правильно** log + метрики + трейс = observability.

Следующий — `39-kafka-basics.md`.
