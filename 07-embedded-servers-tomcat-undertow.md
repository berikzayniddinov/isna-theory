# 07. Embedded HTTP-серверы: Tomcat, Jetty, Undertow, Netty

## Почему в микросервисах embedded

В классической enterprise Java (Java EE эры 2000-х) приложение упаковывалось в WAR (Web ARchive), деплоилось в отдельно установленный **application server** — Tomcat, WildFly, WebSphere, GlassFish. Сервер устанавливался администраторами, конфигурировался централизованно, обслуживал несколько приложений одновременно. Разработчик отдавал WAR, серверная команда его разворачивала.

Такая модель имела смысл когда приложений было мало и они были долгоживущими. Один сервер, несколько крупных приложений, стабильная конфигурация. Deployment через copying WAR файла в webapps директорию сервера. Все приложения делили JVM, connection pools, thread pools. Оптимально по ресурсам для monolithic архитектуры.

Микросервисная архитектура фундаментально изменила подход. У тебя не 3-5 крупных приложений, а 30-50 маленьких. Каждый должен деплоиться независимо, масштабироваться независимо, обновляться независимо. Общий application server становится узким местом: рестарт сервера положит все приложения одновременно, memory leak в одном сервисе затронет остальных, обновление сервера требует downtime для всех.

Ответ индустрии — **embedded server**. HTTP-сервер встраивается прямо в JAR приложения. Один артефакт, `java -jar app.jar` — и всё работает. Никакого отдельного application server. Каждый микросервис изолирован в своей JVM. Обновление одного — не затрагивает других. Идеально для контейнерной архитектуры с Docker и Kubernetes.

Spring Boot популяризировал embedded подход. Стандартные starters включают embedded server, приложение стартует само на своём порту. Простота, скорость деплоя, изоляция — то что нужно для микросервисов.

Есть несколько популярных embedded серверов, каждый со своими характеристиками. Tomcat как default выбор, Jetty как альтернатива, Undertow для low-memory сценариев, Netty как база для reactive стека. В этом файле разберём их подробно: как устроены, чем отличаются, когда какой выбирать.

## Servlet API: общий стандарт

Прежде чем говорить про конкретные серверы, нужно понимать общий стандарт, вокруг которого они построены — **Servlet API**.

Servlet API — стандарт Java EE (сейчас Jakarta EE) для обработки HTTP запросов на серверной стороне. Определяет интерфейс `HttpServlet`, который умеет обрабатывать HTTP через методы `doGet`, `doPost`, `doPut`, `doDelete`, работая с абстракциями `HttpServletRequest` и `HttpServletResponse`.

Любой сервер, реализующий Servlet API, называется **servlet container**. Он берёт на себя network слой (принимает TCP соединения, парсит HTTP), threading (пул потоков для обработки), lifecycle (registration servlets, filter chains), а разработчик работает с высокоуровневыми объектами Request и Response.

Простейший servlet:

```java
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
            throws IOException {
        resp.setContentType("text/html");
        resp.getWriter().write("<h1>Hello</h1>");
    }
}
```

Ты не думаешь про парсинг HTTP байтов, connection pool, socket management. Просто работаешь с request и response.

**Модель thread-per-request** — фундаментальная архитектурная характеристика классического Servlet API. Один поток обрабатывает один запрос от начала до конца. Приходит новый запрос — берётся свободный поток из пула. Пока запрос обрабатывается, поток занят. Когда обработка завершается, поток возвращается в пул.

Пул потоков имеет ограничение (у Tomcat по умолчанию 200). Пришло 201 одновременных запросов — 201-й ждёт свободного потока в очереди acceptCount.

Проблема этой модели проявляется на высоких RPS с I/O-bound операциями. Типичный REST endpoint: принять запрос, сделать SQL query (10-50 мс ожидания I/O), вызвать external API (100-500 мс), сформировать ответ. Пока ждёт I/O, поток заблокирован — CPU простаивает, но поток "занят" с точки зрения пула. При 200 потоках и I/O операциях по 500 мс — максимум 400 запросов в секунду, даже если CPU не загружен на 5%.

