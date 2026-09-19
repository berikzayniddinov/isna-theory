# 64. REST vs SOAP: развёрнутое сравнение, когда что выбирать

## Зачем детально сравнивать

Разработчик который знает и REST и SOAP обычно застревает при architectural decisions. Нужен API для нового сервиса — REST по умолчанию. Но что делать когда партнёр требует SOAP? Legacy system использует SOAP — как переезжать на REST? Enterprise integration — какие критерии выбора? Оба технологии имеют своё место но правильное selection depends на context.

Разница между разработчиком «знающим оба» и «понимающим trade-offs» проявляется в architectural leadership. Первый выбирает REST потому что «modern» или SOAP потому что «enterprise». Второй знает конкретные dimensions. Format (JSON vs XML). Contract (optional OpenAPI vs mandatory WSDL). HTTP usage (semantic vs transport). Security model (transport-level vs message-level). Transaction support (Saga/Outbox vs WS-Transaction). Performance (compact vs verbose). Tooling ecosystem. Знает hybrid approaches — SOAP для external partners, REST для internal microservices, adapter layer between.

В этом файле сравним REST и SOAP comprehensively. Природа fundamental differences. Full comparison table. Сравнение сообщений на конкретных examples. Как каждый использует HTTP. Contract approaches. Security models. Transactions. Performance. Developer experience. Где что используется в реальности. Когда что выбрать. Гибридные подходы. Modern alternatives (gRPC, GraphQL). Migration paths. Реальные ИСНА scenarios.

## Природа fundamental differences

REST — архитектурный стиль. Набор принципов, не протокол. Fielding's constraints. Как строить APIs.

SOAP — протокол. Строгие правила формата и обмена. XML envelope. Как форматировать сообщения.

Разница фундаментальная. REST answers «how to structure APIs». SOAP answers «how to format messages». Different levels of abstraction.

REST leverages HTTP as application protocol. Uses HTTP semantics (methods, status codes, caching). SOAP uses HTTP just as transport. Everything в SOAP layer.

## Основные различия — таблица

| | REST | SOAP |
|---|---|---|
| Что это | Архитектурный стиль | Протокол |
| Формат | Обычно JSON (XML/YAML тоже) | Только XML |
| Транспорт | HTTP (обычно только) | HTTP, SMTP, JMS, TCP |
| Contract | Опционально (OpenAPI) | Обязательно (WSDL) |
| Runtime schema validation | Нет из коробки | Да (XSD) |
| HTTP semantics | Использует (GET/POST/PUT/DELETE, статус-коды) | Всё через POST, статус 200 |
| State | Stateless | Stateful возможен |
| Caching | Отлично (HTTP-кэш) | Плохо (POST только) |
| Standards | Loose | Много (WS-*) |
| Security | HTTPS + JWT/OAuth | WS-Security (message-level) + HTTPS |
| Transactions | Через Saga/Outbox | WS-Transaction (2PC) |
| Payload size | Компактно (JSON) | Verbose (XML boilerplate) |
| Performance | Быстрее | Медленнее |
| Bandwidth | Меньше | Больше |
| Developer experience | Простой | Сложный |
| Tooling | Универсальный (браузер, curl, Postman) | Специальный (SoapUI, wsimport) |
| Browser support | Отлично (JS fetch) | Плохо |
| Разработка | Code-first обычно | Contract-first обычно |
| Async / one-way | Нет из коробки | Да (WS-Addressing) |
| Federation / SSO | OAuth/OIDC | WS-Federation, WS-Trust |
| Learning curve | Пологий | Крутой |
| Год появления | 2000 (Fielding) | 1998 (Microsoft) |

Comprehensive picture. Trade-offs across multiple dimensions.

## Сравнение сообщений

Одна и та же операция — создать order.

REST plus JSON.

Request:
```
POST /orders HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJ...

{
  "customerId": "c1",
  "amount": 100
}
```

Response:
```
HTTP/1.1 201 Created
Location: /orders/42
Content-Type: application/json

{
  "id": 42,
  "status": "NEW"
}
```

Размер приблизительно 200 bytes. Compact. Human-readable. Standard HTTP semantics.

SOAP plus XML.

Request:
```
POST /orders HTTP/1.1
Host: api.example.com
Content-Type: text/xml; charset=utf-8
SOAPAction: "http://example.com/orders/create"
Content-Length: 500

<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Header>
    <wsse:Security xmlns:wsse="...">
      <wsse:UsernameToken>
        <wsse:Username>alice</wsse:Username>
        <wsse:Password>...</wsse:Password>
      </wsse:UsernameToken>
    </wsse:Security>
  </soap:Header>
  <soap:Body>
    <ord:CreateOrder xmlns:ord="http://example.com/orders">
      <ord:CustomerId>c1</ord:CustomerId>
      <ord:Amount>100</ord:Amount>
    </ord:CreateOrder>
  </soap:Body>
</soap:Envelope>
```

Response:
```
HTTP/1.1 200 OK
Content-Type: text/xml

<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <ord:CreateOrderResponse xmlns:ord="http://example.com/orders">
      <ord:OrderId>42</ord:OrderId>
      <ord:Status>NEW</ord:Status>
    </ord:CreateOrderResponse>
  </soap:Body>
</soap:Envelope>
```

