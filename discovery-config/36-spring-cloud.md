# 36. Spring Cloud: инфраструктурный набор для микросервисной архитектуры

## Зачем нужен Spring Cloud

Разработчик, пишущий монолит на Spring Boot, живёт в удобном мире. База данных одна, конфиг в application.yml, вызовы к сторонним сервисам через RestTemplate или Feign с указанием конкретного URL. Всё просто. Но когда монолит распиливается на десятки микросервисов, простые вопросы становятся сложными. Куда обращаться за данными пользователя — какой URL у user-service? Что делать если user-service временно недоступен — падать вместе с ним или как-то деградировать? Как обеспечить чтобы конфиги 30 сервисов не разъехались между окружениями? Как отследить один запрос проходящий через пять сервисов чтобы понять где именно возникла проблема?

Каждый из этих вопросов имеет свой класс решений. Service discovery — где найти сервис. Circuit breaker — как защитить от каскадных отказов. Configuration server — как централизовать конфиги. Distributed tracing — как проследить запрос через границы сервисов. API gateway — как разграничить внешний мир от внутренней топологии. Client-side load balancing — как распределить нагрузку между несколькими инстансами одного сервиса. Все эти решения реализованы через различные technology choices в индустрии, но unified в один umbrella-project — Spring Cloud.

Разница между разработчиком «использующим Spring Cloud» и «понимающим Spring Cloud» проявляется когда что-то идёт не так. Первый следует туториалу и надеется. Второй знает что Netflix Ribbon пропущен и заменён на Spring Cloud LoadBalancer с существенно другой семантикой (например case-sensitivity в service names), знает почему Zuul 1 не совместим с Java 17+ и нужна миграция на Spring Cloud Gateway с reactive stack, знает как Discovery Client'ы кэшируют service instances и почему свежедобавленный сервис не сразу виден остальным. Знает конкретную interoperability версий — Spring Boot 2.6 работает только с Spring Cloud 2021.0.x, попытка использовать 2022.0.x приведёт к странным NoClassDefFoundError в runtime.

В этом файле мы разберём этот landscape. Spring Cloud как umbrella-проект: что такое release train, почему совместимость версий критична. Три категории компонентов: живые (активно развиваются), deprecated Netflix stack (сохранены для legacy но не для новых проектов), attractive-but-rare (interesting concepts но редко нужны в реальных проектах). Detailed разбор каждого живого компонента — Consul discovery, OpenFeign, Spring Cloud LoadBalancer, Spring Cloud Gateway, Resilience4j, Micrometer Tracing. Что реально используется в КНП и почему определённые выборы сделаны так а не иначе.

## Umbrella и release train

Spring Cloud это не одна библиотека а набор проектов. Spring Cloud Config, Spring Cloud Consul, Spring Cloud OpenFeign, Spring Cloud LoadBalancer, Spring Cloud Gateway, Spring Cloud Stream, Spring Cloud Bus, Spring Cloud Sleuth (deprecated) — каждый отдельный Maven/Gradle module со своими versioning и dependencies. Использовать их все одновременно необязательно — типичный микросервис КНП использует Consul, OpenFeign, LoadBalancer, но не Config, Stream, или Bus.

Release train это concept для координации versioning всех этих subprojects. Каждый release train имеет alphabetical name (Angel, Brixton, Camden, Dalston, Edgware, Finchley, Greenwich, Hoxton, Ilford, Jubilee, Kilburn, Leyton) плюс sequence number. Все subprojects в одном train tested to work together. Внутри одного train mismatch между subprojects обычно not допустим.

Совместимость с Spring Boot критична и строго ограничена. Spring Boot 2.2 требует Spring Cloud Hoxton. Spring Boot 2.6 — Spring Cloud 2021.0. Spring Boot 3.0 — Spring Cloud 2022.0. Попытка использовать неcompatible пару обычно даёт странные runtime errors — ClassNotFoundException, NoSuchMethodError, mysterious autoconfiguration failures. Причина — Spring Cloud внутренне использует Spring Boot internals (autoconfiguration mechanisms, ConditionalOn annotations), и эти API меняются между major versions Spring Boot.

