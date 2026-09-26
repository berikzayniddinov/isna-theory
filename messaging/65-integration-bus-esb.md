# 65. Интеграционная шина (ЕСБ / ESB): enterprise integration middleware

## Зачем понимать ESB в эпоху микросервисов

Разработчик работающий преимущественно с микросервисами обычно думает — ESB это legacy что не касается его. Reality приносит ESB в жизнь через enterprise integration scenarios. Legacy monoliths interconnected через central bus. Government systems integrated via SOAP-based ESB. Financial institutions using enterprise integration platforms. Understanding ESB важно для effectively working в enterprise contexts даже когда сам ESB не любимая архитектура.

Разница между разработчиком «не знающим ESB» и «понимающим ESB» проявляется в enterprise integration projects. Первый видит ESB tickets и confused. Второй знает fundamental ESB concept — hub-and-spoke integration reducing N×N point-to-point complexity. Знает classical ESB functions — routing, transformation, mediation, enrichment. Знает Apache Camel patterns для route-based integration. Знает EAI patterns book Hohpe/Woolf как classical reference. Знает why ESB deprecated в микросервисах — «smart endpoints, dumb pipes» philosophy shift. Знает специфический ИСНА context — ЕСБ для government integration, каналы ЛС-ИШ-ОС как критические business paths.

В этом файле разберём ESB глубоко. Что такое интеграционная шина. Классические функции. ESB продукты. Apache Camel workflow. EAI patterns. Почему deprecated в микросервисах. Где ещё живёт. iPaaS modern reincarnation. ИШ в контексте ИСНА (ЕСБ, каналы ЛС-ИШ-ОС). ESB vs API Gateway. ESB vs Message Broker. ESB vs Service Mesh. Когда ещё нужен ESB. Anti-patterns. Modern integration подход.

## Что такое интеграционная шина

Интеграционная шина (ИШ / ESB — Enterprise Service Bus) — централизованное middleware для интеграции разных систем.

Идея. Много приложений разной природы (Java monoliths, SAP, mainframe, БД, Web-сервисы). Все нужно связывать (обмен данными, вызовы). Вместо N × N point-to-point интеграций — все через центральную шину.

Без шины:
```
Без шины (point-to-point):

  App A ─── App B
    │  ╲    ╱  │
    │   ╲  ╱   │
    │    ╳     │
    │   ╱  ╲   │
    │  ╱    ╲  │
  App C ─── App D

  6 связей на 4 приложения.
  N × (N-1) / 2 связей.
```

С шиной:
```
С шиной (hub-and-spoke):

    App A       App B
       ╲       ╱
        ╲     ╱
       [ESB]        ← центральная шина
        ╱     ╲
       ╱       ╲
    App C       App D

  4 связи, каждое приложение только с шиной.
```

Mathematical benefit. Point-to-point requires N(N-1)/2 connections. Hub-and-spoke requires just N. Massive reduction connection count с growth systems.

Плюсы. Loose coupling — приложения не знают друг о друге. Централизованная логика (routing, security, monitoring). Легко добавить новое приложение — connect to bus not to every other system.

Минусы. ESB — SPOF (single point of failure). Централизованная бизнес-логика («smart pipe, dumb endpoints») — трудно поддерживать. Vendor lock-in.

## Классические функции ESB

Routing. Определить куда доставить сообщение. Content-based routing:
```
Если Order.type = "EXPRESS" → shipping-service
Если Order.type = "REGULAR" → warehouse-service
```

Decision made в bus based on message content.

Transformation. Одна система шлёт XML, другая ожидает JSON. Одна — camelCase, другая — snake_case. ESB преобразует на лету. Инструменты — XSLT (XML), XPath, JOLT (JSON), custom mapping.

Removes coupling через format differences. Systems don't need to agree on formats.

Mediation. Между несовместимыми протоколами. SOAP → REST. HTTP → JMS. File → Kafka. Protocol translation в bus.

Protocol bridging. Клиент SOAP, backend REST. ESB — переводчик. Enables mixing technologies.

Message enrichment. Добавить данные из других систем в сообщение по пути:
```
Order → [ESB: спросить у customer-service имя клиента] → order+customerName → dest
```

Middleware layer enriches messages с additional context. Reduces individual system complexity.

Security. Централизованная аутентификация / шифрование. Consistent security policy applied в bus.

