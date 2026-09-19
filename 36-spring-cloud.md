# 36. Spring Cloud обзор

## Что такое Spring Cloud

Spring Cloud это набор проектов на основе Spring Boot для решения типовых задач в микросервисной архитектуре. Не единая библиотека а «зонтик» из множества специализированных компонентов каждый со своей ролью. Service discovery, externalized configuration, client-side load balancing, circuit breakers, distributed tracing, API gateway, messaging — все эти инфраструктурные concerns покрываются отдельными Spring Cloud подпроектами.

Философия Spring Cloud — брать проверенные patterns и tools из индустрии (изначально многое из Netflix стека, потом отдельные independent projects) и предоставлять idiomatic Spring integration. Разработчик получает автоконфигурацию, аннотации, интеграцию с остальной Spring экосистемой вместо ручной настройки каждого компонента.

Каждый компонент называется spring-cloud-* — spring-cloud-consul-discovery, spring-cloud-openfeign, spring-cloud-gateway. Подключаются как отдельные dependencies по необходимости. Проект не заставляет использовать всё — можно выбрать нужные компоненты для конкретной архитектуры.

## Совместимость Spring Boot и Spring Cloud

Spring Cloud тесно привязан к версии Spring Boot. Каждый Spring Cloud release train совместим только с конкретным диапазоном Spring Boot версий. Использование несовместимых версий приведёт либо к ошибкам сборки либо к runtime failures.

Таблица соответствия:

| Spring Boot | Spring Cloud release train |
|---|---|
| 2.2.x | Hoxton |
| 2.3.x | Hoxton |
| 2.4.x | 2020.0.x (Ilford) |
| 2.6.x | 2021.0.x (Jubilee) |
| 3.0.x | 2022.0.x (Kilburn) |
| 3.1.x | 2022.0.x |
| 3.2.x | 2023.0.x (Leyton) |
| 3.3.x | 2023.0.x |

Practical rule — при апгрейде Spring Boot обязательно проверить и обновить Spring Cloud dependency management BOM. При апгрейде Spring Cloud обязательно проверить что версия совместима с текущим Boot.

В КНП master branch использует Boot 2.2 plus Cloud Hoxton.SR3. Master-21 branch с миграцией на Java 21 использует Boot 3 plus Cloud 2022.x. Различные модули на разных Boot версиях подтверждают что Cloud version должен строго matching для каждого.

## Spring Cloud Config

Централизованный конфиг-сервер. Клиенты читают свои конфигурации на старте из этого сервера вместо чтения из локальных application.yml. Позволяет управлять конфигурацией централизованно для множества сервисов.

Backend обычно Git repository содержащий yml или properties файлы:
```
config-repo/
├── application.yml            ← общий для всех
├── isna-knp.yml               ← для isnaknp
└── isna-knp-prod.yml          ← для isnaknp с profile=prod
```

Клиент указывает URL config-server и своё application name:
```yaml
# bootstrap.yml
spring:
  application:
    name: isna-knp
  cloud:
    config:
      uri: http://config-server:8888
```

На старте client tries pull конфиг из Config Server. Если недоступен — startup может fail (или использовать local fallback в зависимости от настройки).

В КНП Config Server не используется. Конфиги через application.yml plus environment variables в K8s манифестах. Проще и не создаёт дополнительной точки отказа. Config Server имеет смысл для больших deployments с многими environments и dynamic reloading требованиями.

## Service Discovery Consul или Eureka

Service discovery позволяет сервисам находить друг друга по имени вместо hard-coded URLs. Клиент запрашивает у registry «где сервис X?» и получает список активных instances.

Consul это решение от HashiCorp. Универсальный service registry plus KV store plus health checking plus multi-datacenter федерация. В КНП стандартный выбор — Consul.

Eureka был решением от Netflix. Только discovery без дополнительных функций. Устарел, находится в maintenance mode. Не рекомендуется для новых проектов. Kubernetes Service обеспечивает discovery через DNS и endpoints автоматически что делает Eureka излишним в K8s environments.