Практическое правило — при апгрейде Spring Boot всегда проверять и обновлять Spring Cloud version согласно compatibility matrix. При апгрейде Spring Cloud — проверить что Spring Boot version подходит. Никаких «downgrade Spring Boot а Cloud оставить» или обратно — recipe для disaster.

В КНП это правило проявляется буквально. Master branch использует Spring Boot 2.2 плюс Spring Cloud Hoxton.SR3. Все модули на этой ветке живут в этом ограничении. Master-21 branch мигрирует на Java 21 плюс Spring Boot 3 плюс Spring Cloud 2022.0. Все модули этой ветки должны быть на consistent versions. Cross-branch merges (например если фича сделана в master и мержится в master-21) требуют careful attention к Spring Cloud API differences.

## Три категории компонентов

Spring Cloud эволюционировал через несколько эр. Изначально heavily leveraged Netflix OSS stack — Eureka, Ribbon, Hystrix, Zuul. Netflix открыл эти компоненты open source в 2013-2015 годах когда они были cutting edge. Но в 2018-2020 годах Netflix постепенно свёл их к maintenance mode — новых features не добавляется, только critical bug fixes.

Первая категория — живые компоненты которые активно развиваются и рекомендованы для новых проектов. Spring Cloud Consul для service discovery. Spring Cloud OpenFeign для declarative HTTP clients. Spring Cloud LoadBalancer как current-generation client-side balancer. Spring Cloud Gateway как reactive gateway. Resilience4j как current circuit breaker library. Micrometer Tracing как distributed tracing. Все эти проекты имеют active development, regular releases, strong community support.

Вторая категория — deprecated Netflix stack. Netflix Eureka для service discovery — не развивается. Netflix Ribbon для client-side balancing — deprecated, users migrate to Spring Cloud LoadBalancer. Netflix Hystrix для circuit breaker — maintenance mode с 2018, миграция на Resilience4j. Netflix Zuul 1 для gateway — не совместим с Java 17+, миграция на Spring Cloud Gateway. Netflix Archaius для dynamic configuration — рекомендуется Spring Cloud Config.

Spring Cloud поддерживает эти компоненты в legacy branches для совместимости с existing systems. Но новые проекты не должны использовать. Существующие проекты обычно вынуждены migrate когда апгрейд Java или Spring Boot делает старые компоненты incompatible. В КНП именно это происходит — isna-knp-gateway остаётся на Java 11 потому что использует Zuul 1, миграция на Java 21 требует переход на Spring Cloud Gateway.

Третья категория — attractive-but-rare компоненты. Spring Cloud Config сентовый conceptual — централизованный git-backed конфиг с dynamic refresh. Practically редко используется потому что в K8s environments ConfigMap+Secret plus environment variables проще. Spring Cloud Stream — declarative messaging поверх RabbitMQ/Kafka. Elegant но abstraction leaky — производственные troubleshoot situations всё равно требуют knowledge underlying broker. Обычно прямой Spring AMQP или Kafka client проще. Spring Cloud Bus — event bus для system-wide events (например refresh конфига через message). В K8s rollout restart deployment проще. Spring Cloud Contract — consumer-driven contract testing. Elegant но требует discipline не available в большинстве команд.

В КНП практически используется только first category. Consul discovery, Feign, LoadBalancer, Circuit Breaker точечно. Gateway на Zuul 1 (legacy) с плановой миграцией на Spring Cloud Gateway. Attractive-but-rare категория не используется — принципиально проще без них.

## Service Discovery: Consul как основа

Service discovery решает проблему «где найти сервис по имени». Сервис А хочет вызвать сервис Б — вместо hardcoded URL «http://users.internal:8080» А спрашивает у Discovery «какие instances Users доступны?» и получает список текущих alive instances.

