# 49. Паттерны декомпозиции и коммуникации

Как правильно разделить систему на микросервисы. Как они общаются.

---

## 1. Проблема: где границы сервисов

Плохое решение — по **техническим слоям**:
```
UI-service, business-logic-service, data-service
```
Каждый запрос идёт через все три → распределённый монолит, chained calls, latency.

Правильное — по **бизнес-возможностям / доменам**:
```
Orders, Payments, Inventory, Shipping, Notifications
```
Каждый сервис = **complete vertical slice** (UI-hooks + business logic + own DB).

---

## 2. DDD — Domain-Driven Design

Методология, помогающая найти границы.

Автор — Eric Evans (книга «Domain-Driven Design», 2003). Модна снова с приходом микросервисов.

### 2.1 Ключевые понятия

- **Domain** — область бизнеса (e-commerce, банкинг, налоги).
- **Subdomain** — часть домена (Orders, Inventory, Payment внутри e-commerce).
- **Bounded Context** — граница языка/модели. Внутри — одна модель, снаружи — другая.
- **Ubiquitous Language** — общий язык между разработчиками и бизнесом внутри одного контекста.

### 2.2 Bounded Context

Ключевая идея.

Одна сущность может значить **разное** в разных контекстах:
- В **Sales** контексте `Product` = имя, цена, картинка, описание.
- В **Warehouse** контексте `Product` = SKU, вес, размер, местоположение.
- В **Accounting** контексте `Product` = стоимость, налоговая ставка, cost center.

Пытаться сделать один `Product` со всеми полями — **God object**. Правильно — **три отдельных класса Product** в трёх сервисах.

Bounded context = **естественная граница микросервиса**.

### 2.3 Aggregate

**Aggregate** — консистентный кластер объектов, обрабатываемых как единое целое.

Пример: `Order` + `OrderItems`. Всегда сохраняются вместе, всегда валидируются вместе.

Правила:
- Один **aggregate root** — сущность, через которую доступ ко всему остальному.
- Транзакция = один aggregate.
- Между aggregates — только по ID (references), не по object references.

### 2.4 В ИСНА

Bounded contexts (грубо):
- **КНП** — Kabinet Nalogoplatelshika (пользовательский portal).
- **ФНО** — формы налоговой отчётности.
- **ФО** — формы отчётности.
- **АРМ (tax-rep)** — рабочее место инспектора.
- **ЕАЭС** — ЕАЭС-контур.
- **NZ** — уведомления.

Внутри каждого — свой язык, своя модель, свои микросервисы.

---

## 3. Декомпозиция стратегии

### 3.1 By business capability

Каждый сервис = отдельная бизнес-функция.

Пример e-commerce:
- Order Management.
- Product Catalog.
- Inventory.
- Pricing.
- Payment.
- Shipping.
- Customer Management.
- Notifications.
- Recommendations.

### 3.2 By subdomain (DDD-based)

Похоже, но через bounded contexts. Более осмысленное разделение.

### 3.3 By actor / user type

Например: `admin-service`, `customer-service`, `partner-service`.

### 3.4 By volatility

- Часто меняющиеся куски — отдельные (гибкий deploy).
- Стабильные — можно вместе.

### 3.5 Anti-patterns

- **По слоям** (UI/BL/Data).
- **CRUD-per-entity** (микро-микросервисы для каждой таблицы).
- **Технически / географически** (не по бизнесу).

---

## 4. Database per service

Обсуждали в `48-monolith-vs-microservices.md`. Ключевое:

- Каждый сервис — **своя схема** (минимум).
- Идеально — **своя БД**.
- Нельзя читать/писать в чужую БД напрямую.
- Только через API/events.

### 4.1 Проблема: shared data

Customer info нужен в Orders, Payments, Shipping.

Решения:
- **API calls** — синхронно спросить customer-service. Проблема: coupling + latency.
- **Data duplication** — каждый сервис держит свою копию нужных полей customer. Обновление через events.
- **CDC** (Change Data Capture) — Debezium читает WAL customer-service → пишет в Kafka → другие сервисы обновляют кэш.

**Правило**: **чуть-чуть дублирования** — норма для микросервисов.

---

## 5. Communication patterns

### 5.1 Sync — REST / HTTP

**Стандарт** для request-response.

Плюсы:
- Простота.
- Всем понятно.
- Легко debug.

Минусы:
- Tight coupling (caller ждёт callee).
- Cascade failures.
- Latency (network + serialization).

Использование:
- Query data.
- Immediate response requests.
- Между UI-backend и микросервисами.

### 5.2 Sync — gRPC

Google RPC. Основан на HTTP/2 + Protobuf.

Плюсы:
- Быстрее REST (binary + HTTP/2 multiplexing).
- Типобезопасно (schema).
- Streaming.

Минусы:
- Сложнее debug (binary).
- Не работает из browser'а напрямую.
- Требует shared schemas.

Использование: internal service-to-service, high-throughput.

### 5.3 Sync — GraphQL

Для UI backend — гибкие запросы. Клиент говорит какие поля нужны.

Плюсы:
- Клиент управляет data shape.
- Меньше over/under fetching.