Размер приблизительно 800-1500 bytes (в 5-10 раз больше). Verbose. Namespace-heavy. HTTP status always 200.

Difference profound. Same operation, dramatically different message sizes. Same HTTP status semantics wildly different (201 vs 200).

## Использование HTTP

REST использует полностью. Methods — GET/POST/PUT/PATCH/DELETE — семантика. Status codes — 200/201/404/409/500 — разные ситуации. Headers — Cache-Control, ETag, Location. Content Negotiation через Accept.

REST leverages HTTP full capabilities. HTTP not just transport — application protocol.

SOAP только транспорт. Всё через POST. Всегда status 200 (или 500 for fault). Информация о fault — в body (не в HTTP). SOAPAction header уточняет операцию.

По сути SOAP игнорирует HTTP-возможности. Мог бы быть на любом транспорте. Transport-agnostic design abstracts HTTP away.

Fundamental philosophical difference. REST embraces HTTP. SOAP treats HTTP как transport pipe.

## Contract

REST обычно code-first. Пишешь Spring controller — генерируется OpenAPI (через springdoc). Опционально. Многие REST APIs без формального contract:
```java
@RestController
class OrderController {
    @PostMapping("/orders")
    public Order create(@RequestBody OrderDto dto) { ... }
}
```

Springdoc авто-сгенерирует:
```yaml
paths:
  /orders:
    post:
      requestBody: ...
      responses:
        '200': ...
```

Flexible. Contract может быть documented if needed. Not mandatory.

SOAP обязательно contract-first. Traditional flow. Пишешь WSDL. Генерируешь код (wsimport → Java classes). Реализуешь методы.

Или code-first:
```java
@WebService
public class OrderService {
    public Long createOrder(...) { }
}
```

Spring генерирует WSDL из аннотаций. Modern option — code-first with generated contract.

Runtime validation. SOAP — сообщения валидируются против XSD автоматически. Invalid = SOAP fault. Runtime type safety enforced. REST — валидация ручная (Bean Validation через @Valid). Или через OpenAPI validator в middleware. Optional validation.

Строгость SOAP — плюс для enterprise integration (партнёры хотят «гарантию»). Strict contracts reduce integration bugs.

## Security

REST Transport-level. Обычно. HTTPS (TLS) — encryption in-transit. JWT / OAuth 2.0 для authentication. API keys для service-to-service.

Security между клиентом и сервером. Через прокси / gateway — прерывается TLS. Encryption ends at endpoints.

Простой, стандартный. Standard web security practices.

SOAP Message-level (WS-Security). WS-Security позволяет. Encrypt конкретные части XML. Sign отдельные элементы. Username tokens в SOAP Header. SAML tokens.

Плюс. End-to-end через intermediary'ов. Клиент — Gateway — Server — сообщение остаётся signed/encrypted всю дорогу. TLS terminates on gateway но message security continues.

Минус. Сложно, много boilerplate.

Different security paradigm. REST relies на transport encryption. SOAP embeds security в message.

Enterprise scenarios с multiple intermediary systems favor SOAP message-level security. Simpler scenarios REST transport security достаточен.

## Transactions

REST. Локальная tx — обычная @Transactional. Distributed — Saga, Outbox, Idempotency (см. файлы 50, 51). Нет стандарта для distributed tx через REST.

Modern approach — eventual consistency через compensating actions. Не 2PC.

SOAP. WS-AtomicTransaction (WS-AT) — 2PC поверх SOAP. WS-BusinessActivity (WS-BA) — long-running compensations (аналог Saga).

Enterprise-ready, но сложно plus требует cross-vendor совместимости.

На практике даже в SOAP-мире 2PC редко. Complexity plus availability trade-offs не warrant. Even enterprise moves к eventual consistency patterns.

## Performance

Bandwidth. REST plus JSON — компактно. SOAP plus XML — verbose (namespaces, envelope, boilerplate). В 5-10 раз больше.

CPU (parsing). JSON — быстро парсится (Jackson). XML — медленнее (SAX/DOM/StAX). Text parsing overhead XML higher.

Latency. REST — быстрее (компактнее, меньше overhead). SOAP — медленнее.

Для high-throughput — REST. Predictable performance advantage.

Caching. REST — GET запросы кэшируются HTTP-кэшом, CDN. SOAP — всё POST, no caching.

Big deal для read-heavy APIs. HTTP caching mechanisms sophisticated.

## Developer experience

REST. Простой — браузер, curl, Postman. JSON — читаемо. Легко debug. Быстрый feedback. Онбординг за часы.

SOAP. Нужен WSDL, XSD tooling. XML сложно писать/читать. SoapUI для тестирования. Дни на первый вызов. Онбординг за недели.

Отсюда — популярность REST для web / mobile. Barrier to entry ниже.

Debugging comparison. REST error — read HTTP status, look at JSON message. SOAP error — parse SOAP Fault XML, understand fault codes, decode Detail element.

Order of magnitude difference в developer productivity для similar tasks.

## Что где используется

