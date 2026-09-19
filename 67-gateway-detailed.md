# 67. Gateway / шлюз углубленно

API Gateway pattern детально. Функции, реализации, паттерны, антипаттерны.

---

## 1. Что такое Gateway / шлюз

**API Gateway** — **единая точка входа** для клиентов в микросервисную систему.

```
Client (browser, mobile, partner)
              │
              ▼
        [API Gateway]     ← единая точка входа
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 Service A  Service B  Service C
```

Все внешние запросы идут через gateway. Backend'ы не видны напрямую.

### 1.1 Зачем

Проблема без gateway:
- Клиент должен знать URLs всех сервисов.
- Каждый сервис реализует auth, rate limit, logging.
- Нельзя менять топологию без ломки клиентов.
- CORS через каждый сервис.
- Нет единой точки для metrics.

Gateway решает всё это.

---

## 2. Функции gateway

### 2.1 Routing

Определяет **какой сервис** обрабатывает URL.

```
GET /api/orders/*     → order-service
GET /api/payments/*   → payment-service
GET /api/users/*      → user-service
POST /api/upload/*    → file-service
```

Правила по:
- Path (`/api/orders/*`).
- Host header (`api.example.com` vs `admin.example.com`).
- HTTP method (POST vs GET).
- Headers (`X-Version: v2`).
- Query params.

### 2.2 Authentication / Authorization

Централизованная проверка **auth**:
- Проверить JWT valid + not expired.
- Проверить token audience / issuer.
- Извлечь user info.
- Отправить `X-User-Id` header в backend.

Backend'ы получают уже **аутентифицированные** запросы. Могут не проверять токен сами (доверяют gateway).

### 2.3 Rate limiting

Ограничить rate от IP / user / API key.

```
Public tier: 100 req/min
Premium tier: 1000 req/min
Admin: unlimited
```

Защита от DDoS, abuse.

### 2.4 SSL termination

Клиент → HTTPS → **Gateway** → HTTP → backends.

Backends не имеют SSL — быстрее, проще управлять сертификатами.

### 2.5 Request/response transformation

- Добавить headers.
- Переименовать paths.
- Скрыть внутренние URLs.
- Convert формат (XML ↔ JSON).

### 2.6 Aggregation

Один client call → несколько backend calls → assemble response.

```
GET /api/user-profile/{id}
    ↓
    ├─ GET /users/{id} → user-service
    ├─ GET /orders?user={id} → order-service
    └─ GET /preferences/{id} → prefs-service
    ↓
    JSON: { user, orders[], preferences }
```

Клиент делает **1 запрос** вместо 3.

### 2.7 Caching

Cache responses (обычно GET).

```
Cache-Control: max-age=60
```

Разгружает backends для read-heavy APIs.

### 2.8 Logging / Metrics / Tracing

Все запросы через gateway → централизованный audit.

- Access logs.
- Latency metrics per endpoint.
- Trace-Id генерация (передаётся downstream).

### 2.9 Circuit breaking / retry

Защита от cascading failures (см. `52-microservices-resilience.md`).

### 2.10 API composition

Похоже на aggregation, но для **сложных workflows**.

Иногда gateway превращается в **BFF** — специфичный для клиента.

---

## 3. Продукты

### 3.1 Nginx / Nginx Plus

Простой gateway (только L7).

Плюсы: быстрый, знакомый, мало ресурсов.
Минусы: routing через config-файлы, ограниченная логика.

### 3.2 Kong

Open-source gateway на Nginx + Lua.

Плюсы: plugins (auth, rate limit), UI, богатая экосистема.
Минусы: сложнее чем nginx, ещё один компонент для управления.

### 3.3 Envoy

Modern proxy (CNCF).

Плюсы: L4/L7, dynamic конфиг через xDS, метрики, tracing.
Минусы: сложный, конфиг разросся.

Основа **Istio** service mesh.

### 3.4 Traefik

Modern, cloud-native, auto-discovery.

