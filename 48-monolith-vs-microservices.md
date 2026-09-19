# 48. Монолит vs Микросервисы: когда что выбирать

## Зачем понимать trade-offs глубоко

Разработчик молодой команды обычно видит две крайности в популярной культуре. Монолит — legacy, medieval, «плохо». Микросервисы — modern, scalable, «правильно». Реальность fundamentally другая. Обе архитектурные модели имеют свои strengths plus weaknesses. Правильный выбор depends на specific context. Wrong choice может damage project irreparably.

Разница между разработчиком «знающим монолит и микросервисы» и «понимающим тrade-offs» проявляется в architectural decisions. Первый пропагандирует микросервисы потому что «modern». Второй знает что startup с 5 человек в 20 микросервисах — recipe для disaster. Понимает что distributed system complexity вводит classes проблем не существовавших в monolith — network unreliability, partial failures, distributed transactions, eventual consistency, harder debugging. Знает что modular monolith — often optimal starting point который позволяет позже decompose если реально нужно. Знает что distributed monolith — antipattern combining worst из обоих worlds.

В этом файле разберём trade-offs глубоко. Что fundamentally такое monolith и микросервисы architecturally. Реальный ИСНА как example микросервисной архитектуры. Detailed advantages plus disadvantages каждой модели. Distributed monolith antipattern — как избежать. Modular monolith как compromise. Когда что выбирать основано на team size, technical maturity, requirements. Communication patterns между микросервисами. Strangler pattern для migration. Data management (database per service, shared database antipattern, CQRS, event sourcing). Deployment considerations. Common mistakes при architectural choices.

## Монолит: одно приложение всё

Монолит = one application, one process, one codebase, one deploy. Всё бизнес-логика внутри одного process. Модули общаются через function calls в памяти:
```
┌─────────── Monolith app.war ───────────┐
│                                         │
│  ┌─────┐ ┌─────────┐ ┌────────┐ ┌────┐ │
│  │ UI  │ │ Orders  │ │Payment │ │... │ │
│  └──┬──┘ └────┬────┘ └───┬────┘ └─┬──┘ │
│     │        │           │        │    │
│     └────────┴───┬───────┴────────┘    │
│                  │  function calls      │
│                  ▼                       │
│           ┌──────────┐                  │
│           │   ORM    │                  │
│           └────┬─────┘                  │
└────────────────┼────────────────────────┘
                 │
                 ▼
          [Single Database]
```

Deploy simple — собрал WAR, положил в Tomcat. Одно приложение — один Tomcat. Configuration в одном месте. Logs в одном файле. Database schema одна.

Historically dominant architecture. Rails, Django, Spring MVC приложения classic monoliths. Netflix started как monolith. Amazon started как monolith. Modern microservices proponents often forget что их applications эволюционировали от successful monoliths.

## Микросервисы: множество маленьких сервисов

Microservices = много маленьких сервисов, каждый:
- Свой процесс (обычно в контейнере).
- Своя кодовая база / repo.
- Свой deploy.
- Своя (часто) БД.
- Общается с другими через сеть (HTTP / gRPC / messaging).

```
┌──── Service A ────┐    ┌──── Service B ────┐   ┌──── Service C ────┐
│  Orders logic    │◄──►│  Payment logic   │◄──►│ Notification logic│
│  own DB          │    │  own DB          │    │  own DB          │
└──────────────────┘    └──────────────────┘    └──────────────────┘
        │                       │                       │
        └─────────┬─────────────┴───────────┬───────────┘
                  │                         │
                  ▼                         ▼
             [Message broker]          [API Gateway]
```

Deploy independent per service. git push в orders repo → CI builds orders → K8s deploy только orders. Другие services untouched.

Emerged from experience большие web companies — Netflix, Amazon, Uber. Scale requirements exceeded monolith capabilities. Different concerns need different technology choices. Multiple teams need independent deployment cycles. Microservices это response на these needs.

## ИСНА как пример микросервисной архитектуры

