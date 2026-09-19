# 62. REST API теория: constraints, HTTP semantics, URI design, best practices

## Зачем понимать REST глубже базового CRUD

Разработчик который пишет @GetMapping и @PostMapping считает — REST понятен. Возвращает JSON, использует HTTP methods, всё работает. Реальность приносит nuances. Почему возвращать 400 или 422 для validation error? Как правильно pagination — offset или cursor? Как versioning API безопасно? Что реально означает stateless? Почему один эндпоинт возвращает Location header но другой нет?

Разница между разработчиком «использующим REST» и «понимающим REST» проявляется в API design decisions. Первый следует intuition — GET для чтения, POST для write, statuses 200 или 500. Второй знает Fielding's six constraints (client-server, stateless, cacheable, uniform interface, layered, code-on-demand). Знает Richardson Maturity Model — most APIs Level 2 (правильные methods plus statuses), HATEOAS Level 3 rarely practical. Знает разницу PUT (idempotent full replace) и PATCH (partial). Знает precise status code semantics — 401 unauthenticated vs 403 forbidden, 409 conflict vs 422 unprocessable. Дизайн выглядит principled не intuitive.

В этом файле разберём REST theory глубоко. Что такое REST fundamentally. Fielding's six constraints с их implications. Richardson Maturity Model levels. HTTP methods detailed (semantics, safety, idempotency). HTTP status codes correctly. URI design conventions. Pagination strategies. Filtering plus sorting patterns. Versioning approaches. Auth mechanisms. HATEOAS. Content types. Caching mechanisms (Cache-Control, ETag, Last-Modified). Error response formats. Documentation (OpenAPI). Idempotency-Key pattern.

## Что такое REST

REST = Representational State Transfer. Определён Roy Fielding в диссертации 2000 года. НЕ протокол — архитектурный стиль.

Основные идеи. Ресурсы (не действия) центральное понятие. HTTP как протокол transport plus semantic layer. Stateless communication. Uniform interface between components.

Semantic difference vs RPC. RPC — «call remote method». REST — «manipulate remote resource». Resources как nouns, methods как verbs operating on them. Fundamentally different mental model.

## Fielding's six constraints

Client-Server. Разделение UI и данных. Клиент и сервер могут развиваться независимо. Different concerns, different lifecycles. Basis для scalability plus modular development.

Stateless. Каждый запрос содержит всю нужную информацию. Сервер не хранит сессию клиента.

Плюсы. Масштабируемость (any server can handle any request). Простота failover (no sticky sessions needed). Кэширование (predictable responses).

Минусы. Больше данных в каждом запросе (auth token везде). Trade-off scalability vs bandwidth.

Cacheable. Ответы должны явно указывать кэшируется или нет.

Через HTTP headers:
```
Cache-Control: max-age=3600
ETag: "abc123"
Last-Modified: Wed, 07 Sep 2026 12:00:00 GMT
```

Enables aggressive caching для read-heavy APIs. Reduces server load, improves latency.

Uniform Interface. Единообразный интерфейс между клиентом и сервером. Four sub-constraints.

Identification of resources — URI. Each resource has unique identifier.

Manipulation via representations — JSON/XML. Clients see representations not actual resources.

Self-descriptive messages — Content-Type, статусы. Messages contain enough metadata для understanding.

HATEOAS — hypermedia (ссылки в response). Discussed в section 11.

Layered System. Клиент не знает — работает с реальным сервером или через прокси / LB / gateway. Проксирование прозрачно.

Enables. Load balancing invisible. Caching proxies. Gateway aggregation. Multi-tier architectures.

Code-On-Demand (опционально). Сервер может отдавать executable code (JS для браузера). Часть web, редко для API. Only optional constraint.

## Richardson Maturity Model

Уровни REST-зрелости API (Leonard Richardson).

Level 0 — Swamp of POX. Один endpoint, всё через POST. По сути RPC поверх HTTP:
```
POST /api
Body: {"action": "getOrder", "id": 42}

POST /api
Body: {"action": "createOrder", "data": {...}}
```

SOAP по этому уровню (тоже RPC).

Level 1 — Resources. Разные URI для разных ресурсов, но всё ещё POST:
```
POST /orders/get         Body: {"id": 42}
POST /orders/create      Body: {...}
POST /orders/delete      Body: {"id": 42}
```

