# 36. Spring Cloud обзор

Что такое Spring Cloud, из чего состоит, что реально живо, что deprecated.

---

## 1. Что такое Spring Cloud

**Spring Cloud** — набор проектов на Spring Boot для типовых задач в микросервисной архитектуре:
- Service discovery.
- Externalized configuration.
- Client-side load balancing.
- Circuit breakers.
- Distributed tracing.
- API gateway.
- Messaging.

Не одна библиотека, а **зонтик**. Каждый компонент — отдельный проект (`spring-cloud-*`).

---

## 2. Совместимость Boot ↔ Cloud

Spring Cloud тесно привязан к версии Boot:

| Spring Boot | Spring Cloud |
|---|---|
| 2.2.x | Hoxton |
| 2.3.x | Hoxton |
| 2.4.x | 2020.0.x (Ilford) |
| 2.6.x | 2021.0.x (Jubilee) |
| 3.0.x | 2022.0.x (Kilburn) |
| 3.1.x | 2022.0.x |
| 3.2.x | 2023.0.x (Leyton) |
| 3.3.x | 2023.0.x |

**Кавет**: несовместимые версии — не будет собираться или упадёт в рантайме. См. официальную таблицу.

В ИСНА (`master`): Boot 2.2 + Cloud Hoxton.SR3.

---

## 3. Ключевые компоненты

### 3.1 Spring Cloud Config

Централизованный конфиг-сервер. Клиенты читают конфиги на старте.

Backend — Git repo с yml/properties файлами:
```
config-repo/
├── application.yml            ← общий для всех
├── isna-knp.yml               ← для isnaknp
├── isna-knp-prod.yml          ← для isnaknp с profile=prod
```

Клиент:
```yaml
# bootstrap.yml
spring:
  application:
    name: isna-knp
  cloud:
    config:
      uri: http://config-server:8888
```

На старте Client вытянет конфиг из Config Server.

**В ИСНА не используется** — конфиги через application.yml + env-переменные в K8s. Проще.

### 3.2 Spring Cloud Consul / Eureka

Service discovery. Обсуждали в `11-consul-detailed.md`.

- **Consul** — HashiCorp, KV + discovery + health.
- **Eureka** — Netflix, только discovery. Устарел.

**В ИСНА — Consul**.

### 3.3 Spring Cloud LoadBalancer

Client-side LB. Замена Ribbon.

```java
@Autowired RestTemplate rest;   // с @LoadBalanced

UserDto u = rest.getForObject("http://isnaknpuser/api/users/1", UserDto.class);
// isnaknpuser → Consul lookup → инстанс → replace URL → request
```

Обсуждали в `31-load-balancer.md`.

### 3.4 Spring Cloud OpenFeign

Декларативный HTTP-клиент. Обсуждали в `11-consul-detailed.md`.

```java
@FeignClient(name = "isnaKnpUser")
public interface UserClient {
    @GetMapping("/api/users/{id}")
    UserDto getById(@PathVariable Long id);
}
```

### 3.5 Spring Cloud Circuit Breaker

Абстракция над разными circuit-breaker реализациями:
- **Resilience4j** — современная замена Hystrix.
- **Sentinel** — от Alibaba.
- **Spring Retry** — простой retry без CB.

**Hystrix** (Netflix) — deprecated с 2018, в maintenance mode.

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

Circuit breaker states:
- **CLOSED** — normal.
- **OPEN** — все запросы сразу fail (не идут в downstream).
- **HALF_OPEN** — тестовые запросы для проверки восстановления.

Порог: например «50% ошибок за последние 20 запросов → OPEN на 30 сек → HALF_OPEN».

Плюсы: защита от каскадных отказов, fast fail.

### 3.6 Spring Cloud Gateway

Современная замена Zuul (Netflix). Reactive gateway на Spring WebFlux + Netty.

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

Возможности:
- Роутинг по любым атрибутам HTTP.
- Filters (auth, rate limit, header manipulation, retry).
- Circuit breaker integration.
- Metrics.

**В ИСНА**: `isna-knp-gateway` = **Zuul 1** (Netflix, deprecated). Не мигрирован (memory `knp-gateway-no-java21` — остаётся на Java 11).

### 3.7 Spring Cloud Sleuth / Micrometer Tracing

Distributed tracing.

- **Sleuth** — deprecated с Boot 3.
- **Micrometer Tracing** — замена.

Что делает:
- Автоматически добавляет `traceId` и `spanId` в MDC.
- Пробрасывает через HTTP headers.
- Экспортирует в **Zipkin** / **Jaeger** / **OpenTelemetry**.

Логи с trace-ID:
```
2026-09-05 10:15:30 [traceId=abc123,spanId=def456] INFO OrderService - Order created
```

В Zipkin/Jaeger видишь распределённый trace: `HTTP /api/knp` → `Feign → isnaKnpUser` → `SQL → PG`.

Использование в ИСНА — частично, через ELK + custom correlation.

### 3.8 Spring Cloud Stream

Абстракция над брокерами (Kafka / Rabbit) — пишешь код независимый от implementation.

```java
@Bean
public Function<OrderEvent, PaymentEvent> processOrder() {
    return event -> {
        // ...
        return new PaymentEvent(...);
    };
}
```