Из memory concepts real ИСНА microservices:
- `isna-knp-integration` — приёмка ФНО из КНП в tax-rep.
- `isna-knp-user` — пользователи, permissions.
- `isna-knp-fno` — работа с ФНО.
- `isna-knp-notification` — уведомления.
- `isna-knp-fs` — файловое хранилище.
- `isna-knp-gateway` — API-gateway (Zuul, Java 11).
- `isna-fno` — сервис форм налоговой отчётности.
- `isna-fo` — формы отчётности.
- `tax-report`, `tax-rep` — АРМ налогового органа.

И ещё десятки. Total около 30-50 микросервисов в production.

Communication через:
- Feign clients (sync HTTP через Consul для service discovery).
- RabbitMQ (async messaging для events).

Классическая микросервисная архитектура. Каждый сервис — отдельный git repo, отдельный CI/CD pipeline, отдельный K8s Deployment, часто своя БД (или своя schema в shared DB).

Real-world microservices deployment имеет свои caveats — memory кейсы вроде knp-gateway-no-java21 показывают что gateway остаётся на Java 11 потому что Zuul 1 не совместим с Java 17+. Migration к Spring Cloud Gateway долгосрочный проект.

## Монолит: плюсы

Простота разработки. Один проект, один IDE-workspace. Refactoring across modules trivial — IDE refactoring tools work universally. Compile-time errors detected при build.

Простота deploy. Один WAR/JAR. Deploy pipeline linear. Rollback простой — deploy previous version.

Простота отладки. Stack traces span entire application. Breakpoints work через все modules. Time-travel debugging feasible. IDEs support monoliths first-class.

ACID transactions. Одна DB — one transaction spans multiple modules easily. Atomicity guaranteed by database.

Быстрая коммуникация. Function call ~1 ns. Compared к HTTP call 1-10 ms — 10 million times faster для inter-module communication. Massive advantage для internal operations.

Меньше инфраструктуры. Одна БД, одна JVM, один процесс. Monitoring простой. No message broker, no service mesh, no distributed tracing infrastructure required.

Меньше overhead для маленькой команды. 5 developers могут comfortably work на one monolith. No infrastructure specialists needed для microservices tooling.

## Монолит: минусы

Scale только целиком. Не можешь скейлить только payment logic независимо. Если payment под нагрузкой — scale весь monolith. Wasteful ресурсов.

Один падает — всё падает. Single process failure kills всё functionality. NPE в notification module crashes JVM — user login тоже гибнет. Single point of failure per process.

Долгий deploy. Маленькое изменение → пересобрать всё WAR → deploy → restart. Downtime plus deploy latency multiplied by service size. Monolith 100 MB deploy может занять minutes.

Coupling растёт с ростом кода. Modules переплетаются. Without strict discipline — implicit dependencies через shared internal state. Refactoring becomes harder over years.

Один stack. Весь monolith на Java или .NET — nothing piece written на Go for performance или Python для ML. Technology lock-in.

Долгий CI. Тесты всей системы часами. Unit tests fast но integration tests span entire application. Feedback loop slow.

Ограничение команды. 100 разработчиков в одном repo = merge conflicts, coordination overhead, painful releases. Not scalable organizational structure.

## Микросервисы: плюсы

Независимый scale. Payment под нагрузкой → 10 pods; остальные — 1. Right-sizing per service. Cost efficient — pay for what needed.

Независимый deploy. Команды не мешают друг другу. Team A deploys hourly. Team B deploys weekly. Не coordination needed. Faster iteration cycles.

Fault isolation. Упал notification → user login работает. Failures contained. Better user experience during partial outages.

Разные stacks. Orders на Java, ML на Python, gateway на Go. Right tool для each job. Technology choices per service.

Меньшие команды на сервис. 2-8 человек, ownership. Small enough to move fast, large enough have autonomy. Two-pizza rule (Amazon).

Быстрее CI. Тесты только своего сервиса. Feedback quick. Developer productivity higher.

Легче переписать сервис. Один маленький service replacable в 1-3 months. Rewrite monolith 5-year project. Technology renewal тактическое not strategic.

## Микросервисы: минусы

Distributed system complexity. Сеть ненадёжна. Latency non-zero. Partial failure normal. Programming models должны account для этого.

