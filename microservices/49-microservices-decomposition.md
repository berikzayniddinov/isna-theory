# 49. Микросервисы: decomposition, DDD, communication patterns

## Зачем нужен systematic подход к decomposition

Разработчик которому дали задачу «сделать микросервисы» обычно начинает делить систему очевидным способом. UI сервис. Business logic сервис. Data сервис. Или по entity — User service, Order service, Product service. Каждая CRUD-сущность превращается в отдельный микросервис. Результат — distributed monolith в лучшем случае. В худшем — chaotic набор сервисов requiring constant coordination.

Разница между разработчиком «делящим на микросервисы» и «понимающим decomposition» проявляется через несколько лет operations. Первый заканчивает с 50 микросервисами tightly coupled — изменение business flow requires changes across 10 services deployed together. Второй знает про Domain-Driven Design bounded contexts — natural boundaries следуют business language differences. Знает что Product в Sales context отличается от Product в Warehouse context — это два разных aggregates в two разных services, не one shared entity forced в service boundary. Знает что sync HTTP chains anti-pattern — async events через bounded contexts правильный подход для inter-service communication.

В этом файле разберём decomposition through systematic lens. Проблема decomposition — где границы. DDD как methodology для finding boundaries. Bounded contexts key concept. Aggregates и их role. Decomposition strategies detailed. Database per service implications. Communication patterns comprehensively. API Gateway pattern. BFF for different clients. Strangler for legacy migration. Sidecar plus Ambassador patterns. Anti-corruption Layer для integration с legacy. API versioning стратегии. Service discovery. Configuration management. Distributed tracing. Best practices real-world proven.

## Проблема: где границы сервисов

Плохое решение по техническим слоям:
```
UI-service, business-logic-service, data-service
```

Каждый запрос идёт через все три — distributed monolith, chained sync calls, cumulative latency. Change в UI feature requires coordinated deploy across all layers. Boundaries совершенно неправильные — они following architectural pattern не business reality.

Правильное — по business capabilities или domains:
```
Orders, Payments, Inventory, Shipping, Notifications
```

Каждый сервис = complete vertical slice (UI hooks + business logic + own DB). Independent evolution. Business feature change contained within one или few services.

Vertical vs horizontal decomposition. Horizontal splits by technical layer. Vertical splits by business capability. Микросервисы should be vertical. Layers within microservice могут exist (controller, service, repository) но crossing service boundary should be business-driven.

## DDD как methodology

Domain-Driven Design разработан Eric Evans в 2003 году. Стал критически важен снова с приходом микросервисов. Provides systematic approach к finding meaningful boundaries.

Ключевые concepts.

Domain — область бизнеса. E-commerce, banking, taxes. High-level scope of business problem being solved.

Subdomain — часть domain. Внутри e-commerce — orders, inventory, payments, shipping, marketing. Разбиение большого domain на manageable pieces.

Bounded Context — граница языка и модели. Внутри — одна модель, снаружи — другая. Ключевая concept.

Ubiquitous Language — общий язык между разработчиками и бизнесом внутри одного контекста. Terms mean same thing internally. Between contexts могут differ.

Bounded Context — critical concept для understanding.

Одна сущность может значить разное в разных контекстах:
- В Sales контексте Product = имя, цена, картинка, описание.
- В Warehouse контексте Product = SKU, вес, размер, местоположение.
- В Accounting контексте Product = стоимость, налоговая ставка, cost center.

Пытаться сделать один Product со всеми полями — God object. Grows uncontrollably. Different teams need different aspects. Changes ripple everywhere.

Правильно — три отдельных класса Product в трёх сервисах. Каждый bounded context has own model reflecting local needs. Communication между contexts через explicit contracts, не shared entities.

Bounded context = естественная граница микросервиса. Not always one-to-one — bounded context sometimes contains multiple microservices если internal complexity warrants. But service boundary should never cross bounded context — creates coupling.