Как это работает механически. При старте каждый Spring Boot микросервис регистрируется в Consul через spring-cloud-consul-discovery starter. Регистрация включает service name (isnaknpuser), IP address (обычно pod IP в K8s), port, plus health check URL. Consul сохраняет эту регистрацию в своей data structure (KV store через Raft consensus). Периодически (каждые 10 seconds) Consul вызывает health check URL — если возвращает 200, instance считается healthy, если нет — marked unhealthy. Автоматически unhealthy instances убираются из discovery results через несколько failed checks.

Client discovery работает так. Когда сервис А хочет позвать service B — spring-cloud-loadbalancer запрашивает у Consul список healthy instances of B. Consul возвращает список (может быть кэшированный локально). LoadBalancer выбирает один instance согласно algorithm (round-robin по default). Actual HTTP request идёт напрямую к выбранному instance IP:port — Consul только предоставил discovery, не participating в actual data path.

Detailed Consul обсуждается в файле 11. Здесь важно отметить как Spring Cloud interfaces с Consul. Spring-cloud-consul-discovery starter добавляет ConsulDiscoveryClient bean который implements DiscoveryClient interface. LoadBalancer uses ServiceInstanceListSupplier which internally uses DiscoveryClient. Feign integrates с LoadBalancer через LoadBalancerInterceptor — Feign target «lb://isnaknpuser» resolves к service name, LoadBalancer picks instance, Feign делает call.

Health check тонкость. По default Consul-registered service exposes /actuator/health как health check endpoint. Spring Boot Actuator health включает многие indicators — БД, RabbitMQ, Redis, disk space. Если любой indicator DOWN — general health DOWN — Consul marks unhealthy. При проблемах с одной dependency (например Redis) весь сервис может стать unavailable из discovery perspective, даже если он способен обрабатывать requests не требующих Redis. Practical mitigation — configurate health indicators explicitly, use readinessState indicator instead of general health для discovery purposes.

## OpenFeign: декларативные HTTP clients

OpenFeign это declarative REST client library. Вместо императивного кода вызывающего RestTemplate или WebClient, разработчик описывает REST API как Java interface с annotations, и Feign генерирует implementation:
```java
@FeignClient(name = "isnaKnpUser")
public interface UserClient {
    @GetMapping("/api/users/{id}")
    UserDto getById(@PathVariable Long id);
    
    @PostMapping("/api/users")
    UserDto create(@RequestBody UserCreateRequest req);
}
```

Injection работает как обычно:
```java
@Autowired UserClient userClient;
UserDto u = userClient.getById(42L);
```

Механика под капотом сложнее чем кажется. При старте Spring @EnableFeignClients scans classes с @FeignClient. Для каждого interface создаётся FeignClient proxy через JDK dynamic proxy. Proxy implements указанный interface. Method invocation intercepted — Feign inspects annotations (@GetMapping, @PathVariable, etc), builds HTTP request с proper URL, headers, body serialization. Отправляет через underlying HTTP client (Apache HttpComponents по default, но конфигурируется). Deserializes response через Jackson. Возвращает результат.

Integration со Spring Cloud LoadBalancer автоматическая. Feign target «name = isnaKnpUser» resolves как service name не URL. Perinstead LoadBalancer picks instance from Consul-registered list. Actual URL built dynamically per-call. Из view of application code это transparent.

Retry, timeout, error handling конфигурируется через various mechanisms. Default retry disabled — Feign fails fast on any HTTP error. Explicit retryer bean можно добавить для retry logic. Feign имеет timeout settings (connectTimeout, readTimeout) конфигурируемые per client или globally. ErrorDecoder для custom translation HTTP errors в specific exceptions.

Circuit Breaker integration с Resilience4j через spring-cloud-starter-circuitbreaker-resilience4j. При enabled — Feign calls wrapped в CircuitBreaker automatically. Failed calls counted, thresholds trigger open state, subsequent calls fail-fast с fallback if configured.

Practical caveats. Feign уходит в maintenance mode — не отвергнут но не активно развивается. Spring Cloud shifts focus к WebClient (reactive) и RestClient (blocking, Spring 6+). Для существующих проектов Feign still fine — миграция не urgent. Для новых проектов возможно RestClient более future-proof choice.