Плюс каждый поток стоит ресурсов. Stack size (по умолчанию 1 MB) × 200 потоков = 200 MB только на stacks. При тысячах потоков (что нужно для тысяч concurrent connections) — гигабайты только на инфраструктуру.

Async Servlet API (3.0+) частично решает это, позволяя освободить поток на время долгой операции. Spring MVC поддерживает через возврат `Callable` или `DeferredResult`. Non-blocking I/O (3.1+) позволяет писать в response без блокировки. Но настоящее решение для high-concurrency I/O — reactive стек (WebFlux + Netty), обсуждается ниже.

## Apache Tomcat: default выбор

**Apache Tomcat** — самый известный и старый servlet container, существует с 1999 года. Написан на чистой Java. Реализует полный Servlet API плюс JSP (Java Server Pages). Проверен временем на миллионах production deployments.

В Spring Boot стартер `spring-boot-starter-tomcat` автоматически подтягивается со `spring-boot-starter-web`. Именно поэтому Tomcat — сервер по умолчанию. Никаких дополнительных действий не нужно, чтобы получить Tomcat.

**Архитектура Tomcat**. Основные компоненты работают последовательно.

**Connector** — сетевой слой. Принимает TCP соединения на порту (обычно 8080), парсит HTTP запросы. Современный Tomcat использует NIO connector — accept соединений через `java.nio.channels.Selector`, non-blocking. Read HTTP request тоже non-blocking. Это позволяет один I/O thread обслуживать множество соединений на этапе приёма.

**Executor** — пул потоков для обработки запросов. По умолчанию 200 max threads, 10 min-spare. Каждый запрос попадает в один из этих потоков и обрабатывается синхронно от начала до конца. Настраивается через application.yml:

```yaml
server:
  tomcat:
    threads:
      max: 200
      min-spare: 10
    accept-count: 100          # очередь ожидающих потока
    connection-timeout: 20s
    max-connections: 8192
```

**Servlet container** — здесь работает DispatcherServlet Spring MVC. Диспатчер получает запросы от executor threads, находит handler methods по URL, вызывает controllers.

Full chain: HTTP запрос приходит → Connector парсит → Executor выделяет поток → DispatcherServlet роутит → Controller выполняет → response обратно через chain.

Полезные настройки Tomcat, часто дёргаемые в production.

```yaml
server:
  port: 8080
  tomcat:
    threads:
      max: 400                     # больше для high-throughput
    max-http-form-post-size: 10MB   # для больших POST
    max-swallow-size: 10MB          # size uploads
    connection-timeout: 30s          # HTTP request timeout после accept
    keep-alive-timeout: 60s          # HTTP keep-alive
    max-keep-alive-requests: 100     # запросов на одном connection
```

**Плюсы Tomcat**. Проверен временем, минимум сюрпризов. Огромное community, любой баг гуглится за секунды. Полная поддержка Servlet API и JSP. Стабильный, предсказуемый. Работает без магии — что настроил, то и получил.

**Минусы**. Больше памяти на поток по сравнению с Undertow. Не самый быстрый на очень высоких RPS. Тяжеловат для мini-микросервисов, где каждый мегабайт RAM важен.

Для типичного КНП микросервиса Tomcat — правильный выбор. Стабильный, знакомый, работает.

## Jetty: модульная альтернатива

**Eclipse Jetty** — альтернативный servlet container. Аналогичный по функциональности Tomcat, но с другой архитектурной философией. Модульный дизайн — включаешь только те компоненты, которые нужны. Легче стартует. Активно используется в embedded-сценариях, где Jetty встраивается в другие продукты (Hadoop, Kafka Connect, Google App Engine, ActiveMQ).

Для переключения на Jetty в Spring Boot — исключить Tomcat и подключить Jetty:

```gradle
implementation('org.springframework.boot:spring-boot-starter-web') {
    exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
}
implementation 'org.springframework.boot:spring-boot-starter-jetty'
```