Auditing / Monitoring. Все сообщения через шину — легко логировать / метрики. Centralized observability.

Guaranteed delivery. С persistent queues — сообщение не потеряется. Enterprise messaging reliability.

Comprehensive set функций. ESB does many things — often too many в one place.

## Классические ESB продукты

Enterprise Java экосистема (1990-2010е).

Apache ServiceMix (open source). Community-driven ESB. Java-based. Used в some open source integrations.

MuleSoft (сейчас Salesforce). Commercial popular platform. Now iPaaS focused.

IBM Integration Bus (IIB / DataPower). Enterprise IBM product. Comprehensive but expensive.

Oracle Service Bus (OSB). Oracle enterprise product. Common в Oracle-heavy shops.

TIBCO BusinessWorks. Legacy enterprise integration platform.

Microsoft BizTalk. Microsoft enterprise integration. Windows/.NET-centric.

JBoss ESB (deprecated). Red Hat's ESB product. Discontinued.

Apache Camel — не ESB but integration framework часто в связке. Route-based DSL для integration patterns.

Все — тяжелые, дорогие enterprise systems. License costs plus operational complexity substantial.

## Apache Camel пример

Apache Camel популярный integration framework (может работать как lightweight ESB):
```java
@Component
class OrderRoute extends RouteBuilder {
    @Override
    public void configure() {
        from("jms:queue:orders.incoming")
            .marshal().json(JsonLibrary.Jackson)
            .enrich("http://customer-service/api/customers/{customerId}",
                    new CustomerEnricher())
            .choice()
                .when(header("type").isEqualTo("EXPRESS"))
                    .to("http://shipping-service/api/express")
                .when(header("type").isEqualTo("REGULAR"))
                    .to("http://warehouse-service/api/order")
                .otherwise()
                    .to("jms:queue:orders.error")
            .end();
    }
}
```

Читабельно. Входящий JMS → transform → enrich → routing → outgoing HTTP/JMS.

Camel — 300+ компонентов (JMS, HTTP, File, Kafka, Salesforce, S3, ...). Comprehensive integration ecosystem.

Lightweight alternative к heavy ESB. Programming model более developer-friendly. Route definitions в code not GUI.

## EAI patterns

Enterprise Integration Patterns (Hohpe и Woolf, 2003) — классическая книга. Foundational reference для integration architecture.

Ключевые паттерны.

Message Channel — очередь между sender и receiver. Basic communication primitive.

Message Endpoint — как приложение подключается к каналу. Connection point.

Message Router — выбирает канал по content. Routing logic pattern.

Message Translator — трансформирует формат. Format bridging.

Content Enricher — добавляет данные. Message augmentation.

Content Filter — убирает лишнее. Data reduction.

Aggregator — собирает связанные сообщения в одно. Correlation and combination.

Splitter — разбивает одно на много. Fan-out processing.

Wire Tap — копия для audit. Non-intrusive monitoring.

Dead Letter Channel — куда деть проблемные сообщения. Error handling.

Idempotent Receiver — избежать дублей. Reliability pattern.

Все реализуются в ESB / Camel / любом integration tool. Universal patterns transcending specific technologies.

Book still relevant даже в микросервисах. Patterns apply к any integration scenario.

## Почему ESB deprecated в новых системах

С развитием микросервисов и cloud-native подход изменился.

«Smart endpoints, dumb pipes». Мартин Фаулер. Логика — в приложениях, не в шине. Шина — просто транспорт.

ESB накапливает бизнес-логику — превращается в integration monolith. Одну строчку изменить — тестирование всей шины. Central bottleneck.

Микросервисы вместо интеграции. Раньше приложения были монолитами (SAP + Oracle + custom Java). ESB интегрировала. Теперь приложения сами — набор микросервисов, общаются через HTTP/events. ESB не нужен.

Cloud-native альтернативы. API Gateway — routing plus auth (обычно). Message broker (Kafka/Rabbit/SQS) — async messaging. Service Mesh (Istio/Linkerd) — mTLS, retry, observability. Serverless / Cloud Functions — трансформации.

Каждый инструмент делает одну вещь хорошо. Композиция > монолитный ESB. Best-of-breed vs monolithic platform.

DevOps / Continuous Deployment. ESB deploy — недели (изменения в central hub). Микросервисы — минуты. Iteration speed matters.

Vendor lock-in. Проприетарные ESB — тяжело мигрировать. Enterprise dependencies persist для years.