Serialization gotchas. Feign по default uses Jackson через SpringDecoder. LocalDateTime и другие Java time types требуют proper Jackson configuration — обычно ObjectMapper.registerModule(new JavaTimeModule()) через Spring auto-config. Отсутствие configuration приводит к ошибкам serialization/deserialization на HTTP calls.

## Spring Cloud LoadBalancer: замена Ribbon

Netflix Ribbon был classic client-side load balancer десятилетие назад. Ribbon интегрировался с Feign, RestTemplate, Eureka discovery. Netflix прекратил его развитие. Spring Cloud LoadBalancer это preferred replacement.

Semantic difference критична. Ribbon был case-insensitive при resolving service names — «IsnaKnpUser», «isnaknpuser», «ISNA_KNP_USER» все resolved to same registered service. Spring Cloud LoadBalancer строгий case-sensitive. Migration Ribbon → LoadBalancer в existing codebase может expose bugs где service references case-inconsistent.

Реальный кейс из КНП memory knp-fo-consul-lb-mr1223-latent-mine. При миграции с Ribbon на Spring Cloud LoadBalancer часть Feign clients использовала camelCase service names («isnaKnpUser») в @FeignClient(name = ...) annotation. Consul registered services как lowercase («isnaknpuser»). Ribbon lookup workedдержался благодаря case-insensitive comparison. LoadBalancer failed lookup — service not found — Feign call fails with 500 error. Все places где service names могли differ должны быть carefully audited и unified.

Механика LoadBalancer. При call на service name — LoadBalancer использует ServiceInstanceListSupplier для получения текущего списка available instances. Список обычно fetched from Consul через DiscoveryClient. Instances могут быть filtered — например через ZoneAffinityFilter for preferring instances в same availability zone. Затем ReactorLoadBalancer picks one instance по algorithm.

Default algorithm RoundRobinLoadBalancer — простая round-robin distribution requests между instances. RandomLoadBalancer alternative для non-sequential distribution. Custom algorithms реализуются через ReactorLoadBalancer interface — например weighted based на capacity или health metrics.

Кэширование instances важно для performance. Fetching from Consul на каждый request — network round-trip. Locally cached list refreshed periodically. По default refresh каждые 15-30 seconds. Между refreshes changes в registered services не visible — new service just registered в Consul может быть invisible for a period. Практически not проблема для normal operations но important для understanding behavior at scale.

## Circuit Breaker с Resilience4j

Circuit breaker решает cascading failure problem. Downstream service starts failing — upstream service continues calling — retries pile up — resources exhaust — upstream service tоже становится unresponsive — clients upstream тоже начинают fail — весь system down.

Idea — detect downstream problem quickly, stop calling для period, wait for recovery, gradually resume. Analogous к electrical circuit breaker — при overcurrent разомкается предотвращая damage.

Три состояния circuit breaker. CLOSED — normal operation, all requests pass through к downstream. OPEN — detected sustained failures, all requests immediately fail с error/fallback без attempt call downstream. HALF_OPEN — after cooldown period, allow limited number of test requests to check если downstream recovered.

Transition CLOSED → OPEN triggered когда failure rate exceeds threshold within sliding window. Например «50% failure rate за last 20 requests → OPEN for 30 seconds». Configuration через properties или @CircuitBreaker annotation:
```java
@CircuitBreaker(name = "isnaKnpUser", fallbackMethod = "fallback")
public UserDto getUser(Long id) {
    return userClient.getById(id);
}

public UserDto fallback(Long id, Throwable t) {
    return UserDto.EMPTY;
}
```

Fallback method имеет same signature plus Throwable как last parameter. Вызывается когда circuit OPEN или actual call fails. Returns fallback value обеспечивая graceful degradation.