Функционально Spring Boot приложение работает идентично — DispatcherServlet, controllers, все Spring MVC инфраструктура. Меняется только servlet container под капотом.

**Когда выбирать Jetty**. В embedded-сценариях, когда сервер встраивается в другой продукт (не только для стандартного backend). Если хочешь legче Tomcat, но без специфики Undertow. В некоторых специализированных экосистемах — стандартная деталь.

**Плюсы**. Быстрый старт. Модульная архитектура (можно подключать только нужное). Меньше memory footprint чем Tomcat в базовой конфигурации.

**Минусы**. Меньшее community чем Tomcat — реже гуглятся специфические проблемы. Некоторые Spring Boot фичи могут иметь квирки, специфичные для Jetty.

В КНП Jetty почти не встречается. Tomcat покрывает потребности, переключение без веских причин не оправдано.

## Undertow: легковесный async-friendly

**Undertow** — HTTP-сервер от команды JBoss/Red Hat. Написан с нуля, не является форком Tomcat или Jetty. Изначальная архитектурная цель — очень легковесный, async-friendly сервер. Часть **WildFly** application server как embedded HTTP модуль.

Undertow поддерживает Servlet API (через отдельный модуль), но родная модель — async, non-blocking через XNIO framework. Это делает его особенным среди других servlet containers.

**Архитектура Undertow** имеет два уровня.

**I/O threads** — маленький пул (по количеству CPU cores, обычно 4-8). Не блокируются, только читают и пишут в сокеты через NIO. Обрабатывают тысячи соединений на этапе network I/O.

**Worker threads** — большой пул для блокирующих операций. Servlet API — синхронный, требует thread-per-request. Когда Undertow получает HTTP запрос, он передаёт его worker thread. Worker выполняет весь Servlet chain (включая DispatcherServlet и controllers) синхронно. По окончании возвращается в пул.

Такое разделение позволяет эффективно использовать ресурсы. I/O threads не блокируются, обслуживают много connections. Worker threads только для actual work. Меньше overall потоков — меньше memory на stacks.

Переключение на Undertow:

```gradle
implementation('org.springframework.boot:spring-boot-starter-web') {
    exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
}
implementation 'org.springframework.boot:spring-boot-starter-undertow'
```

Настройка:

```yaml
server:
  undertow:
    threads:
      io: 4                    # I/O потоки, обычно = число CPU
      worker: 200              # blocking worker потоки
    buffer-size: 1024          # размер buffer'а
    direct-buffers: true        # direct memory для I/O
```

**Плюсы Undertow**. Меньше памяти чем Tomcat. Async-friendly для мест где нужна non-blocking обработка. Быстрее в микро-бенчмарках на высоких RPS.

**Минусы**. Меньшее community. Некоторые Servlet-фильтры или расширения могут работать по-другому. WebSocket implementation специфична.

**Когда выбирать Undertow**. Если реально нужно снизить memory footprint (много инстансов на одной ноде, IoT edge deployment). Если планируется async-heavy архитектура. Если профилирование показало bottleneck в Tomcat threading (редкость).

**Реалистично для КНП**. Стоит ли переключаться? Обычно нет. Разница в производительности на средних нагрузках — процент-полтора. Tomcat стабильнее в понимании команды. Undertow — правильный выбор только когда есть конкретная measurable причина.

## Netty: не servlet, база reactive

**Netty** — асинхронный event-driven networking framework. Кардинально отличается от Tomcat/Jetty/Undertow — это **не** servlet container. Работает на уровне TCP/UDP + HTTP, без Servlet API.

Netty создан для очень высоких RPS с async I/O. Используется в множестве high-performance систем: Cassandra, Elasticsearch, gRPC internally, Vert.x. В Spring мире — фундамент для **WebFlux** reactive stack.

**Модель Netty** — event loop. Небольшой пул потоков (обычно = CPU cores), каждый обрабатывает много соединений через NIO Selector.