## Где ESB ещё живёт

Не всё умерло. ESB жив в specific contexts.

Enterprise Java (banking, insurance, gov). Legacy monoliths plus SAP plus mainframe требуют интеграции. Нельзя переписать всё. ESB как integration layer legacy investments.

Регулируемые отрасли. Строгий audit, security, contract compliance. ESB даёт централизованный контроль. Regulator wants «one point» integration.

Государственные системы. Как ИСНА в Казахстане. Historical foundation SOAP-based ESB integration.

Modern ESB — iPaaS. Integration Platform as a Service — cloud-based ESB. MuleSoft Anypoint Platform. Boomi. Workato. AWS Step Functions + EventBridge. Azure Logic Apps.

Легче чем legacy ESB, но идея та же. Central integration platform с cloud economics. Managed service model.

## ИШ в контексте ИСНА

ЕСБ — Единая Сервисная Шина. В ИСНА есть ЕСБ — центральная integration bus для интеграции с внешними государственными системами.

Через ЕСБ идут. Взаимодействие с системами других ведомств. Приёмка ФНО / ФО (save-fno-<code> SOAP endpoints). Синхронизация данных между налоговой и другими системами. Проверки ИИН/БИН, регистрационных данных.

Из memory. knp-fno-outer-sync-esb-dead-route — SOAP «Requested service is not found» когда маршрут BT_OUTER_FNOSYNC_OUTER_TAXREP_SYNC не поднят на ЕСБ. Логика — KnpOuterSystemFnoSyncService вызывает ЕСБ — падает если маршрут не настроен — 1586 ошибок за 13 часов.

Урок. Зависимость от ЕСБ равно зависимость от инфры. Circuit breaker plus флаг отключения важны. Central bus becomes SPOF.

Каналы ЛС-ИШ-ОС. Из memory knp-e2e-prod-smoke-scope-rules. «Не смокать мутирующие/опасные API (send/запись/запрос в ЛС-ИШ-ОС под ЭЦП владельца)».

Что означает. ЛС — Личный Счёт налогоплательщика (транзакции, разноска платежей). ИШ — Интеграционная Шина (обмен с внешними системами). ОС — Отправка/Обмен Сообщениями (или Openservice, «Отчётная Система»).

Три канала критичных операций требующих ЭЦП владельца.

Правило смока. Не мутировать ничего в эти каналы (не создавать нагрузки на прод, не влиять на состояние ЛС, не слать в ИШ, не отправлять в ОС). Test scope restricted для business safety.

SOAP vs REST в ИСНА. Внешний слой (интеграция с гос. системами) — SOAP через ЕСБ/ИШ. Внутренний слой (микросервисы КНП, АРМ) — REST через Feign + Consul. Gateway — isna-knp-gateway (Zuul) на входе, роутит.

Так исторически. SOAP был стандартом gov, REST пришёл позже для user-facing. Hybrid architecture reflects evolution.

## ESB vs API Gateway

Часто путают. Разница важна.

| | ESB | API Gateway |
|---|---|---|
| Роль | Интеграция систем | Точка входа для клиентов |
| Направление | Internal ↔ Internal | External → Internal |
| Функции | Transformation, routing, mediation, enrichment | Auth, rate limit, routing, aggregation |
| Message-first | Да (JMS, queues) | Нет (HTTP-first) |
| Модель | Push (event-driven) | Pull (request-response) |
| Сложность | Высокая | Средняя |
| Async | Естественная | Обычно sync (или facade over async) |

API Gateway — edge для internet клиентов. ESB — hub для internal системной интеграции.

Иногда сливаются в одном продукте (MuleSoft умеет оба).

В ИСНА. isna-knp-gateway — API Gateway (Zuul) для внешних клиентов. ЕСБ/ИШ — ESB для интеграции с гос-системами. Different roles complement.

## ESB vs Message Broker

| | ESB | Broker (Rabbit/Kafka) |
|---|---|---|
| Логика | В шине | В consumer'ах |
| Transformation | Да | Нет (в приложении) |
| Routing | Content-based | Простой (queue/topic name) |
| Enrichment | Да | Нет |
| «Smartness» | Smart pipe | Dumb pipe |

Broker — простой транспорт. ESB — умный посредник.

Современный подход. Broker plus smart consumers. Logic distributed to endpoints, transport dumb.