Детально Consul обсуждался в файле 11 consul-detailed.

## Spring Cloud LoadBalancer

Client-side load balancer как замена deprecated Ribbon. Работает с RestTemplate, WebClient, Feign через integration.

Использование через service name вместо URL:
```java
@Autowired RestTemplate rest;   // с @LoadBalanced аннотацией

UserDto u = rest.getForObject("http://isnaknpuser/api/users/1", UserDto.class);
// isnaknpuser → Consul lookup → выбор instance → replace URL → request
```

Под капотом Spring Cloud LoadBalancer использует ServiceInstanceListSupplier получающий список instances из Consul или другого registry. Применяет configured algorithm (RoundRobin, Random, custom) для выбора конкретного instance. Заменяет service name в URL на actual host:port и выполняет запрос.

Детально в файле 31 load-balancer.

## Spring Cloud OpenFeign

Декларативный HTTP клиент. Определяется как Java interface с annotations, Spring генерирует реализацию:
```java
@FeignClient(name = "isnaKnpUser")
public interface UserClient {
    @GetMapping("/api/users/{id}")
    UserDto getById(@PathVariable Long id);
}
```

Использование как обычный @Autowired bean:
```java
@Autowired UserClient userClient;
UserDto u = userClient.getById(42L);
```

Интегрируется автоматически с Spring Cloud LoadBalancer для service discovery. Поддерживает Circuit Breaker integration для resilience. Настраиваемая retry и timeout policies.

Замена ручного использования RestTemplate или WebClient плюс manual service discovery. Cleaner code, less boilerplate, easier testing через interface mocking.

Детально Feign обсуждался в файле 11 consul-detailed.

## Spring Cloud Circuit Breaker

Абстракция над разными circuit breaker implementations. Позволяет менять underlying реализацию без изменения application code.

Реализации. Resilience4j — современная замена Hystrix, наиболее часто используется. Sentinel — решение от Alibaba, популярно в Азии. Spring Retry — простой retry без full circuit breaker behavior.

Hystrix от Netflix был первым mainstream circuit breaker в Java. Deprecated с 2018 года. В maintenance mode — только критические bug fixes, никакого нового функционала. Не рекомендуется для новых проектов.

Пример с Resilience4j:
```java
@CircuitBreaker(name = "isnaKnpUser", fallbackMethod = "fallback")
public UserDto getUser(Long id) {
    return userClient.getById(id);
}

public UserDto fallback(Long id, Throwable t) {
    return UserDto.EMPTY;
}
```

Circuit breaker имеет три состояния. CLOSED normal operation — все запросы проходят. OPEN при обнаружении проблем downstream — запросы сразу fail с fallback, не отправляются в downstream. HALF_OPEN после cooldown period — pass через определённое количество test запросов, based на их результате принимается решение возвращаться в CLOSED или оставаться OPEN.

Настройка порогов — например «50 процентов ошибок за последние 20 запросов → OPEN на 30 секунд → HALF_OPEN с 5 test запросами». Настраивается per-circuit для разных services.

Плюсы circuit breaker. Защита от каскадных отказов — падение одного downstream не блокирует upstream сервисы. Fast fail улучшает UX — пользователь получает ошибку быстро вместо ожидания timeout. Даёт downstream время на recovery без continued нагрузки.

## Spring Cloud Gateway

Современная замена Zuul (устаревшего Netflix gateway). Reactive gateway на Spring WebFlux plus Netty. Обеспечивает production-grade API gateway functionality.

Конфигурация через yml или Java:
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

Возможности. Роутинг по любым атрибутам HTTP — path, headers, method, cookies, host. Filters для transformations — authentication, rate limiting, header manipulation, retry. Circuit breaker integration через Resilience4j. Metrics через Micrometer. Custom filters через Java для сложной logic.