Уже лучше — есть resource concept. But not leveraging HTTP semantics.

Level 2 — HTTP Verbs. Правильно используются HTTP-методы plus статус-коды:
```
GET    /orders/42
POST   /orders          Body: {...}
PUT    /orders/42       Body: {...}
DELETE /orders/42
```

Большинство REST APIs — Level 2. Практически достаточно. HTTP используется по назначению.

Level 3 — HATEOAS. Response содержит ссылки для дальнейших действий:
```json
{
  "id": 42,
  "status": "NEW",
  "_links": {
    "self":   { "href": "/orders/42" },
    "cancel": { "href": "/orders/42/cancel" },
    "pay":    { "href": "/orders/42/pay" }
  }
}
```

Клиент следует за ссылками — не нужно знать URI заранее.

Теоретически правильно, практически редко используется. Клиенты обычно захардкоживают URLs. HATEOAS overhead usually не warranted.

## HTTP methods

GET получить ресурс. Никаких side effects:
```
GET /orders/42
```

Свойства. Safe — не изменяет state. Idempotent — многократный вызов = один эффект. Cacheable.

POST создать новый ресурс (или общее действие):
```
POST /orders
Body: {"customerId": "c1", ...}
```

Свойства. Not safe. Not idempotent (два POST = два ordera).

Response обычно 201 Created plus Location: /orders/43 header.

PUT полная замена ресурса:
```
PUT /orders/42
Body: {"id": 42, "customerId": "c1", "amount": 100, "status": "NEW"}
```

Все поля заменены. Отсутствующие — null / default.

Свойства. Not safe. Idempotent (тот же PUT дважды = тот же результат).

PATCH частичное обновление:
```
PATCH /orders/42
Body: {"status": "SHIPPED"}
```

Только указанные поля меняются.

Форматы. JSON Merge Patch (RFC 7396) — простой. JSON Patch (RFC 6902) — операции с path plus op.

Свойства. Not safe. Not idempotent обычно (depends on impl).

DELETE удалить ресурс:
```
DELETE /orders/42
```

Свойства. Not safe. Idempotent (второй DELETE на удалённое = 404 или 204).

HEAD как GET, но только headers (без body). Проверить существование / метаданные. Cache validation. Lightweight probe.

OPTIONS узнать какие методы поддерживаются. CORS preflight использует OPTIONS.

Safe vs Idempotent summary:

| Method | Safe | Idempotent |
|---|---|---|
| GET | да | да |
| HEAD | да | да |
| OPTIONS | да | да |
| PUT | нет | да |
| DELETE | нет | да |
| POST | нет | нет |
| PATCH | нет | нет (обычно) |

Safe — не меняет state. Idempotent — многократный вызов = один эффект.

Важно для retry. Retry OK для idempotent methods. Non-idempotent требуют Idempotency-Key или other deduplication.

## HTTP статус-коды

Разбираться в них — обязательно.

1xx Informational. 100 Continue — сервер получил headers, шли body (для больших uploads). 101 Switching Protocols — WebSocket upgrade.

2xx Success. 200 OK — стандартный success. 201 Created — ресурс создан (POST) plus Location header. 202 Accepted — принято, будет обработано (async). 204 No Content — success, нет body (DELETE, PUT without response). 206 Partial Content — range request (video, download).

3xx Redirection. 301 Moved Permanently — ресурс переехал навсегда. 302 Found — временный redirect. 304 Not Modified — с ETag/Last-Modified (клиент возьмёт из кэша). 307 Temporary Redirect — то же что 302 но метод сохраняется. 308 Permanent Redirect — то же что 301 но метод сохраняется.

4xx Client Error. 400 Bad Request — invalid syntax / validation. 401 Unauthorized — нет auth или невалидная (плохое название, надо было "Unauthenticated"). 403 Forbidden — auth OK, но нет прав. 404 Not Found — ресурс не существует. 405 Method Not Allowed — метод не поддерживается. 406 Not Acceptable — не можем отдать в запрошенном формате (Accept). 408 Request Timeout — клиент не досылал. 409 Conflict — состояние конфликтует (dup key, версия). 410 Gone — было, но нет (навсегда, в отличие от 404). 413 Payload Too Large. 415 Unsupported Media Type — Content-Type не поддерживается. 422 Unprocessable Entity — синтаксис OK, но semantically invalid. 429 Too Many Requests — rate limit.