Fundamental design philosophy shift. Microservices avoid centralized logic.

## ESB vs Service Mesh

Service Mesh (Istio, Linkerd) — infrastructure layer для sidecar proxy.

| | ESB | Service Mesh |
|---|---|---|
| Уровень | Приложение | Инфраструктура |
| Реализация | Центральный broker | Sidecar на каждом service |
| Функции | Transformation, routing, business logic | Retry, mTLS, tracing, LB |
| Coupling | High (все через ESB) | Low (transparent) |

Service Mesh — cross-cutting infrastructure без бизнес-логики. Sidecars intercept traffic. Application unaware.

ESB — business-level integration. Contains business logic within bus.

Разные слои. Different concerns. Complementary rather than alternative usually.

## Когда ещё нужен ESB

Не для новых микросервис-проектов. Но specific scenarios.

Legacy integration. 10 monolithic apps, каждый со своим API — ESB решает. Bridge existing systems.

B2B integration. Партнёры со своими протоколами (SOAP, EDI, файлы). ESB — точка перевода. Multi-protocol translation.

Enterprise workflows. Сложные orchestration (BPEL) — ESB часто включает workflow engine.

Централизованный audit / compliance. Регулятор требует «все сообщения через одну точку». Compliance driven.

Gov / regulated. Как ИСНА — исторически SOAP plus ESB для gov integration.

## Anti-patterns

Business logic в ESB. Расчёты, валидации, decisions — в приложениях, не в шине. Rule ownership belongs в services owning business capability.

Огромный ESB monolith. Один ESB для всей компании — все зависят от одной команды. Правильно — несколько специализированных (по domain). Bounded contexts apply к ESBs too.

God's message. Одно сообщение несёт всё для всех. Sender не знает что receiver'ы возьмут — coupling скрыт. Explicit message contracts better.

Sync через async broker. RPC поверх Rabbit/Kafka — ждёшь ответ через тот же broker. Медленно, сложно debug. Async should be async fully.

## Modern подход к integration

Choreography (см. файл 50 saga-pattern). Каждый сервис публикует events. Другие подписываются. Никого центрального посредника.

Инструменты. Kafka plus Schema Registry — event streaming backbone. Rabbit — task queues. API Gateway — HTTP edge. Service Mesh — infrastructure. Event Sourcing / CQRS — data.

Distributed, cloud-native, микросервисы. ESB не нужен. Modern architecture composes лучших-of-breed tools instead of monolithic platform.

## Итоги

ESB (Enterprise Service Bus) централизованное middleware для интеграции разных систем. Hub-and-spoke pattern reducing N×N connections.

Классические функции. Routing (content-based). Transformation (formats). Mediation (protocols). Enrichment (adding data). Security (centralized). Auditing. Guaranteed delivery.

ESB продукты. Apache ServiceMix, MuleSoft, IBM IIB, Oracle OSB, TIBCO, BizTalk. Все heavy enterprise systems.

Apache Camel как lightweight integration framework. Route DSL. Comprehensive component library.

EAI patterns (Hohpe/Woolf) — foundational integration architecture patterns. Message Channel, Router, Translator, Enricher, Aggregator, Splitter, Dead Letter, Idempotent Receiver.

ESB deprecated в микросервисах. «Smart endpoints, dumb pipes» philosophy. Микросервисы plus broker plus gateway plus mesh replace monolithic ESB.

Ещё живёт в. Banking, insurance, government, regulated industries, legacy integration. Compliance plus existing investments sustain.

iPaaS modern reincarnation. MuleSoft Anypoint, Boomi, Workato. Cloud-native ESB. Managed service.

В ИСНА. ЕСБ для integration с external government systems. Каналы ЛС-ИШ-ОС критические business paths. SOAP-based enterprise integration.

ESB vs API Gateway. ESB internal integration hub. Gateway external entry point. Different roles.

ESB vs Message Broker. ESB smart pipe. Broker dumb pipe. Modern preference broker plus smart consumers.

ESB vs Service Mesh. ESB business-level. Service Mesh infrastructure-level. Different concerns.

Anti-patterns. Business logic в ESB. Огромный monolithic ESB. God messages. Sync через async broker.

Modern integration через choreography. Kafka events. API Gateway edge. Service Mesh infrastructure. Composed tools not monolithic ESB.

Дальше — sync vs async communication patterns. When to use which. Trade-offs.