Resilience4j implementation через AOP proxy подобный @Transactional. При @CircuitBreaker annotated method call — proxy checks circuit state. CLOSED — actual call, result или exception recorded в metrics. OPEN — immediate return fallback. HALF_OPEN — limited test calls, based on results transition either back to CLOSED or forward to OPEN.

Sliding window это внутренняя structure tracking recent call results. Two types available. Count-based — considers last N calls (например 20). Time-based — considers calls в last M seconds. Threshold calculated based on window contents.

Metrics exposed via Micrometer для monitoring. resilience4j.circuitbreaker.state gauge showing текущее состояние. resilience4j.circuitbreaker.calls counter partitioned by outcome (successful, failed, ignored, not_permitted). Alerts на sustained OPEN state critical для operational awareness.

Practical rules для usage. Не любой downstream call нуждается в circuit breaker — overhead и complexity. Полезно для критичных external services или потенциально slow downstream. Fallback method должна быть fast и safe — не calling other services которые тоже могут fail.

## Spring Cloud Gateway

Netflix Zuul 1 был popular gateway solution — thread-per-request model, blocking IO. Zuul 2 released но никогда не supported by Spring Cloud. Spring Cloud Gateway — modern replacement built on Spring WebFlux plus Netty — reactive, non-blocking, higher throughput per resource.

Gateway это entry point для внешнего трафика. Все external requests hit gateway first. Gateway routes to appropriate internal service based on rules. Along the way applies filters — authentication, rate limiting, header manipulation, request logging.

Configuration через yml declarative:
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: knp-api
          uri: lb://isna-knp
          predicates:
            - Path=/api/knp/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Source, gateway
```

Predicates определяют когда route applies. Path predicate matches URL patterns. Method matches HTTP methods. Header matches specific headers. Query matches query parameters. Multiple predicates combined через AND — все должны match.

Filters transform request/response. Pre-filters run before forwarding to downstream. Post-filters run after response received. Built-in filters include StripPrefix (remove path prefixes), AddRequestHeader/AddResponseHeader (inject headers), RewritePath (URL rewriting), Retry (automatic retries), CircuitBreaker (integration with Resilience4j), RequestRateLimiter (rate limiting), TokenRelay (OAuth2 token propagation).

Custom filters через Java классы implementing GatewayFilter interface. Мощно для business-specific transformations — например custom authentication schemes, complex routing based на content, tenant identification.

Reactive nature imposes некоторые constraints. All Java code в custom filters должно быть non-blocking. Blocking calls (например synchronous JDBC, synchronous HTTP) в reactive stack приводят к thread starvation. Debugging reactive code more complex — stack traces less informative, error propagation через reactive operators.

В КНП isna-knp-gateway использует Zuul 1 не Spring Cloud Gateway. Legacy решение сохранившееся с ранних этапов проекта. Memory knp-gateway-no-java21 отмечает что gateway остаётся на Java 11 потому что Zuul 1 не совместим с Java 17+. Миграция на Spring Cloud Gateway необходима для дальнейшего upgrade path но complex — все routing rules plus custom filters нужно переписать в reactive style. Постоянно в planning не выполнено пока.

## Distributed Tracing: Sleuth и Micrometer

Distributed tracing решает problem tracing единого request через несколько services. Пользователь делает HTTP request → gateway → service A → service A calls service B → service B queries database. Если что-то slow или fails — как определить где именно?

Concept trace and span. Trace это все work поставленная одному logical request. Trace ID uniquely identifies trace. Span это piece of work within trace — HTTP request, database query, external call. Каждый span has unique span ID plus optional parent span ID forming tree.

Как propagates через service boundaries. Service A doing outgoing HTTP call inserts headers — X-B3-TraceId, X-B3-SpanId, X-B3-ParentSpanId. Service B receiving request extracts these headers и creates new span as child of received span. Continued down the chain — trace tree builds up.

Sleuth (deprecated with Boot 3) был Spring's implementation. Provided integration с popular tracing backends — Zipkin, Jaeger. Automatically instrumented common integrations — Spring MVC controllers, RestTemplate, Feign, RabbitMQ, JDBC.

Micrometer Tracing это replacement в Boot 3+. Same concept но based на Micrometer's observation abstraction. Same integrations available. Bridge adapters для both Zipkin and OpenTelemetry backends.

MDC integration. Trace and span IDs automatically populated в SLF4J MDC. Logs включают them через pattern:
```
2026-09-05 10:15:30 [traceId=abc123,spanId=def456] INFO OrderService - Order created
```

При правильно configured centralized logging (ELK), searching for traceId shows все logs всей trace across services. Powerful debugging capability.

Sampling important. Instrumenting каждый request creates overhead. High-volume services often sample — например 10% of requests fully traced. Trade-off между observability и performance. Configuration через management.tracing.sampling.probability.

В КНП distributed tracing partially used. Full Zipkin/Jaeger deployment absent но traceId propagation через MDC and correlated logs в ELK provide most of the value. Full-fledged distributed tracing solutions пока не implemented.

## Полная production конфигурация

Собранная воедино dependencies для типичного микросервиса КНП:
```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.cloud:spring-cloud-starter-consul-discovery'
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer'
    implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
    implementation 'io.micrometer:micrometer-tracing-bridge-brave'
    implementation 'io.zipkin.reporter2:zipkin-reporter-brave'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    runtimeOnly 'org.postgresql:postgresql'
}
```

Настройки в application.yml объединяют многие concerns:
```yaml
spring:
  application:
    name: isna-knp
  cloud:
    consul:
      host: consul.isna
      discovery:
        health-check-path: /actuator/health
        query-passing: true
    openfeign:
      circuitbreaker.enabled: true
  security:
    oauth2:
      resourceserver.jwt.issuer-uri: https://keycloak.isna/realms/knp