Aggregate — консистентный кластер объектов обрабатываемых как единое целое.

Пример. Order plus OrderItems. Always saved together, validated together, considered as one atomic unit. Business rules apply к whole aggregate.

Правила aggregates. Один aggregate root — сущность, через которую доступ ко всему остальному. Транзакция = один aggregate — сохранение atomic на уровне aggregate. Между aggregates — только по ID (references), не по object references — prevents transactional coupling.

Aggregate size guidance. Small aggregates preferred. Contain only what needs to change together atomically. Large aggregates — lock contention, complex validation, hard-to-reason concurrent modifications.

В КНП bounded contexts (примерно):
- КНП — Kabinet Nalogoplatelshika (пользовательский portal).
- ФНО — формы налоговой отчётности.
- ФО — формы отчётности.
- АРМ (tax-rep) — рабочее место инспектора.
- ЕАЭС — ЕАЭС-контур.
- NZ — уведомления.

Внутри каждого — свой язык, своя модель, свои микросервисы. Разбиение исторически happened based on business domains. Fits DDD naturally.

## Стратегии decomposition

By business capability. Каждый сервис = отдельная бизнес-функция. Мышление в терминах what business does not how technically implemented.

Пример e-commerce. Order Management. Product Catalog. Inventory. Pricing. Payment. Shipping. Customer Management. Notifications. Recommendations. Каждая capability owned by team. Independent evolution.

By subdomain (DDD-based). Более осмысленное разделение. Uses DDD subdomain analysis. Bounded contexts drive boundaries. More rigorous than raw capability listing.

By actor / user type. Разделение по типу пользователя. admin-service, customer-service, partner-service. Sometimes appropriate когда different user types have completely different workflows. Often less clean than capability-based.

By volatility. Часто меняющиеся куски — отдельные (гибкий deploy). Стабильные — можно вместе. Optimization strategy — decompose parts requiring frequent iteration.

Anti-patterns decomposition.

По слоям (UI/BL/Data) — distributed monolith. Every user action requires calls across all layers. Not independent evolution.

CRUD-per-entity — micro-микросервисы для каждой таблицы. Chain of sync calls для business operations. High coordination overhead.

Технически или географически — по региону, DC. Не business-driven. Doesn't reflect actual system evolution needs.

## Database per service

Правило микросервисов. Каждый сервис — своя schema (минимум). Идеально — своя БД. Нельзя читать/писать в чужую БД напрямую. Только через API или events.

Проблема shared data. Customer info нужен в Orders, Payments, Shipping. Как handling без violating database-per-service?

Решения.

API calls — синхронно спросить customer-service при each need. Проблема — coupling plus latency. Каждый order display requires call к customer-service. Load pattern amplified.

Data duplication — каждый сервис держит свою копию нужных полей customer. Update через events (CustomerUpdated event → all subscribers update local copies). Slight data staleness acceptable. Autonomous services.

CDC (Change Data Capture) — Debezium читает WAL customer-service → пишет в Kafka → другие сервисы обновляют кэш. Automated propagation без explicit event publishing.