Distributed transactions. Нет ACID через несколько сервисов. Saga, Outbox, compromise. Business processes require careful design. Eventual consistency становится norm.

Debugging сложнее. traceId по сервисам, множество логов. Distributed tracing required — Zipkin, Jaeger. Without observability infrastructure — debugging nightmares.

Много инфраструктуры. Kubernetes для orchestration. Service mesh для communication. Message broker для async. Monitoring stack. Все обязательны even до launch.

Overhead коммуникации. HTTP round-trip 1-10 ms vs function call 1 ns. Multiple hops между services adds up. Latency budget spent on communication overhead.

Data consistency eventual not strong. Reads могут return stale data briefly. Business logic должно tolerate. Users могут see inconsistent states temporarily.

Сложность deploy. Нужен CI/CD per сервис. Разные версии в production. Compatibility между services managed carefully.

Testing сложнее. Integration много сервисов. E2E tests fragile. Contract testing plus service virtualization required.

Больше стоимость. Больше подов, больше compute. Base overhead per service (JVM start memory, sidecar containers).

Не для маленьких команд. 5 человек в 20 сервисах = боль. Not enough people для operating multiple services well.

## Distributed monolith: worst of both

Distributed monolith = микросервисы но:
- Все жёстко связаны (нельзя поменять один без всех).
- Deploy одновременно.
- Общая БД.
- Sync HTTP chain (A → B → C → D → E, если один упал, все упали).

Худшее из обоих миров — сложность микросервисов плюс coupling монолита.

Симптомы. Все сервисы deployment'ся одновременно. Изменение одного API ломает 5 других. Общая БД, все читают/пишут. Circular dependencies между сервисами. Deploy plans требуют coordination across teams.

Common causes. Sharing common libraries too aggressively. Shared database. Sync HTTP chains without asynchronous alternatives. Not applying bounded contexts. Lack of API versioning discipline.

Fix — правильные boundaries (DDD), event-driven communication, database per service. Break dependencies through async events instead of sync calls. Introduce service versioning. Enforce contract testing.

## Modular monolith: compromise

Modular monolith = монолит с чёткими границами модулей внутри. Один процесс, один deploy. Но modules изолированы (пакетная структура, не общие модели). Общение между modules через явные интерфейсы (не прямые вызовы random классов).

Пример structure. Packages по business domain — com.example.orders, com.example.payments, com.example.notifications. Each package имеет public API (interfaces, DTOs) и private implementation. Cross-module calls только через public API interfaces.

Плюсы. Простота монолита retained. Готов к разделению на микросервисы если понадобится — модуль borders явные, можно extract to separate deploy.

Минусы. Требует дисциплины. Легко нарушить границы без architectural enforcement (например, Java 9 modules или ArchUnit tests catching violations).

Рекомендация большинства (Martin Fowler, Sam Newman) — начинать с модульного монолита. Разделять только когда реально нужно. Start simple, evolve based on actual constraints.

## Когда что выбирать

Стартап / MVP. Монолит. Быстрая разработка critical. Малая команда. Ещё не знаешь domain — микросервисы decompose based on domain understanding.

Маленькая команда (< 10 человек). Монолит. Overhead микросервисов не окупается. Better spent developer time on features than infrastructure.

Средний продукт с большой командой. Modular monolith или несколько крупных сервисов (не микросервисов). 3-5 сервисов по бизнес-доменам. Balance between simplicity и team parallelism.

Большая система, много команд. Микросервисы. Каждая команда — свой сервис. Independent deployment critical для team autonomy.

Разные требования к scale. Микросервисы. Payment под жёсткой нагрузкой, admin — редкий → раздельный scale. Cost efficiency substantial.

Legacy монолит с постоянными incidents. Strangler pattern (см. ниже). Gradual migration.

## Коммуникация между микросервисами

Sync via HTTP/REST — стандарт. JSON payloads. Human-readable. Универсально понятно. Ubiquitous tools.

Плюсы. Простота. Всем понятно. Легко debug.

Минусы. Caller зависит от callee. Upstream down → downstream упал. Cascade failures. Latency plus serialization overhead.