В КНП isna-knp-gateway использует Zuul 1 (Netflix, deprecated). Не мигрирован на Spring Cloud Gateway. Memory кейс knp-gateway-no-java21 отмечает что gateway остаётся на Java 11 потому что Zuul 1 не совместим с новыми Java версиями. Миграция на Spring Cloud Gateway необходима для дальнейшего upgrade path но не выполнена по причине сложности и complexity.

## Distributed Tracing

Sleuth или Micrometer Tracing. Sleuth deprecated с Boot 3, Micrometer Tracing это замена в новых версиях.

Что делает. Автоматически добавляет traceId и spanId в MDC — все логи request имеют одинаковый traceId. Пробрасывает эти identifiers через HTTP headers к downstream services — все связанные операции получают одинаковый traceId. Экспортирует traces в специализированные системы — Zipkin, Jaeger, OpenTelemetry compatible backends.

Пример лога с trace ID:
```
2026-09-05 10:15:30 [traceId=abc123,spanId=def456] INFO OrderService - Order created
```

В Zipkin или Jaeger UI видна полная trace одного запроса. HTTP /api/knp вызывает Feign который делает HTTP request к isnaKnpUser, который делает SQL query к PostgreSQL. Каждый шаг — отдельный span в trace с timing information. Легко определить где latency, где ошибки.

В КНП distributed tracing используется частично. ELK plus custom correlation через headers покрывает большую часть use cases. Полноценный Zipkin/Jaeger может быть добавлен но требует инфраструктурной инвестиции.

## Spring Cloud Stream

Абстракция над message brokers (Kafka, RabbitMQ) — код независим от конкретной implementation:
```java
@Bean
public Function<OrderEvent, PaymentEvent> processOrder() {
    return event -> {
        // обработка
        return new PaymentEvent(...);
    };
}
```

Binding конфигурируется в yml:
```yaml
spring.cloud.stream.bindings:
  processOrder-in-0:
    destination: orders
    binder: kafka
  processOrder-out-0:
    destination: payments
```

Плюсы. Смена Kafka на RabbitMQ или наоборот одним изменением binder configuration без изменения application code. Единый API для всех message brokers. Автоматическая интеграция с Spring ecosystem.

Минусы. Абстракция утечная — специфические features конкретных brokers не полностью доступны через generic API. Дополнительный слой между application и broker увеличивает complexity. Обычно требуется troubleshooting на уровне underlying broker что делает abstraction менее полезной.

В КНП Spring Cloud Stream не используется. Прямой Spring AMQP через RabbitTemplate для Rabbit, прямой Kafka client если Kafka используется. Более straightforward, easier to reason about.

## Spring Cloud Contract

Consumer-driven contract testing framework. Producer генерирует stubs основанные на contract definitions. Consumer использует эти stubs в своих тестах. Гарантия — producer не сломает contract потому что stubs обновляются с каждым producer release.

Идея — вместо polling downstream API для changes, contract testing catches API breaks early в CI pipeline. Consumer знает что если тесты proceed с новыми stubs — API совместим.

Использование редкое. Применяется только в зрелых микросервисных командах с advanced testing practices. Requires discipline с contracts definitions и stubs versioning. Overhead может не оправдываться для small teams.

## Spring Cloud Bus

Распределённая шина для events, обычно поверх RabbitMQ или Kafka. Основной use case — refresh конфигурации на всех instances одним запросом:
```
POST /actuator/bus-refresh   → шлётся событие в Rabbit
→ все подписчики (все instances) → refresh конфига из Config Server
```

Позволяет centralized configuration updates без manual redeploy или scripted process. Полезно в environments где instances плавают (auto-scaling groups) и hard to track individually.

В Kubernetes окружениях необходимость меньше — просто rollout restart deployment применяет configuration changes. Spring Cloud Bus имеет смысл в non-K8s environments или для scenarios требующих runtime configuration updates без restart.

## Spring Cloud Kubernetes