Один event loop обслуживает список соединений. Приходит событие "пришли байты" на соединение — event loop обрабатывает. Приходит событие "БД ответила" — event loop обрабатывает. Один поток обслуживает тысячи соединений, потому что никогда не блокируется. Все I/O операции — async.

Ключевое правило: **не блокировать event loop поток**. Если внутри handler'а сделать synchronous DB call — весь event loop встанет, все connections на нём перестанут обрабатываться. Приходится всё делать async — DB через async driver (R2DBC), HTTP calls через async client, никаких synchronous операций.

**Reactive Streams** — стандартная модель для async композиции. `Publisher` (кто-то производит данные) → `Subscriber` (кто-то читает) с **backpressure** (subscriber говорит "стоп, я не успеваю"). В Spring реализация — **Reactor** (Mono, Flux).

Сравним обычный Spring MVC (Tomcat) с WebFlux (Netty).

Обычный MVC:

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    User u = userRepo.findById(id).orElseThrow();
    Address a = addressService.get(u.getAddressId());
    return u.withAddress(a);
}
```

Reactive (WebFlux):

```java
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable Long id) {
    return userRepo.findById(id)
        .flatMap(u -> addressService.get(u.getAddressId())
            .map(u::withAddress));
}
```

Никакого блокирующего кода. Цепочка async операций через `flatMap`, `map`. Event loop не блокируется — работает параллельно с тысячами других запросов.

**Когда выбирать WebFlux + Netty**. Очень много одновременных соединений (WebSocket, SSE, long polling). Massive I/O с большим fan-out (5+ параллельных API вызовов на один запрос). Streaming (файлы, видео).

**Когда не стоит**. Обычные CRUD с JDBC — JDBC блокирует, нивелирует преимущество. Есть R2DBC как reactive JDBC, но экосистема беднее. Команда не знакома с reactive программированием (кривая обучения крутая). Требует рефакторинга всей цепочки на async.

В КНП — в основном MVC + Tomcat. Reactive не используется на серьёзных модулях. Слишком много legacy Hibernate кода, слишком велика инженерная стоимость перехода.

## Сравнительная характеристика

Итог по четырём серверам.

**Tomcat**: servlet container, thread-per-request, default в Spring Boot, огромное community, стабильный. Средняя memory footprint, средняя производительность. Правильный выбор для большинства случаев.

**Jetty**: servlet container, thread-per-request, модульный, легче стартует. Меньше community чем Tomcat, но полностью совместим. Ниша — embedded в другие продукты.

**Undertow**: servlet container + native async, I/O + worker threads. Меньше памяти, лучше на очень высоких RPS. Ниша — где critical memory или async-heavy архитектура.

**Netty**: не servlet, event loop, база для reactive стека (WebFlux). Кардинально другая модель программирования. Ниша — high-concurrency I/O, streaming, где traditional MVC не тянет.

Выбор в Spring Boot — через включение/exclude в build.gradle. Все они через тот же Spring MVC (кроме Netty с WebFlux), контроллеры пишутся одинаково.

## Как узнать какой сервер работает

При старте Spring Boot логирует:

```
Tomcat initialized with port(s): 8080 (http)
```

Или:

```
Undertow started on port 8080
```

Через код:

```java
@Bean
public CommandLineRunner runner(WebServerApplicationContext ctx) {
    return args -> {
        System.out.println("Server: " + ctx.getWebServer().getClass().getSimpleName());
    };
}
```

Actuator `/actuator/info` (если сконфигурировано) может включать эту информацию.

## Настройки, часто дёргаемые в production

**Порт и context path**:

```yaml
server:
  port: 8080
  address: 0.0.0.0                # bind interface
  servlet:
    context-path: /api            # все URL с префиксом /api
```

`address: 0.0.0.0` — listen на всех интерфейсах. `127.0.0.1` — только localhost (полезно за reverse proxy).

**Timeouts** — контроль долгих запросов и keep-alive:

```yaml
server:
  tomcat:
    connection-timeout: 30s           # ждём HTTP-запрос после accept
    keep-alive-timeout: 60s            # держим соединение открытым
    max-keep-alive-requests: 100