REST examples. Публичные web APIs — GitHub, Twitter, Stripe, Slack. Mobile apps. SPA / React frontends. Микросервисы (внутренняя коммуникация).

SOAP examples. Banking (SWIFT, ISO 20022 XML-based). Government (Казахстан ИСНА, ЕС ecosystems). Telecom (OSS/BSS). Enterprise B2B (EDI, SAP). Legacy integrations.

Pattern clear. Consumer-facing plus modern services — REST. Legacy enterprise plus regulated industries — SOAP.

## Когда что выбирать

REST большинство случаев. Микросервисы. Публичные APIs. Mobile / SPA backends. Simple CRUD. High-throughput read-heavy.

По default REST. Modern development standard.

SOAP специфичные. Требование партнёра (банк / gov). Legacy integration (существующий SOAP). Строгий contract обязателен. WS-Security end-to-end требуется. Distributed transactions (2PC — редко). Complex enterprise workflows (BPEL).

SOAP не выбирают greenfield. Только когда constraints требуют.

Selection criteria checklist. External requirements (partner mandates)? Existing systems (legacy integration)? Compliance requirements (audit, strong contracts)? End-to-end security (message-level)? Complex workflows (BPEL)? If yes to any — consider SOAP. If no — REST.

## Гибридные подходы

Часто оба в одной системе. Внешние партнёры — SOAP (по их требованию). Внутренние сервисы — REST. Adapter переводит SOAP → REST.

Так в ИСНА. ЕСБ / SOAP-шина для интеграции с внешними гос-системами. Внутренние микросервисы — REST plus Feign. Gateway (Zuul) — L7 маршрутизация.

Adapter pattern common. Facade REST в front of legacy SOAP. Gradual migration path enabled.

## Modern alternatives

Не только REST vs SOAP. Landscape richer.

gRPC. Protobuf binary. HTTP/2. Contract-first (.proto). Быстрее REST. Streaming.

Плюсы SOAP (strong contract, performance) без XML overhead. Modern binary alternative.

Использование — internal microservices, high-throughput. Better для service-to-service чем client-facing.

GraphQL. Query language. Client запрашивает какие поля нужны. Один endpoint.

Плюсы. Гибкие read (no over/under fetching). Efficient для complex UI queries.

Использование. BFF для сложных UIs. Rich mobile apps с varied data needs.

JSON-RPC / XML-RPC. Legacy RPC. Redko used в new projects.

WebSocket / SSE. Для real-time. Bidirectional (WebSocket) или server-push (SSE).

Правило современности. REST — стандарт. gRPC — internal high-performance. GraphQL — flexible reads для UI. SOAP — только когда обязательно. WebSocket — real-time push.

## Миграция SOAP → REST

Если legacy SOAP и хочешь на REST. Не переписывай сразу — Strangler pattern. Facade — REST-фасад перед SOAP-backend. Постепенно — новые features REST, старые остаются SOAP. Deprecate SOAP — 6-12 месяцев параллельно. Delete SOAP.

Incremental migration. Big-bang rewrites risky и often fail. Facade approach enables gradual transition.

## Реальный кейс ИСНА

Из memory. knp-fno-outer-sync-esb-dead-route — SOAP интеграция с ЕСБ через BT_OUTER_FNOSYNC_OUTER_TAXREP_SYNC маршрут. knp-fno-reception-api-mgu-vs-knp — приёмка ФНО через MGU (REST) или КНП (SOAP).

Гибрид. Внешние гос-системы — SOAP-шина. Внутренние микросервисы КНП — REST plus Feign.

Historical accretion. New systems REST. Legacy government integrations SOAP. Bridge layer connects.

## Итоги

REST vs SOAP разные things. REST архитектурный стиль на HTTP. SOAP protocol на XML.

Format. REST JSON (обычно). SOAP XML always.

HTTP usage. REST leverages methods, status codes, caching. SOAP transport only, always POST, 200 status.

Compactness. REST 5-10 раз меньше SOAP.

Contract. REST optional (OpenAPI). SOAP mandatory (WSDL/XSD).

Runtime validation. REST manual. SOAP automatic via XSD.

Security. REST transport-level (TLS + JWT). SOAP message-level (WS-Security) для end-to-end.

Transactions. REST через Saga/Outbox eventual consistency. SOAP WS-Transaction 2PC (редко used).

Performance. REST faster. SOAP more overhead.

Caching. REST HTTP-cache works. SOAP no caching.

Developer experience. REST simple. SOAP complex, weeks onboarding.

Where used. REST public APIs, mobile, SPA, microservices. SOAP banking, government, enterprise legacy.

When SOAP. Requirement, legacy, strict contract, end-to-end security, complex workflows.

When REST. Everything else. Default choice.

Modern alternatives. gRPC для internal high-throughput. GraphQL для flexible UI queries. WebSocket для real-time.

Migration path. Strangler pattern. Facade approach. Incremental replacement.

В ИСНА hybrid. SOAP для ЕСБ external integration. REST для internal microservices.

Дальше — Integration bus (ЕСБ) и ESB patterns. Enterprise integration middleware, when it makes sense, when modern alternatives better.