Минусы:
- Сложность backend.
- N+1 проблема (нужен DataLoader).
- Кэширование сложнее REST.

Использование: **BFF** (Backend for Frontend) для сложных UI.

### 5.4 Async — Message broker

Rabbit / Kafka / SQS / NATS.

Producer шлёт event → broker → Consumer(s).

Плюсы:
- **Decoupling** — producer не знает о consumer.
- **Buffering** — пик нагрузки поглощается.
- **Multiple consumers** — pub-sub.
- **Retry / durability**.

Минусы:
- Eventual consistency.
- Сложнее debug (async trace).
- Требует broker infrastructure.

Использование: **предпочтительно между backend сервисами**. Events для state propagation.

### 5.5 Async — Event streaming

Kafka. **Event log** — можно replay.

Плюсы:
- Event sourcing.
- Analytics.
- Multiple consumers, каждый со своим offset.

### 5.6 Правило (Sam Newman, Chris Richardson)

- Sync REST только для **outer edge** (UI → gateway → services).
- Async events между **backend services**.
- gRPC internal high-throughput.
- Minimize sync chains (A → B → C → D → E — плохо).

---

## 6. API Gateway pattern

**Единая точка входа** для внешних клиентов.

```
Клиенты  ─────► [API Gateway]  ─────►  services
                       │
                       ├─ routing
                       ├─ auth
                       ├─ rate limit
                       ├─ SSL termination
                       ├─ logging / metrics
                       └─ response aggregation
```

### 6.1 Зачем

- Скрыть внутреннюю топологию.
- Централизованная auth.
- Rate limiting.
- Cross-cutting concerns (logging, tracing).
- Aggregation (собрать данные из нескольких сервисов в один response).

### 6.2 Реализации

- **Kong** — популярный open-source.
- **Envoy** — modern, service mesh backbone.
- **Spring Cloud Gateway** — на Spring.
- **Zuul 1 / 2** (Netflix) — legacy.
- **AWS API Gateway** — managed.
- **nginx** — простой gateway.

**В ИСНА**: `isna-knp-gateway` = **Zuul 1** (Java 11, memory `knp-gateway-no-java21`).

### 6.3 Кавет

Gateway = **single point of failure** для внешнего трафика. HA обязательна.

Не превращать в бизнес-логику. Только routing/cross-cutting.

---

## 7. BFF — Backend For Frontend

Отдельный API Gateway для **каждого типа клиента**:
- BFF for Web.
- BFF for Mobile.
- BFF for Partners.

```
Web UI       →  BFF-Web    ─┐
Mobile app   →  BFF-Mobile ─┼──►  Services
Partners     →  BFF-API    ─┘
```

Каждый BFF:
- Оптимизирует API под свой клиент.
- Aggregate запросы.
- Разные data shapes.

Плюсы:
- Клиент-специфичная оптимизация.
- Меньше over/under fetching.
- Frontend команды владеют своим BFF.

Минусы:
- Дублирование логики.
- Больше сервисов.

Использование: когда клиенты сильно отличаются.

---

## 8. Strangler Fig pattern

Уже упомянул в `48-monolith-vs-microservices.md`. Здесь глубже.

Название от **strangler fig** — растение, обвивающее дерево и постепенно его убивающее.

### 8.1 Схема

Шаг 0: Legacy monolith.
```
Client → Monolith
```

Шаг 1: API Gateway перед монолитом.
```
Client → Gateway → Monolith
```

Шаг 2: Выделяем **Feature A** в новый microservice.
```
Client → Gateway → { Feature A → New Service }
                  { Everything else → Monolith }
```

Шаг 3-N: постепенно другие features.

Шаг Final: Monolith пустой → удаляем.

### 8.2 Практические советы

- Начинай с **stable** features (не под активной разработкой).
- Или наоборот — **самые проблемные** (получаешь value быстро).
- **Не переписывай 1-в-1** — используй возможность улучшить.
- **Data migration** — сложный шаг, может занять больше кода.
- **Rollback** должен работать в каждый момент.

---

## 9. Sidecar pattern

**Sidecar** — контейнер, добавляемый в pod рядом с основным.

Пример: **service mesh** (Istio, Linkerd) добавляет **Envoy** proxy как sidecar. Envoy перехватывает весь traffic → добавляет mTLS, retry, circuit breaker, metrics.

Плюсы:
- Cross-cutting concerns вне приложения.
- Language-agnostic.
- Централизованное управление.

Минусы:
- Overhead (extra процесс).
- Дополнительная complexity.

---

## 10. Ambassador pattern

Похож на sidecar, но для **outbound** communication.

Ambassador proxy обрабатывает исходящие вызовы: retry, load balancing, service discovery.

Клиент вызывает `localhost:9999`, ambassador разбирается.

Часто = sidecar Envoy в service mesh.

---

## 11. Anti-corruption Layer (ACL)

Изолирует **новый чистый bounded context** от **legacy**.

```
[Clean new service] → [ACL] → [Ugly legacy]
                         │
                         └─ переводит терминологию legacy в новую
```