Использование. Query data. Immediate response requests. Между UI-backend и микросервисами.

Sync via gRPC. Google RPC. HTTP/2 based. Protobuf serialization.

Плюсы. Быстрее REST (binary plus multiplexing). Типобезопасно (schema). Streaming support.

Минусы. Сложнее debug (binary). Не работает из browser напрямую (нужен gRPC-Web proxy). Требует shared schemas.

Использование. Internal service-to-service где performance critical. High-throughput internal APIs.

Sync via GraphQL. Client specifies exact fields needed. Reduces over/under fetching.

Плюсы. Клиент управляет data shape. Оптимальные responses per use case.

Минусы. Сложность backend. N+1 проблема без DataLoader. Кэширование сложнее REST.

Использование. BFF (Backend for Frontend) для сложных UI. Rich mobile applications с многими data needs.

Async via Message broker. Rabbit, Kafka, SQS, NATS. Producer publishes event → broker → consumer(s) process.

Плюсы. Decoupling — producer не знает о consumer. Buffering — пик нагрузки поглощается. Multiple consumers — pub-sub. Retry plus durability.

Минусы. Eventual consistency. Сложнее debug (async traces). Требует broker infrastructure.

Использование. Предпочтительно между backend сервисами. Events для state propagation. Fire-and-forget notifications.

Async via Event streaming (Kafka). Event log — можно replay. Multiple consumers each со своим offset.

Использование. Event sourcing. Analytics pipelines. Historical event processing.

Правило (Sam Newman, Chris Richardson). Sync only для outer edge (UI → gateway → services). Async events между backend services. gRPC internal high-throughput. Minimize sync chains (A → B → C → D → E — плохо).

## Strangler pattern для миграции

Как из монолита сделать микросервисы постепенно. Название от strangler fig — растение, обвивающее дерево и постепенно его убивающее.

Идея — не переписывать всё за раз. Наоборот incremental replacement.

Шаг 0 baseline. Legacy monolith serving traffic directly.

Шаг 1. Поставить API Gateway перед монолитом. All traffic routes через gateway.

Шаг 2. Выделить одну функциональность (например user management) в новый microservice. Deploy alongside monolith. Gateway routes user requests к new service. Everything else продолжает go to monolith.

Шаг 3-N. Постепенно выделять другие modules. Каждый migration независимый — gateway routing decisions.

Финальный шаг. Monolith пустой — все functions в микросервисах. Retire monolith.

Плюсы. Работает всё время (нет big bang). Можно откатить любой шаг. Обучение команды параллельно с migration.

Минусы. Долго — 2-5 years для significant migrations. Двойная поддержка (монолит plus микросервисы). Требует четкого API Gateway.

Практические советы. Начинай с stable features (не под активной разработкой). Или наоборот — самые проблемные (получаешь value быстро). Не переписывай 1-в-1 — используй возможность улучшить. Data migration — сложный шаг, может занять больше кода чем service logic. Rollback должен работать в каждый момент.

## Data management

Database per service — стандартное правило microservices. Каждый микросервис — своя БД.

Плюсы. Изоляция schema — каждая service evolves schema independently. Свой tuning — different services могут have very different DB needs. Свободный refactoring — schema changes не affect другие services.

Минусы. Дубли данных — customer info в orders plus payments plus shipping. Сложность консистентности между services — Saga plus Outbox required.

Shared database — antipattern. Все сервисы читают/пишут общую БД → distributed monolith.

Проблемы. Изменение schema ломает все сервисы. Кто-то держит lock — все ждут. Coupling через данные. Cannot evolve schemas independently.

Иногда допустимо для legacy migration с постепенным разделением. Not intentional долгосрочный design.

CQRS (Command Query Responsibility Segregation) — разделение write и read моделей. Command side — write model (normalized). Query side — read model (denormalized, optimized). Sync через events или CDC.

Полезно когда read/write очень разные требования — много reads, complex analytics. Complexity added — worthwhile только для specific scenarios.

Event Sourcing — хранить события (что произошло), а не текущее состояние. OrderCreated, OrderPaid, OrderShipped — записи. Current state = replay всех events.