Правило. Чуть-чуть дублирования — норма для микросервисов. Not violation DRY (Don't Repeat Yourself) — dublication for independence. Trade autonomy для minor storage overhead.

## Communication patterns comprehensively

Sync via REST/HTTP — стандарт. JSON payloads. Всем понятно.

Плюсы. Простота. Legacy compatibility. Universal tooling.

Минусы. Tight coupling — caller ждёт callee. Cascade failures. Latency (network plus serialization). Blocking threads waiting for response.

Использование. Query data. Immediate response requests. Между UI-backend и микросервисами (outer edge).

Sync via gRPC. Google RPC. HTTP/2 based. Protobuf serialization.

Плюсы. Быстрее REST (binary plus HTTP/2 multiplexing). Типобезопасно (schema). Streaming support (bidirectional streams). Auto-generated clients для multiple languages.

Минусы. Сложнее debug (binary format). Не работает из browser'а напрямую (нужен gRPC-Web proxy). Требует shared schemas managed carefully.

Использование. Internal service-to-service где performance critical. High-throughput internal APIs. Streaming scenarios.

Sync via GraphQL. Для UI backend — гибкие запросы. Клиент говорит какие поля нужны.

Плюсы. Клиент управляет data shape. Меньше over/under fetching. Single endpoint.

Минусы. Сложность backend implementation. N+1 проблема без DataLoader. Кэширование сложнее REST.

Использование. BFF (Backend for Frontend) для сложных UI. Rich mobile applications с многими different views over same data.

Async via Message broker (Rabbit/Kafka/SQS/NATS). Producer шлёт event → broker → Consumer(s).

Плюсы. Decoupling — producer не знает о consumer. Buffering — пик нагрузки поглощается. Multiple consumers — pub-sub natural. Retry plus durability built into broker.

Минусы. Eventual consistency. Сложнее debug (async traces). Требует broker infrastructure.

Использование. Предпочтительно между backend сервисами. Events для state propagation. Fire-and-forget notifications.

Async via Event streaming (Kafka). Event log — можно replay. Multiple consumers each со своим offset.

Плюсы. Event sourcing enable. Analytics pipelines. Multiple consumers each со своим processing.

Использование. Event-sourced systems. Analytics workloads. Historical event replay для new consumers.

Правило (Sam Newman, Chris Richardson). Sync REST только для outer edge (UI → gateway → services). Async events между backend services. gRPC internal high-throughput. Minimize sync chains (A → B → C → D → E — плохо).

## API Gateway pattern

Единая точка входа для внешних клиентов:
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

Зачем.

Скрыть внутреннюю топологию. Клиенты не знают which services существуют. Refactoring services transparent для клиентов.

Централизованная auth. Один place для authenticating requests. Services trust gateway's authentication.

Rate limiting. Уровень gateway easier to configure than in each service.

Cross-cutting concerns. Logging, tracing, metrics — implemented once в gateway.

Aggregation. Собрать данные из нескольких сервисов в один response. UI request satisfied one API call вместо multiple.

Реализации. Kong — популярный open-source. Envoy — modern, service mesh backbone. Spring Cloud Gateway — на Spring. Zuul 1 / 2 (Netflix) — legacy. AWS API Gateway — managed. nginx — простой gateway.

В КНП — isna-knp-gateway = Zuul 1 (Java 11, memory knp-gateway-no-java21). Legacy tech. Migration planned но complex.

Caveat. Gateway = single point of failure для внешнего трафика. HA обязательна — multiple instances, load balanced.

Не превращать в бизнес-логику. Только routing и cross-cutting concerns. Business logic в services. Gateway thin.

## BFF: Backend For Frontend

Отдельный API Gateway для каждого типа клиента:
- BFF for Web.
- BFF for Mobile.
- BFF for Partners.

Каждый BFF оптимизирует API под свой клиент. Aggregate запросы appropriately для that client type. Разные data shapes.

Плюсы. Клиент-специфичная оптимизация. Меньше over/under fetching. Frontend команды владеют своим BFF.

Минусы. Дублирование логики между BFFs. Больше сервисов to manage.

Использование. Когда клиенты сильно отличаются (rich desktop web vs mobile with limited data). Simple API sufficient — one gateway достаточно.

## Strangler Fig pattern

Уже упомянут в файле 48. Здесь глубже. Название от strangler fig — растение обвивающее дерево и постепенно его убивающее.

Шаг 0. Legacy monolith serving traffic.

Шаг 1. API Gateway перед monolith:
```
Client → Gateway → Monolith
```

Шаг 2. Выделяем Feature A в новый microservice:
```
Client → Gateway → { Feature A → New Service }
                  { Everything else → Monolith }
```

Шаг 3-N. Постепенно другие features migrate.

Final. Monolith пустой — удаляем.

Практические советы.

Начинай с stable features (не под активной разработкой). Or наоборот — самые проблемные (получаешь value быстро от improvements).

Не переписывай 1-в-1. Migration opportunity для improving architecture, cleaning tech debt.

Data migration — сложный шаг. Часто занимает больше кода чем service logic itself. Plan carefully.

Rollback должен работать в каждый момент. Gateway routing allows quick reversal — flip traffic обратно к monolith при issues.

Timeline realistic. Significant migrations take 2-5 years. Overnight rewrites for large monoliths не realistic.

## Sidecar pattern

Sidecar — контейнер добавляемый в pod рядом с основным. Shares network, IPC, volumes с main container.

Пример service mesh (Istio, Linkerd) добавляет Envoy proxy как sidecar. Envoy перехватывает весь traffic — добавляет mTLS, retry, circuit breaker, metrics.

Плюсы. Cross-cutting concerns вне приложения. Language-agnostic — same sidecar works с Java, Python, Go services. Централизованное управление через platform.

Минусы. Overhead (extra процесс per pod). Дополнительная complexity troubleshooting.

Полезно когда consistent infrastructure concerns нужны across polyglot services. Enterprise Kubernetes deployments часто используют service mesh sidecars.

## Ambassador pattern

Похож на sidecar но для outbound communication.

Ambassador proxy обрабатывает исходящие вызовы — retry, load balancing, service discovery.

Клиент вызывает localhost:9999, ambassador разбирается со сложностью outbound routing.

Часто = sidecar Envoy в service mesh. Terminology overlaps — implementation similar, concept differentiated by direction.

## Anti-Corruption Layer

ACL изолирует новый чистый bounded context от legacy:
```
[Clean new service] → [ACL] → [Ugly legacy]
                         │
                         └─ переводит терминологию legacy в новую
```

Если бы новый сервис напрямую вызывал legacy — заразился бы legacy-концепциями. Ugly names, weird semantics, historical baggage would leak в clean new codebase.

ACL — переводчик. Новый сервис знает только clean model. ACL adapts к legacy API. Isolation preserved.

Common pattern при Strangler migration. Legacy monolith not disappearing overnight. New services need to communicate с it. ACL wraps legacy interactions cleanly.

## API versioning

Как менять API без ломки клиентов. Multiple approaches.

URI versioning:
```
/v1/orders
/v2/orders
```

Простой, видимый в logs, easy to route.

Header versioning:
```
Accept: application/vnd.myapi.v2+json
```

Чище URL. Content negotiation через HTTP semantics. Less visible в quick log inspection.

Query param:
```
/orders?version=2
```

Гибко, но не clean. Some argue anti-pattern.

Backward compatibility rules.

Adding fields — safe. Клиенты старой версии ignore new fields.

Removing fields — breaking. Old clients expect these.

Changing type или semantics — breaking. Same field name с different meaning особенно opsсно.

Renaming — breaking. Same as remove plus add.

Правило. Добавлять, не удалять. Deprecated пометки для fields going away. Удаление через major version bump. Long deprecation windows.

Consumer-driven contracts. Spring Cloud Contract, Pact — producer генерирует stubs, consumer использует. Гарантия что producer не сломал contract expected clients.

## Service discovery

Как сервисы находят друг друга. Two main approaches.

Client-side — Consul, Eureka. Клиент сам знает про all instances, выбирает один по algorithm. Каждый service instance registers на startup. Clients query registry, cache locally. Load balancing decided at client. See file 11 for Consul details.

Server-side — LB (K8s Service, nginx). Клиент шлёт на VIP (Virtual IP). LB behind picks actual instance. Simpler для clients. Central LB может become bottleneck. See file 31 for load balancer details.

## Configuration management

Config files (application.yml) — базово. Static config compiled с service.

Env variables — 12-factor apps style. Config injected via environment. Different values per deployment без rebuild.

Spring Cloud Config — централизованный сервер. Git-backed configuration. Refresh при updates. See file 36 for details.

Consul KV / etcd — distributed KV stores. Dynamic configuration. Watched через listeners для immediate updates.

Kubernetes ConfigMap / Secret — K8s-native. Mounted as files или env vars в pods. Managed через K8s API.

HashiCorp Vault — secrets management. Encrypted at rest. Access controlled через policies. Rotation supported.

Правило. Код одинаковый для всех environments. Configuration изменяется. Enables reproducible deployments.

## Distributed tracing

Уже обсуждали в файле 36. Один запрос идёт через N сервисов — как отследить?

TraceId plus SpanId прокидываются через все сервисы (HTTP headers, Kafka headers, MDC). Каждый service creates spans для work done. Parent-child relationships between spans form trace tree.

Экспорт в Zipkin / Jaeger / OpenTelemetry — visualization tools. Full request path visible. Timing breakdown per service. Errors correlated с specific spans.

Sampling для reducing overhead. 100% tracing expensive для high-volume services. 10% sampling captures patterns без overwhelming infrastructure.

## Централизованное logging

Обсуждали в файле 38. Каждый сервис → JSON logs → Filebeat → ELK.

Correlation ID (traceId) в logs enables tracing plus logs correlation. Search by traceId shows all logs of specific request across services.

Structured JSON format преферентен для machine parsing. Easier to query in Kibana. Less error-prone than regex parsing text logs.

## Distributed data patterns — brief

Разберём подробно в файле 50.

Saga — distributed tx через compensation. Долгие процессы crossing multiple services.

Outbox — atomic DB commit + publish (файл 51). Reliable event publishing.

CQRS — split read/write. Different models для different access patterns.

Event Sourcing — хранить events. Full history plus replay.

CDC — Change Data Capture (Debezium). Database changes → events без explicit publishing.

## Best practices

Bounded context для границ сервисов. DDD analysis provides principled boundaries.

Database per service. Autonomy плюс independent evolution.

Async events преферентно между backend. Sync только через API Gateway для outer edge.

API Gateway для внешних клиентов. Cross-cutting concerns centralized.

Circuit Breakers, timeouts, retry everywhere. See file 52 for resilience patterns.

Distributed tracing обязательно. Without it — impossible troubleshoot distributed systems.

CI/CD per service независимо. Team autonomy. Fast iteration.

Backward compatible APIs. Deprecation cycles. Never break clients without notice.

Contract testing (Pact или Spring Cloud Contract). Catches breaking changes early.

Idempotency для всех write ops. Retry-safe operations. Handle duplicates gracefully.

## Итоги

Decomposition решает проблему где границы сервисов. Wrong boundaries → distributed monolith. Right boundaries → autonomous services.

DDD provides methodology. Bounded contexts natural boundaries. Ubiquitous language plus aggregates guide design decisions.

Decomposition strategies. By business capability. By subdomain (DDD-based). By actor. By volatility. Not by technical layers or per-entity.

Database per service — autonomy imperative. Shared data managed через API calls, duplication, или CDC.

Communication. Sync REST outer edge. gRPC high-performance internal. GraphQL rich UI. Async events preferred между backend services.

API Gateway centralizes cross-cutting concerns. Hides internal topology. BFF variant для different client types.

Strangler pattern для gradual migration legacy к microservices. Incremental replacement.

Sidecar/Ambassador patterns для infrastructure concerns via co-located proxies. Service mesh through sidecars.

Anti-Corruption Layer isolates new services от legacy semantics.

API versioning через URI, headers, или query params. Backward compatibility rules. Contract testing enforces stability.

Configuration through env vars, config servers, K8s ConfigMap, or Vault. Code identical across envs.

Distributed tracing plus centralized logging обязательны. Without observability — cannot operate distributed systems.

Best practices — bounded contexts, database per service, async events, API Gateway, resilience patterns, contract testing, idempotency. Все mutually reinforcing.

Дальше — Saga pattern deeply для distributed transactions across services.