Если бы новый сервис напрямую вызывал legacy — заразился бы legacy-концепциями.

ACL — переводчик. Новый сервис знает только clean model.

---

## 12. API versioning

Как менять API без ломки клиентов.

### 12.1 URI versioning

```
/v1/orders
/v2/orders
```

Простой, видимый.

### 12.2 Header versioning

```
Accept: application/vnd.myapi.v2+json
```

Чище URL, но менее видимо.

### 12.3 Query param

```
/orders?version=2
```

Гибко, но некрасиво.

### 12.4 Backward compatibility правила

- **Adding** поля — safe (клиенты игнорируют).
- **Removing** — breaking.
- **Changing** тип / semantics — breaking.
- **Renaming** — breaking.

Правило: **добавлять, не удалять**. Deprecated пометки, удаление через major version.

### 12.5 Consumer-driven contracts

**Spring Cloud Contract**, **Pact** — producer генерирует stubs, consumer использует. Гарантия что producer не сломал contract.

---

## 13. Service discovery

Как сервисы находят друг друга.

- **Client-side** — Consul, Eureka. Клиент сам выбирает инстанс.
- **Server-side** — LB (K8s Service, nginx). Клиент шлёт на VIP.

См. файлы `11-consul-detailed.md`, `31-load-balancer.md`.

---

## 14. Configuration management

- **Config files** (application.yml) — базово.
- **Env variables** — для 12-factor apps.
- **Spring Cloud Config** — централизованный сервер.
- **Consul KV / etcd**.
- **Kubernetes ConfigMap / Secret**.
- **HashiCorp Vault** — секреты.

Правило: код одинаковый для всех env, отличается только configuration.

---

## 15. Distributed tracing

Уже обсуждали в `36-spring-cloud.md`. Один запрос идёт через N сервисов — как отследить?

**TraceId + SpanId** — прокидываются через все сервисы (HTTP headers, Kafka headers, MDC).

Экспорт в **Zipkin / Jaeger / OpenTelemetry** — визуализация.

---

## 16. Централизованный logging

Обсуждали в `38-logging.md`. Каждый сервис → JSON logs → Filebeat → **ELK**.

Correlation ID для поиска цепочки.

---

## 17. Distributed data patterns (краткое напоминание)

Разберём подробно в `50-saga-pattern.md`.

- **Saga** — distributed tx через compensation.
- **Outbox** — atomic DB commit + publish (следующий файл 51).
- **CQRS** — split read/write.
- **Event Sourcing** — хранить events.
- **CDC** — Change Data Capture (Debezium).

---

## 18. Best practices

1. **Bounded context** для границ сервисов.
2. **Database per service**.
3. **Async events** > sync между backend.
4. **API Gateway** для внешних клиентов.
5. **Circuit Breakers + timeouts + retry** everywhere (см. `52-microservices-resilience.md`).
6. **Distributed tracing** обязательно.
7. **CI/CD per service** независимо.
8. **Backward compatible** APIs.
9. **Contract testing** (Pact/Spring Cloud Contract).
10. **Idempotency** для всех write ops.

---

## 19. Собесные вопросы

1. **Что такое bounded context?** — Граница модели/языка; естественная граница микросервиса.
2. **Что такое aggregate в DDD?** — Кластер объектов, транзакционно консистентный; один aggregate root.
3. **Как декомпозировать монолит?** — По бизнес-возможностям / bounded context, не по слоям.
4. **Что такое API Gateway?** — Единая точка входа для клиентов; routing, auth, rate limit.
5. **Что такое BFF?** — Backend for Frontend; отдельный gateway на каждый тип клиента.
6. **Sync vs Async communication?** — Sync (REST/gRPC) для immediate; async (events) для state propagation.
7. **Что такое Strangler pattern?** — Постепенное вытеснение legacy микросервисами через gateway.
8. **Что такое CQRS?** — Разделение read и write моделей.
9. **Event sourcing?** — Хранить events, не state; audit + replay.
10. **Что такое CDC?** — Change Data Capture — читать WAL БД → публиковать events (Debezium).
11. **Service mesh — зачем?** — Sidecar proxy (Envoy) для cross-cutting concerns (mTLS, retry, metrics).
12. **Как версионировать API?** — URI (/v1, /v2), header, query. Backward compatible additions only.
13. **Что такое contract testing?** — Producer генерирует stubs; consumer тестируется против stubs; Pact/Spring Cloud Contract.
14. **Database per service — почему?** — Изоляция schema, свобода эволюции; требует Saga/Outbox для consistency.
15. **Как избежать shared data problem?** — API calls (coupling), data duplication (events), CDC.

---

## Итог

- **DDD + bounded context** = основа для границ.
- **По бизнес-возможностям**, не по техническим слоям.
- **Database per service** — правило.
- **Async events** предпочтительно; sync только где нужно.
- **API Gateway** для внешнего входа; **BFF** для разных клиентов.
- **Strangler** для миграции монолита.
- **Sidecar / Ambassador / ACL** — architectural patterns.
- **Contract testing** + **distributed tracing** обязательны.

Следующий — `50-saga-pattern.md`.