Binding в yml:
```yaml
spring.cloud.stream.bindings:
  processOrder-in-0:
    destination: orders
    binder: kafka
  processOrder-out-0:
    destination: payments
```

Плюс: смена Kafka↔Rabbit одним изменением конфига. Минус: абстракция утечная, лишний слой.

**В ИСНА** — прямой Spring AMQP (RabbitTemplate), без Stream.

### 3.9 Spring Cloud Contract

Consumer-driven contract testing.

Producer генерирует стабы; Consumer использует стабы в тестах. Гарантия что producer не сломает contract.

Использование редкое, только в зрелых микросервисных командах.

### 3.10 Spring Cloud Bus

Распределённая шина для событий (обычно поверх RabbitMQ / Kafka). Пример: refresh конфига на всех инстансах одним запросом.

```
POST /actuator/bus-refresh   → шлётся событие в Rabbit
→ все подписчики (все инстансы) → refresh конфига
```

Редко нужен в K8s (просто rollout restart).

### 3.11 Spring Cloud Kubernetes

Специфичный для K8s. Позволяет использовать ConfigMap/Secret как источники конфига, K8s Service вместо Consul для discovery.

**Альтернатива Consul** для чисто K8s-развёртывания.

---

## 4. Что deprecated (Netflix стек)

Netflix открыли много компонентов, потом закрыли развитие. Spring Cloud поддерживает старое, но советует замены.

| Deprecated | Замена |
|---|---|
| **Hystrix** (circuit breaker) | Resilience4j |
| **Ribbon** (LB) | Spring Cloud LoadBalancer |
| **Zuul 1** (gateway) | Spring Cloud Gateway |
| **Eureka** (discovery) | Consul / K8s Service |
| **Archaius** (config) | Spring Cloud Config |

В новых проектах — только замены. Legacy может держать Netflix, но нельзя обновиться на новые Boot.

---

## 5. Как выглядит типичная микросервисная архитектура (Spring Cloud)

```
Users
  │
  ▼
[Ingress / nginx / F5]  ← L7 внешний
  │
  ▼
[Spring Cloud Gateway]   ← routing, auth, rate limit
  │
  ▼
[Consul discovery]       ← service registry
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

+ инфраструктура:
- **ELK** для логов.
- **Prometheus + Grafana** для метрик.
- **Zipkin/Jaeger** для tracing.
- **Keycloak** для auth.

---

## 6. Пример полной конфигурации сервиса

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

---

## 7. Что реально используется в ИСНА

Из memory и опыта:
- **Consul** — service discovery + health.
- **Feign** — HTTP клиенты (isnaKnpUserFeignClient, IsnaFnoDictionaryFeignClient).
- **Spring Cloud LoadBalancer** (в master-21, Java 21) / **Ribbon** (в master, Java 11).
- **Actuator + Micrometer** — метрики.
- **Zuul 1** — gateway (`isna-knp-gateway`), только Java 11 (memory `knp-gateway-no-java21`).
- Circuit breaker — точечно.
- **НЕ используется**: Spring Cloud Config, Stream, Bus, Contract.

---

## 8. Собесные вопросы

1. **Что такое Spring Cloud?** — Зонтик проектов для микросервисной инфраструктуры (discovery, LB, config, gateway, tracing).
2. **Какие Netflix компоненты deprecated?** — Hystrix (→ Resilience4j), Ribbon (→ SC LoadBalancer), Zuul 1 (→ SC Gateway), Eureka (→ Consul).
3. **Что делает Circuit Breaker?** — Защита от каскадных отказов; при N ошибках подряд размыкается → fast fail.
4. **Три состояния circuit breaker?** — CLOSED (normal), OPEN (fail-fast), HALF_OPEN (тест восстановления).
5. **Что такое distributed tracing?** — Прокидывать `traceId`/`spanId` через все сервисы; экспорт в Zipkin/Jaeger.
6. **Sleuth vs Micrometer Tracing?** — Sleuth deprecated с Boot 3; Micrometer Tracing — замена.
7. **Что такое Spring Cloud Config?** — Централизованный конфиг-сервер (обычно Git backend).
8. **Разница Consul и Eureka?** — Consul: registry + KV + health + DNS + multi-DC; Eureka: только registry, deprecated.
9. **Что такое Spring Cloud Gateway?** — Reactive gateway (WebFlux + Netty), замена Zuul.
10. **bootstrap.yml — зачем?** — Читается до application.yml; нужен для Spring Cloud (Consul, Config).
11. **Какую версию Spring Cloud выбрать?** — По совместимости с Boot (см. release train).
12. **Spring Cloud Stream — что и когда?** — Абстракция над брокерами (Kafka/Rabbit); редко нужна, лишний слой.
13. **Как distributed traces выглядят в логе?** — `[traceId=abc,spanId=def]` в MDC.

---

## Итог

- **Spring Cloud** = зонтик для микросервисной инфры.
- **Живо**: Consul, Feign, LoadBalancer, Gateway (новый), Resilience4j, Micrometer Tracing.
- **Deprecated Netflix**: Hystrix, Ribbon, Zuul 1, Eureka.
- **Attractive-но редко нужно**: Config Server, Stream, Bus, Contract.
- **В ИСНА**: Consul + Feign + Zuul 1 (legacy) + LoadBalancer/Ribbon (в зависимости от Java-версии).

Следующий — `37-nodes-detailed.md`.