Плюсы. Audit trail из коробки. Time-travel debug. Естественно для event-driven архитектур.

Минусы. Сложно — не для CRUD-приложений. Snapshots для performance. Event schema migration challenging.

## Deployment considerations

Микросервисы предполагают. Docker containerization. Kubernetes orchestration. CI/CD per service. Blue-green или Canary deployment strategies. Service mesh опционально (Istio, Linkerd). Observability stack (Prometheus, ELK, Jaeger).

Все это = infrastructure investment. Не начинай с микросервисов если нет ресурсов на platform team supporting всю эту infrastructure.

Deployment automation critical. Manual deployments 20 services не scale. Full pipeline от git commit к production must be automated.

Rollback strategies. Blue-green — parallel deployment new version, switch when ready. Canary — gradual rollout, watch metrics, expand если ok. Feature flags — deploy code hidden behind flags, enable independently от deployment.

## Common mistakes

«Микросервисы = современно». Нет, микросервисы = инструмент. Стартап с 5 человек → пиши монолит. Cargo culting microservices without understanding trade-offs recipe для disaster.

«Разделим на 50 сервисов сразу». Начни с 3-5. Разделяй только когда чувствуешь боль монолита. Big-bang micro-decomposition rarely works.

«Каждая сущность — свой сервис». Нет, по бизнес-домену / bounded context (см. файл 49). Entity per service — wrong granularity. Cross-entity operations force sync HTTP chains.

«Common library для всех». Опасно. Легко превратить микросервисы в distributed monolith. Общий код — только stable утилиты (в ИСНА — isna-commons, isna-global). Business logic sharing = tight coupling.

«Каждая команда — свой стек». Плюс — свобода. Минус — нет переиспользования, migration новых людей сложна. Обычно — договориться на 1-2 stacks (Java plus Node, или Java only).

## Real-world КНП опыт

Микросервисы в КНП практика. Много сервисов (~30-50). Java stack. Общие библиотеки (isna-commons, isna-global) держат единый стиль. API Gateway — Zuul (legacy, Java 11 only). Consul для discovery. RabbitMQ для async. PostgreSQL — своя БД у большинства сервисов. Keycloak для auth. K8s для orchestration. ELK для logs.

Caveats из memory. Гейт остался на Java 11 (knp-gateway-no-java21) — миграция трудна из-за Zuul 1 incompatibility. Общие библиотеки создают coupling (isna-commons-dual-keep) — держим дважды для Java 11 и 21. Много sync HTTP → risk каскадных отказов → нужны Circuit Breaker plus timeouts.

Lessons learned. Common libraries carefully — think about coupling implications. Legacy technology (Zuul 1) creates long-term migration debts. Sync communication defaults problematic — invest в async infrastructure. Even mature microservices architectures continue evolving.

## Итоги

Монолит — один процесс, один deploy, одна БД. Простота, ACID transactions, быстрая коммуникация, легче debug. Но scale только целиком, deploy longer, coupling grows.

Микросервисы — независимый deploy, fault isolation, разные stacks, independent scaling. Но distributed complexity, eventual consistency, много infrastructure, сложный debug.

Distributed monolith — antipattern combining worst обоих worlds. Tight coupling через API dependencies plus shared databases plus sync HTTP chains. Fix через bounded contexts, async events, независимый deploy.

Modular monolith — compromise. Простота монолита plus готовность к microservices decomposition. Optimal starting point.

Choice based на context. Startups и small teams — monolith. Large systems, many teams — microservices. Middle ground varies.

Communication patterns. Sync для immediate response. Async предпочтительно между backend services. Sync только через API Gateway для outer edge.

Strangler pattern для incremental migration legacy к microservices. Не big bang.

Database per service — правило microservices. Shared database antipattern. CQRS plus Event Sourcing для complex data patterns.

Common mistakes — cargo culting microservices, wrong granularity (entity per service), overshared common libraries, mismatched team stacks.

Дальше — глубже в микросервисы decomposition через DDD, communication patterns, service discovery, contract testing.