Специфичный для K8s компонент интегрирующий Spring с Kubernetes native concepts. Позволяет использовать ConfigMap и Secret как источники конфигурации через standard Spring PropertySources. Kubernetes Service вместо Consul для service discovery — просто DNS resolution.

Альтернатива Consul для чисто K8s deployments. Убирает необходимость в external service registry когда всё в одном cluster. Simpler infrastructure — меньше компонентов для support.

В КНП может быть alternative migration path если решается уйти от Consul. Требует reevaluation service-to-service communication patterns.

## Deprecated Netflix stack

Netflix открыли много компонентов, потом остановили их развитие. Spring Cloud поддерживает старые версии для legacy compatibility но рекомендует замены для новых проектов.

Таблица соответствия deprecated к current:

| Deprecated | Замена |
|---|---|
| Hystrix (circuit breaker) | Resilience4j |
| Ribbon (LB) | Spring Cloud LoadBalancer |
| Zuul 1 (gateway) | Spring Cloud Gateway |
| Eureka (discovery) | Consul или K8s Service |
| Archaius (config) | Spring Cloud Config |

В новых проектах — только замены. Legacy может продолжать использовать Netflix stack но не может upgrade на новые Boot версии без миграции.

## Типичная микросервисная архитектура

Собранная воедино архитектура с Spring Cloud компонентами:
```
Users
  │
  ▼
[Ingress / nginx / F5]         ← L7 внешний
  │
  ▼
[Spring Cloud Gateway]         ← routing, auth, rate limit
  │
  ▼
[Consul discovery]             ← service registry
  │
  ├─ service A (Boot + Consul client)
  │    ├─ Feign → service B
  │    ├─ Circuit Breaker (Resilience4j)
  │    └─ PostgreSQL
  │
  ├─ service B
  │
  └─ service C
       └─ RabbitMQ / Kafka
```

Дополнительная инфраструктура. ELK для logs. Prometheus plus Grafana для metrics. Zipkin/Jaeger для distributed tracing. Keycloak для authentication.

## Пример полной конфигурации

Dependencies для типового микросервиса КНП стиля:
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

Конфигурация:
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

## Что реально используется в КНП

Из практики и memory кейсов. Consul для service discovery и health checking. Feign для HTTP клиентов (isnaKnpUserFeignClient, IsnaFnoDictionaryFeignClient и другие). Spring Cloud LoadBalancer в master-21 (Java 21 модули), Ribbon в master (Java 11 модули). Actuator plus Micrometer для metrics в Prometheus format. Zuul 1 для gateway isna-knp-gateway, только Java 11 (memory knp-gateway-no-java21). Circuit breaker точечно применяется в критических интеграциях.

Не используется. Spring Cloud Config — конфигурация через yml и env variables. Spring Cloud Stream — прямой Spring AMQP предпочтительнее. Spring Cloud Bus — K8s rollout restart достаточен. Spring Cloud Contract — тестирование через integration tests без formal contracts.

## Итоги

Spring Cloud это зонтик проектов для микросервисной инфраструктуры. Не единая библиотека — набор специализированных компонентов. Подключаются по необходимости.

Живые актуальные компоненты. Consul для service discovery. Feign для HTTP clients. Spring Cloud LoadBalancer как замена Ribbon. Spring Cloud Gateway как замена Zuul. Resilience4j для circuit breaker. Micrometer Tracing для distributed tracing.

Deprecated Netflix стек. Hystrix, Ribbon, Zuul 1, Eureka. В новых проектах не использовать.

Attractive но редко нужны. Config Server, Stream, Bus, Contract. Имеют место в специфических scenarios но обычно overkill.

В КНП практика — Consul plus Feign plus Zuul 1 (legacy) plus LoadBalancer или Ribbon в зависимости от Java version модуля.

Совместимость Boot и Cloud версий критична. Использование несовместимых версий приведёт к build или runtime failures. Проверять release train при upgrade.

Дальше — Nodes как universal термин в distributed systems, различные интерпретации в разных технологиях.