```

Правильные timeouts важны для стабильности. Слишком короткие — обрывают долгие запросы. Слишком длинные — забиваются connections от медленных клиентов.

**Размер запроса** — контроль memory:

```yaml
server:
  tomcat:
    max-http-form-post-size: 10MB
    max-swallow-size: 10MB
  max-http-header-size: 32KB
spring:
  servlet:
    multipart:
      max-file-size: 50MB
      max-request-size: 50MB
```

Ограничение защищает от DoS через огромные запросы.

**SSL** — HTTPS terminate на приложении:

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
```

Обычно в микросервисной архитектуре SSL терминируется на ingress/nginx уровне, приложение получает plain HTTP внутри кластера. Меньше сложности, единая точка управления сертификатами.

**GZIP compression** — экономия network:

```yaml
server:
  compression:
    enabled: true
    min-response-size: 1024
    mime-types: application/json,text/html,text/xml
```

Ответы больше 1 KB сжимаются. Хорошо для JSON API — сжимается легко, экономит bandwidth.

## DispatcherServlet и servlet chain

Внутри embedded server Spring Boot регистрирует **DispatcherServlet** — единственный servlet, обрабатывающий все URL приложения. Он же — front controller Spring MVC.

Полный chain HTTP запроса:

```
HTTP request
  → Tomcat (accept, parse HTTP)
  → Servlet chain (filters — Spring Security, CORS, logging)
  → DispatcherServlet (Spring MVC front controller)
    → HandlerMapping (найти @Controller.method по URL)
    → HandlerInterceptor pre (Spring MVC уровня)
    → Controller.method (твой код)
    → HandlerInterceptor post
    → HttpMessageConverter (Jackson serialize)
  → Servlet chain (filters return path)
  → Tomcat (write HTTP response)
```

**Filters** — уровень servlet container. Работают до DispatcherServlet. Используются для cross-cutting concerns на низком уровне: authentication, CORS, logging, encoding.

**Interceptors** — уровень Spring MVC. Работают внутри DispatcherServlet. Более удобно для Spring-specific логики: authorization, application-level logging.

Разница практическая — filters увидят все запросы (включая static resources), interceptors только те что попали в DispatcherServlet.

## Заключение

Embedded HTTP-серверы — стандарт для микросервисной архитектуры. Приложение упаковано с сервером в один JAR, один процесс, один порт, полная изоляция от других сервисов. Идеально для Docker и Kubernetes.

Servlet API — общий стандарт для четырёх classic серверов. Thread-per-request модель — синхронная, простая для разработки, но ограниченная на высоких RPS с I/O.

Tomcat — default в Spring Boot, проверен временем, стабильный, большое community. Правильный выбор в 90% случаев. Средняя memory footprint, средняя производительность.

Jetty — модульный, легковесный старт. Ниша — embedded в другие продукты. В типичном Spring Boot приложении даёт мало преимуществ над Tomcat.

Undertow — легковесный, async-friendly через XNIO. Меньше memory, лучше на high RPS. Выбирать когда есть конкретная measurable причина — обычно нет.

Netty — не servlet, event loop, база для reactive стека WebFlux. Кардинально другая модель. Выбирать когда really нужна high-concurrency non-blocking архитектура. Для типичного CRUD с JDBC — не оправдано.

Для КНП контекста — Tomcat как default, работает надёжно. Изменять только когда есть реальная причина, не по моде или в надежде на magic performance improvement. Всегда сначала профилирование, потом решение.

Знание отличий важно для собеседования (типичный вопрос: "чем Tomcat отличается от Undertow?") и понимания архитектурных альтернатив. Практически — уверенное владение Tomcat покрывает большинство рабочих задач.

Дальше — углублённое понимание того, что происходит при старте Spring Boot приложения, где embedded server стартует в общей цепочке. Это тема следующего файла — application startup chain.