5xx Server Error. 500 Internal Server Error — общая ошибка сервера. 501 Not Implemented — метод не реализован. 502 Bad Gateway — upstream вернул invalid. 503 Service Unavailable — сервер overloaded / down. 504 Gateway Timeout — upstream timeout.

Как выбирать. Ресурс не найден — 404. Валидация упала — 400 или 422. Не аутентифицирован — 401. Нет прав — 403. Дубликат (unique) — 409. Rate limit — 429. Внутренняя ошибка — 500. Downstream down — 503 (или 502).

НЕ — 500 с {"error": "not found"}. Используй правильный код. Consistent status usage enables client automation.

## URI дизайн

Nouns not verbs. GET /orders/42 correct. GET /getOrder?id=42 anti-pattern.

POST /orders correct. POST /createOrder anti-pattern.

Plural. /orders/42 correct. /order/42 anti-pattern. Коллекции — множественное.

Hierarchy:
```
/customers/1/orders           — все ордера клиента 1
/customers/1/orders/42        — конкретный ордер
/customers/1/orders/42/items  — items ордера
```

Логично, читабельно. Reflects business relationships.

Kebab-case. /order-items correct. /orderItems anti-pattern (camelCase URLs). /order_items anti-pattern (underscores).

По convention для URLs. HTTP spec case-sensitive но kebab-case standard convention.

Query parameters для фильтрации, sort, pagination:
```
GET /orders?status=NEW&customerId=c1&sort=createdAt,desc&page=0&size=20
```

Verbs как sub-resources когда действие не CRUD:
```
POST /orders/42/cancel      — отменить
POST /orders/42/pay         — оплатить
POST /users/authenticate    — login
```

Не всё вписывается в CRUD. Sub-resource с verb-noun. Reasonable pragmatism.

Bad practices. Смешанные URL — /orders/42/getStatus (get не нужен). Технические детали — /api/v1/db/select/orders. Файловые расширения — /orders.json (используй Accept).

## Pagination

Offset-based:
```
GET /orders?page=0&size=20
```

Response:
```json
{
  "content": [...],
  "totalElements": 1000,
  "totalPages": 50,
  "number": 0,
  "size": 20
}
```