management:
  endpoints.web.exposure.include: health,metrics,prometheus,info
  tracing.sampling.probability: 0.1
```

Consul discovery configured. Feign integrated с circuit breaker. OAuth2 resource server для JWT. Actuator plus metrics exposed для Prometheus scraping. Tracing sampling at 10%.

## Итоги

Spring Cloud это umbrella для микросервисной infrastructure. Не single library — collection of specialized subprojects. Использование selective — pick what needed for specific project.

Release train coordination critical. Spring Boot и Spring Cloud versions must match — incompatible combinations produce mysterious errors. В КНП master branch на Boot 2.2 + Cloud Hoxton, master-21 branch on Boot 3 + Cloud 2022.

Три категории компонентов. Живые — Consul, OpenFeign, LoadBalancer, Gateway, Resilience4j, Micrometer Tracing. Deprecated Netflix stack — Eureka, Ribbon, Hystrix, Zuul 1. Attractive-but-rare — Config, Stream, Bus, Contract.

Service Discovery через Consul в КНП. Каждый микросервис self-registers, health checks периодически, discovery client fetches instance lists. Case-sensitive service names in LoadBalancer (unlike old Ribbon) — miграция может обнаружить latent bugs.

OpenFeign declarative HTTP clients. Interface plus annotations plus generated proxy. Integration с LoadBalancer transparent. Serialization gotchas around Java time types.

Spring Cloud LoadBalancer replaces Ribbon. Case-sensitive resolution. Round-robin default algorithm. Cached instance lists refreshed периодически.

Circuit Breaker via Resilience4j. Three states CLOSED/OPEN/HALF_OPEN. Threshold-based transitions. Fallback methods для graceful degradation. AOP proxy implementation similar to @Transactional.

Spring Cloud Gateway replaces Zuul 1. Reactive stack на WebFlux plus Netty. Declarative routing rules plus filters. В КНП still Zuul 1 legacy — миграция pending.

Distributed Tracing через Micrometer Tracing (Sleuth deprecated). Trace and span propagation через HTTP headers. MDC integration для traceId в logs. Sampling для managing overhead. В КНП partial usage — traceId в logs but not full-fledged Zipkin/Jaeger.

Дальше — nodes как universal concept в distributed systems, разные interpretations в разных technologies.