Плюсы: интеграция с K8s / Docker, автоматическое SSL (Let's Encrypt).
Минусы: молодой (относительно), меньше plugins.

### 3.5 Spring Cloud Gateway

Reactive (WebFlux + Netty) Java-based gateway.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway, spring-cloud
        - id: payments
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          filters:
            - CircuitBreaker=paymentBreaker
```

Плюсы: Java экосистема, интеграция со Spring Boot, filters из Java.
Минусы: Java overhead (память), reactive learning curve.

### 3.6 Zuul (Netflix)

**Zuul 1** — legacy, blocking, Java. Deprecated.
**Zuul 2** — async, но малоактивно поддерживается.

Заменяется на Spring Cloud Gateway.

**В ИСНА**: `isna-knp-gateway` = **Zuul 1** (Java 11, `knp-gateway-no-java21`). Не мигрирован на Java 21 из-за Zuul.

### 3.7 AWS API Gateway

Managed от AWS. Интеграция с Lambda, IAM, Cognito.

Плюсы: managed, не надо поддерживать.
Минусы: cost, vendor lock-in.

### 3.8 Google Cloud API Gateway, Azure API Management

Аналоги от других cloud vendors.

---

## 4. Spring Cloud Gateway — детально

Полный пример:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders-service
          uri: lb://order-service   # через Consul discovery
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST,PUT,DELETE
            - Header=X-Version, v[12]
          filters:
            - StripPrefix=1                              # /api/orders/1 → /orders/1
            - AddRequestHeader=X-Source, gateway
            - AddResponseHeader=X-Gateway-Version, 1.0
            - name: CircuitBreaker
              args:
                name: ordersBreaker
                fallbackUri: forward:/fallback/orders
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20

globalcors:
  cors-configurations:
    '[/**]':
      allowedOrigins: "https://example.com"
      allowedMethods: "*"
```

Основные компоненты:
- **Route** — правило маршрутизации.
- **Predicate** — условие (path, header, method).
- **Filter** — pre/post обработка.
- **URI** — где backend (обычный URL или `lb://service-name` через discovery).

### 4.1 Custom filter

```java
@Component
public class TraceIdFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String traceId = UUID.randomUUID().toString();
        ServerHttpRequest req = exchange.getRequest().mutate()
            .header("X-Trace-Id", traceId)
            .build();
        return chain.filter(exchange.mutate().request(req).build());
    }

    @Override
    public int getOrder() {
        return -1;   // высокий приоритет
    }
}
```

### 4.2 Auth в gateway

```yaml
filters:
  - name: TokenRelay        # пробрасывает JWT downstream
```

Или через Spring Security на gateway → decodes JWT → adds user info headers.

---

## 5. Zuul в ИСНА

`isna-knp-gateway` = **Zuul 1**.

Из memory `knp-gateway-no-java21` — остался на Java 11 (Zuul 1 не supports Java 21 нормально).

Также `knp-gateway-no-java21`: на **knp21** контуре отсюда 404 на `/services/isnaknpintegration` — потому что gateway не в контуре Java 21.

Пример типичной конфигурации Zuul:
```yaml
zuul:
  routes:
    isnaknpintegration:
      path: /services/isnaknpintegration/**
      serviceId: isnaknpintegration
      stripPrefix: false
    isnaknpuser:
      path: /services/isnaknpuser/**
      serviceId: isnaknpuser
  ribbon:
    eager-load:
      enabled: true
  host:
    connect-timeout-millis: 5000
    socket-timeout-millis: 30000
```

Маршрутизирует по префиксу URL на разные микросервисы через Consul (Ribbon LB).

---

## 6. Nginx как gateway

Простой, но эффективный:

```nginx
upstream orders_backend {
    server order-service-1:8080;
    server order-service-2:8080;
    server order-service-3:8080;
}

upstream payments_backend {
    server payment-service-1:8080;
    server payment-service-2:8080;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;

    # Global rate limit
    limit_req_zone $binary_remote_addr zone=global:10m rate=100r/s;

    # JWT validation через auth_request (extra service)
    auth_request /auth;

    # Orders
    location /api/orders/ {
        limit_req zone=global burst=200 nodelay;
        proxy_pass http://orders_backend/;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-User-Id $auth_user;
    }

    # Payments
    location /api/payments/ {
        limit_req zone=global burst=50 nodelay;
        proxy_pass http://payments_backend/;
    }

    # Auth check
    location = /auth {
        internal;
        proxy_pass http://auth-service/validate;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URI $request_uri;
    }
}
```

Простые cases — nginx идеален. Сложные — Kong / Envoy / Spring Cloud Gateway.

---

## 7. BFF — Backend for Frontend

Разбирали в `49-microservices-decomposition.md`.

**Отдельный gateway на каждый тип клиента**:

```
Web browser  →  BFF-Web    ─┐
Mobile app   →  BFF-Mobile ─┼──►  Services
Partners     →  BFF-Partner─┘
```

Каждый BFF:
- Optimizes API под свой клиент.
- Aggregate запросы.
- Различные data shapes.

Пример: Mobile хочет compact response (мало данных); Web — full. BFF-Mobile stripp'ит fields, BFF-Web полный.

Плюсы:
- Клиент-специфичная оптимизация.
- Frontend команды владеют BFF.

Минусы:
- Дублирование логики.
- Больше сервисов.

---

## 8. Gateway vs Service Mesh

Часто путают. Разница — **где живут cross-cutting concerns**.

### 8.1 API Gateway

- **Edge** — внешний трафик.
- **Централизованный** — один или несколько.
- Функции: routing клиентов, auth, rate limit, aggregation.
- Обычно stateless.

### 8.2 Service Mesh (Istio, Linkerd)

- **Sidecar** — на каждом сервисе.
- **Distributed** — везде.
- Функции: mTLS internal, retry, circuit breaker, tracing, LB.
- Cross-cutting infrastructure.

### 8.3 Часто вместе

```
External → API Gateway → Service (with sidecar) ──mTLS──► Service (with sidecar)
```

Gateway для клиентов; mesh для internal.

---

## 9. Ingress в K8s

**Ingress** — K8s абстракция для L7 роутинга. Реализуется **Ingress Controller** (обычно nginx-ingress).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: knp-ingress
spec:
  rules:
    - host: knp.kgd.gov.kz
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: isnaknpgateway
                port:
                  number: 8080
```

**Ingress vs Gateway**:
- **Ingress** = базовый L7 роутинг в K8s.
- **API Gateway** = full-featured (auth, rate limit, aggregation).

Часто **Ingress перед Gateway**:
```
Internet → nginx-ingress → API Gateway → backends
```

Ingress для базового routing, Gateway для бизнес-логики.

В ИСНА так и есть: nginx-ingress → `isna-knp-gateway` (Zuul) → микросервисы.

---

## 10. Antipatterns

### 10.1 Business logic в gateway

Gateway накапливает бизнес-правила → превращается в **integration monolith**.

Правило: gateway = **routing + cross-cutting** (auth, rate limit). Никакой доменной логики.

Если нужна логика — **BFF** отдельный сервис.

### 10.2 Один огромный gateway

Все routes, все команды в одном gateway → узкое место коммуникации между командами.

Правило: **несколько gateways** по domain / клиенту (public API gateway + admin gateway + partner gateway).

### 10.3 Sync-only gateway

Клиент ждёт response через gateway. Для долгих операций — плохо.

Правило: **async patterns** для long-running (202 Accepted + polling).

### 10.4 SPOF

Один gateway = single point of failure.

Правило: **HA** — несколько replicas + LB перед ними.

### 10.5 Проброс backend'ов «как есть»

Gateway тупо форвардит запрос → нет ценности.

Правило: gateway должен добавлять value: auth, rate limit, transformation, aggregation.

### 10.6 Coupling к backend

Gateway знает internal detail backends (specific paths, data shapes).

Правило: **stable API contract** между gateway и backends. Изменение backend не должно ломать gateway.

---

## 11. Deployment patterns

### 11.1 One gateway

Простой случай.
```
Client → [Gateway] → Services
```

### 11.2 Regional gateways

Multi-region deployment.
```
US Client → US Gateway → US services
EU Client → EU Gateway → EU services
```

### 11.3 Multi-layer

```
Internet → CDN → Ingress → API Gateway → Services
```

- CDN для static + cache.
- Ingress для K8s.
- Gateway для API logic.

### 11.4 Micro-gateway

Каждый сервис имеет свой мини-gateway (для auth, rate limit).

Разница со sidecar: sidecar transparent; micro-gateway explicit.

---

## 12. Мониторинг gateway

Ключевые метрики:
- **Rate** — RPS per route.
- **Latency** — p50, p95, p99.
- **Error rate** — % 4xx, 5xx.
- **Backend errors** — какой сервис возвращает 500.
- **Circuit breaker state**.
- **Rate limit rejections**.
- **Auth failures**.

Alerts:
- p99 latency > threshold.
- Error rate > 5%.
- Circuit breaker OPEN для важного backend.
- Rate limit hits (может атака).

---

## 13. Как gateway работает под нагрузкой

Gateway часто — bottleneck.

### 13.1 Оптимизации

- **Async/non-blocking** — Spring Cloud Gateway (Netty), Kong (nginx).
- **Connection pooling** к backends.
- **HTTP/2** к backends.
- **Response caching**.
- **Horizontal scaling**.

### 13.2 Резервирование

Gateway ест меньше CPU/memory чем backends, но нужно резервировать:
- CPU: 1-2 cores.
- Memory: 512 MB - 2 GB.
- 3+ replicas для HA.

---

## 14. Security в gateway

### 14.1 Standard

- **TLS termination**.
- **JWT / OAuth validation**.
- **CORS**.
- **Rate limiting**.
- **Request size limits**.
- **XSS/SQL injection prevention** (headers, patterns).

### 14.2 WAF (Web Application Firewall)

Продвинутая защита:
- OWASP Top 10.
- Bot detection.
- DDoS mitigation.
- IP reputation.

Отдельный layer или встроен в gateway (Kong Plus, AWS WAF).

---

## 15. Пример полного flow ИСНА

```
Пользователь → https://knp.kgd.gov.kz/api/fno/123
                    │
                    ▼
            [nginx-ingress]
                    │  routes: knp.kgd.gov.kz/api → isnaknpgateway
                    ▼
            [isna-knp-gateway] (Zuul)
                    │  routes: /api/fno → isnaknpintegration
                    │  Consul discovery: pod 10.0.1.5:8080
                    ▼
            [isna-knp-integration pod]
                    │
                    ├─ Feign → Consul → isnaknpuser
                    ├─ Feign → Consul → isnaknpfno
                    └─ PG / RabbitMQ
```

**Ingress**: SSL termination + host routing.
**Gateway (Zuul)**: path routing + auth + LB.
**Backend**: бизнес-логика.

---

## 16. Когда gateway не нужен

- **Небольшой monolith** — один сервис, gateway избыточен.
- **Internal-only APIs** — service mesh может заменить.
- **Serverless** (AWS Lambda) — API Gateway встроен.

Но для микросервисов **обычно нужен**.

---

## 17. Собесные вопросы

1. **Что такое API Gateway?** — Единая точка входа для клиентов; routing + cross-cutting.
2. **Функции Gateway?** — Routing, auth, rate limit, SSL termination, transformation, aggregation, caching, monitoring.
3. **Gateway vs Ingress?** — Ingress: базовый K8s L7 routing; Gateway: full-featured (auth, rate limit, aggregation).
4. **Gateway vs Service Mesh?** — Gateway edge; mesh sidecar на каждом сервисе; часто вместе.
5. **Что такое BFF?** — Backend for Frontend; отдельный gateway per client type.
6. **Продукты gateway?** — nginx, Kong, Envoy, Traefik, Spring Cloud Gateway, Zuul, AWS API Gateway.
7. **Spring Cloud Gateway vs Zuul?** — SCG reactive (WebFlux+Netty), новее; Zuul 1 legacy blocking.
8. **Почему в ИСНА gateway на Java 11?** — Zuul 1 не supports Java 21 нормально; не мигрирован.
9. **Anti-pattern gateway?** — Business logic в gateway; огромный monolith gateway; SPOF без HA.
10. **API composition?** — Gateway агрегирует несколько backend calls в один response.
11. **Rate limiting алгоритмы?** — Token bucket, sliding window; per IP/user/tenant.
12. **Что такое WAF?** — Web Application Firewall; OWASP Top 10, bot detection, DDoS.
13. **HA gateway — как?** — Несколько replicas + LB перед ними; stateless design.
14. **Как gateway аутентифицирует?** — Проверяет JWT / API key; отправляет `X-User-Id` header downstream.
15. **Что мониторить в gateway?** — Rate, latency (p50/p95/p99), error rate per route, circuit breaker states.

---

## Итог

- **API Gateway** = single entry point для клиентов.
- Функции: **routing, auth, rate limit, SSL, transformation, aggregation, monitoring**.
- Продукты: **nginx / Kong / Envoy / Spring Cloud Gateway / Zuul**.
- В ИСНА: **nginx-ingress → Zuul (`isna-knp-gateway`) → микросервисы**.
- **BFF** для клиент-специфичных gateways.
- **Gateway + Service Mesh** — часто вместе.
- **Anti-patterns**: business logic в gateway, SPOF, coupling к backends.

---

## Финальный итог блока 65-67

- 65 — ИШ / ESB / интеграционные шины (для интеграции legacy/gov систем).
- 66 — Синхронная vs асинхронная обработка (все уровни: метод, HTTP, messaging, server, DB).
- 67 — Gateway углубленно (функции, продукты, паттерны).

**Всего в isna-theory теперь 67 файлов**.

Дальше: **Redis / in-memory cache**, **CAP теорема / distributed systems**, **криптография / ЭЦП Kalkan для ИСНА**, **GitLab CI / DevOps**, **алгоритмы + system design для собесов**. Что берём?