Проблема. OFFSET на больших страницах медленный (см. файл 29). При изменении данных — sliding (страница 5 после INSERT'а покажет часть страницы 4). Correctness issue при mutations during pagination.

Keyset (cursor-based):
```
GET /orders?cursor=eyJjcmVhdGVkQXQiOiIyMDI2LTA5LTA3In0
```

Cursor — encoded position (last created_at plus id).

Response:
```json
{
  "content": [...],
  "nextCursor": "eyJjcmVhdGVkQXQiOi..."
}
```

Плюсы. Быстрее (WHERE created_at < X использует индекс efficiently). Стабильно при изменениях (cursor position fixed).

Минусы. Не можешь прыгать сразу на страницу 50. Sequential navigation only.

Link header (RFC 5988):
```
Link: </orders?page=2>; rel="next",
      </orders?page=0>; rel="prev",
      </orders?page=50>; rel="last"
```

GitHub API использует. HATEOAS-lite. Client follows links.

## Filtering и sorting

Filtering:
```
GET /orders?status=NEW
GET /orders?status=NEW,SHIPPED   — множество
GET /orders?minAmount=100&maxAmount=1000
GET /orders?createdAfter=2026-01-01
```

Custom queries сложнее. GraphQL или dedicated search endpoint (POST /orders/search с complex body).

Sorting:
```
GET /orders?sort=createdAt
GET /orders?sort=createdAt,desc
GET /orders?sort=status,asc&sort=createdAt,desc   — множественная сортировка
```

Spring Data JPA Pageable парсит автоматически. Ordering multi-column supported.

Field selection:
```
GET /orders/42?fields=id,status,total
```

Возвращает только указанные поля. Экономит трафик. Sparse fieldsets.

GraphQL — более гибко, но сложнее. Different query language.

## Versioning

Как менять API без ломки клиентов. Discussed в файле 49.

URI versioning:
```
/v1/orders
/v2/orders
```

Простой, видимый в logs. Easy для routing decisions.

Header versioning:
```
Accept: application/vnd.myapi.v2+json
```

Чище URL, менее видимый в logs. HTTP content negotiation semantics.

Query parameter:
```
/orders?v=2
```

Легко для тестирования, некрасиво. Sometimes anti-pattern viewed.

Правила compatibility. Adding fields — safe (backward compatible clients ignore new). Removing / renaming — breaking. Changing types / semantics — breaking. Deprecated — предупредить, потом удалить в major version.

Deprecation cycles важны. Never break without notice. Long transition periods (6+ months typical).

## Auth

Basic Auth:
```
Authorization: Basic <base64(user:pass)>
```

Всегда с HTTPS. Простой, но раскрывает credentials. Not preferred для APIs.

Bearer / JWT:
```
Authorization: Bearer eyJ...
```

Стандарт для API. Detailed в файле 25 oauth2-oidc-theory.

API Key:
```
X-API-Key: abc123
```

Для service-to-service, публичных APIs. Simple но limited (no user context typically).

OAuth 2.0. Delegated authorization. Comprehensive в файле 25. Standard для most modern APIs.

## HATEOAS

Level 3 Richardson:
```json
{
  "id": 42,
  "status": "NEW",
  "customerId": "c1",
  "_links": {
    "self":     { "href": "/orders/42" },
    "customer": { "href": "/customers/c1" },
    "cancel":   { "href": "/orders/42/cancel" },
    "pay":      { "href": "/orders/42/pay" }
  }
}
```

Клиент следует за ссылками — не хардкодит URL.

Spring HATEOAS:
```java
EntityModel<Order> model = EntityModel.of(order,
    linkTo(methodOn(OrderController.class).get(order.getId())).withSelfRel(),
    linkTo(methodOn(OrderController.class).cancel(order.getId())).withRel("cancel"));
```

Practice. Большинство REST APIs — Level 2 без HATEOAS. Level 3 сложно, редко нужно. Practical overhead usually не warranted.

## Content Types

Common types. application/json — де-факто стандарт. application/xml — legacy / SOAP-подобные. application/x-www-form-urlencoded — HTML forms. multipart/form-data — file uploads. text/plain, text/html. application/octet-stream — binary. application/problem+json — RFC 7807 errors. application/vnd.company.resource+json — vendor-specific.

Accept header — что клиент хочет получить. Content-Type — что клиент шлёт / сервер возвращает.

## Caching

Cache-Control:
```
Cache-Control: max-age=3600, public
Cache-Control: no-cache, no-store, must-revalidate
Cache-Control: private, max-age=0
```

max-age=N — валидно N секунд. public — CDN и proxies тоже могут. private — только клиент. no-cache — всегда re-validate с сервером. no-store — вообще не кэшировать.

ETag уникальный идентификатор версии ресурса:
```
Response:
    ETag: "v1"
    Cache-Control: max-age=0

Client cache:
    ETag "v1"

Next request:
    If-None-Match: "v1"

Server:
    Если не изменилось → 304 Not Modified (без body)
    Иначе → 200 + новое ETag + body
```

Экономит трафик — сервер отвечает 304 если не изменилось. Bandwidth savings substantial для repeated reads.

Last-Modified:
```
Response:
    Last-Modified: Wed, 07 Sep 2026 12:00:00 GMT

Next request:
    If-Modified-Since: Wed, 07 Sep 2026 12:00:00 GMT

Server:
    Если не изменилось → 304
```

Аналогично ETag, но по времени. Precision less than ETag но simpler.

## Error responses

Стандартный формат — важно для клиентов.

Простой:
```json
{
  "error": "not_found",
  "message": "Order 42 not found"
}
```

Ad hoc. Каждое API свой формат.

Problem+JSON (RFC 7807):
```json
{
  "type": "https://example.com/errors/not-found",
  "title": "Order not found",
  "status": 404,
  "detail": "Order with id 42 not found",
  "instance": "/orders/42",
  "orderId": 42
}
```

Спецификация. Standard fields plus extension fields. Consistent format across services.

Spring 6 / Boot 3 поддерживает ProblemDetail. Native support для стандартного format.

Validation errors:
```json
{
  "status": 400,
  "title": "Validation failed",
  "errors": [
    { "field": "customerId", "message": "required" },
    { "field": "amount", "message": "must be > 0" }
  ]
}
```

Field-level errors для форм. Enables client-side field highlighting.

## Documentation

OpenAPI (formerly Swagger). Стандарт описания REST APIs. YAML или JSON format:
```yaml
openapi: 3.0.0
info:
  title: Orders API
  version: 1.0.0
paths:
  /orders:
    get:
      summary: List orders
      parameters:
        - name: status
          in: query
          schema:
            type: string
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Order'
components:
  schemas:
    Order:
      type: object
      properties:
        id: { type: integer }
        status: { type: string, enum: [NEW, SHIPPED, CANCELLED] }
```

Springdoc-openapi автогенерирует OpenAPI из Spring контроллеров:
```gradle
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0'
```

URLs. /v3/api-docs — JSON описание. /swagger-ui.html — интерактивный UI для тестирования.

Аннотации:
```java
@Tag(name = "Orders")
@RestController
class OrderController {

    @Operation(summary = "Get order by id")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Found"),
        @ApiResponse(responseCode = "404", description = "Not found")
    })
    @GetMapping("/orders/{id}")
    public Order get(@Parameter(description = "Order id") @PathVariable Long id) {
        return service.find(id);
    }
}
```

Auto-generated documentation. Kept в sync с actual code.

## Idempotency-Key

Для POST с retry (см. файл 51):
```
POST /transfer
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Body: {"from": "c1", "to": "c2", "amount": 100}
```

Server. Первый вызов — обработать, сохранить result по key. Повторный с тем же key — вернуть сохранённый result.

Защита от дублей при retry. Stripe, банковские APIs. Critical для non-idempotent POST operations.

## Best practices

Nouns not verbs, plural. HTTP methods правильно (GET/POST/PUT/PATCH/DELETE). Правильные статус-коды (не всё 200 или 500). JSON default. XML только legacy. snake_case или camelCase — выбрать один, не смешивать. Pagination обязательна для коллекций. Versioning с самого начала. HTTPS обязательно. Auth — JWT Bearer для APIs. Rate limiting на публичных. OpenAPI для документации. Consistent error format (Problem+JSON). Idempotency-Key для критичных POST. Не возвращай Entity — DTO. CORS правильно (не звёздочка в prod).

## Итоги

REST архитектурный стиль на HTTP. Not protocol.

Six constraints Fielding — client-server, stateless, cacheable, uniform interface, layered, code-on-demand.

Richardson Maturity Model. Level 0 POX (RPC), Level 1 Resources, Level 2 Verbs, Level 3 HATEOAS. Most APIs Level 2.

HTTP methods semantics. GET safe idempotent. POST not idempotent (create). PUT idempotent (full replace). PATCH partial (usually not idempotent). DELETE idempotent.

Status codes reflect specific situations. 200/201/204 success. 400/401/403/404/409/422/429 client errors. 500/502/503/504 server errors. Correct usage enables client automation.

URI design. Nouns not verbs, plural. Hierarchy through paths. Kebab-case URLs. Query params для filtering/sorting/pagination.

Pagination options. Offset (simple но slow deep pages plus sliding issue). Keyset/cursor (efficient plus stable). Link header for HATEOAS-lite.

Filtering, sorting, field selection through query params. Complex queries через dedicated search endpoint или GraphQL.

Versioning. URI, header, или query param. Backward-compatible additions только. Deprecation cycles.

Auth mechanisms. Bearer/JWT стандарт для APIs. Basic только с HTTPS. API keys для service-to-service.

HATEOAS Level 3. Theoretical правильно но редко practical. Most clients hardcode URLs.

Caching через Cache-Control, ETag, Last-Modified. 304 Not Modified saves bandwidth.

Error responses. Problem+JSON (RFC 7807) стандарт. Field-level errors для validation.

OpenAPI (Swagger) для documentation. Springdoc-openapi auto-generation из Spring controllers.

Idempotency-Key для non-idempotent POST retry safety. Financial APIs standard.

Дальше — SOAP API как contrast к REST. XML-based protocol с contract-first approach, WS-* standards.
