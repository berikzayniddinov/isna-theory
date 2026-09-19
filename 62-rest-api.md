# 62. REST API теория

Что такое REST, HTTP-методы, статус-коды, best practices.

---

## 1. Что такое REST

**REST** = **RE**presentational **S**tate **T**ransfer.

Определён Roy Fielding в диссертации 2000 года. НЕ протокол, а **архитектурный стиль**.

Основные идеи:
- Ресурсы (не действия) — центральное понятие.
- HTTP как протокол.
- Stateless communication.
- Uniform interface.

---

## 2. Ключевые принципы (Fielding)

### 2.1 Client-Server

Разделение UI и данных. Клиент и сервер могут развиваться независимо.

### 2.2 Stateless

**Каждый запрос содержит всю нужную информацию**. Сервер не хранит сессию клиента.

Плюсы:
- Масштабируемость.
- Простота failover.
- Кэширование.

Минусы:
- Больше данных в каждом запросе (auth token везде).

### 2.3 Cacheable

Ответы должны явно указывать: **кэшируется или нет**.

Через HTTP headers:
```
Cache-Control: max-age=3600
ETag: "abc123"
Last-Modified: Wed, 07 Sep 2026 12:00:00 GMT
```

### 2.4 Uniform Interface

Единообразный интерфейс между клиентом и сервером:
- **Identification of resources** — URI.
- **Manipulation via representations** — JSON/XML.
- **Self-descriptive messages** — Content-Type, статусы.
- **HATEOAS** — hypermedia (ссылки в response).

### 2.5 Layered System

Клиент не знает — работает с реальным сервером или через прокси / LB / gateway. Проксирование прозрачно.

### 2.6 Code-On-Demand (опционально)

Сервер может отдавать executable code (JS для браузера). Часть web, редко для API.

---

## 3. Richardson Maturity Model

Уровни REST-зрелости API (Leonard Richardson).

### 3.1 Level 0 — Swamp of POX

Один endpoint, всё через POST. По сути RPC поверх HTTP.

```
POST /api
Body: {"action": "getOrder", "id": 42}

POST /api
Body: {"action": "createOrder", "data": {...}}
```

SOAP по этому уровню (тоже RPC).

### 3.2 Level 1 — Resources

Разные URI для разных ресурсов, но всё ещё POST.

```
POST /orders/get         Body: {"id": 42}
POST /orders/create      Body: {...}
POST /orders/delete      Body: {"id": 42}
```

Уже лучше — есть resource concept.

### 3.3 Level 2 — HTTP Verbs

Правильно используются HTTP-методы + статус-коды.

```
GET    /orders/42
POST   /orders          Body: {...}
PUT    /orders/42       Body: {...}
DELETE /orders/42
```

**Большинство "REST APIs" — Level 2**. Практически достаточно.

### 3.4 Level 3 — HATEOAS

Response содержит **ссылки** для дальнейших действий.

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

Клиент **следует за ссылками** — не нужно знать URI заранее.

**Теоретически правильно**, практически редко используется. Клиенты обычно захардкоживают URLs.

---

## 4. HTTP-методы (verbs)

### 4.1 GET

Получить ресурс. **Никаких side effects**.

```
GET /orders/42
```

Свойства:
- **Safe** — не изменяет state.
- **Idempotent** — многократный вызов = один эффект.
- **Cacheable**.

### 4.2 POST

Создать новый ресурс (или общее действие).

```
POST /orders
Body: {"customerId": "c1", ...}
```

Свойства:
- **Not safe**.
- **Not idempotent** (два POST = два ordera).

Response обычно **201 Created** + `Location: /orders/43` header.

### 4.3 PUT

**Полная замена** ресурса.

```
PUT /orders/42
Body: {"id": 42, "customerId": "c1", "amount": 100, "status": "NEW"}
```

Все поля заменены. Отсутствующие → null / default.

Свойства:
- **Not safe**.
- **Idempotent** (тот же PUT дважды = тот же результат).

### 4.4 PATCH

**Частичное обновление**.

```
PATCH /orders/42
Body: {"status": "SHIPPED"}
```

Только указанные поля меняются.

Форматы:
- **JSON Merge Patch** (RFC 7396) — простой:
  ```json
  {"status": "SHIPPED"}
  ```
- **JSON Patch** (RFC 6902) — операции:
  ```json
  [
    {"op": "replace", "path": "/status", "value": "SHIPPED"},
    {"op": "remove", "path": "/reason"}
  ]
  ```

Свойства:
- **Not safe**.
- **Not idempotent** обычно (depends on impl).

### 4.5 DELETE

Удалить ресурс.

```
DELETE /orders/42
```

Свойства:
- **Not safe**.
- **Idempotent** (второй DELETE на удалённое = 404 или 204).

### 4.6 HEAD

Как GET, но **только headers** (без body). Проверить существование / метаданные.

### 4.7 OPTIONS

Узнать какие методы поддерживаются:
```
OPTIONS /orders/42
Response headers:
Allow: GET, PUT, DELETE
```

CORS preflight использует OPTIONS.

### 4.8 Safe vs Idempotent

| Method | Safe | Idempotent |
|---|---|---|
| GET | ✅ | ✅ |
| HEAD | ✅ | ✅ |
| OPTIONS | ✅ | ✅ |
| PUT | ❌ | ✅ |
| DELETE | ❌ | ✅ |
| POST | ❌ | ❌ |
| PATCH | ❌ | ❌ (обычно) |

**Safe** — не меняет state.
**Idempotent** — многократный вызов = один эффект.

Важно для retry: retry OK для idempotent methods.

---

## 5. HTTP статус-коды

Разбираться в них — обязательно.

### 5.1 1xx — Informational

- **100 Continue** — сервер получил headers, шли body (для больших uploads).
- **101 Switching Protocols** — WebSocket upgrade.

### 5.2 2xx — Success

- **200 OK** — стандартный success.
- **201 Created** — ресурс создан (POST). + `Location` header.
- **202 Accepted** — принято, будет обработано (async).
- **204 No Content** — success, нет body (DELETE, PUT without response).
- **206 Partial Content** — range request (video, download).

### 5.3 3xx — Redirection

- **301 Moved Permanently** — ресурс переехал навсегда.
- **302 Found** — временный redirect.
- **304 Not Modified** — с ETag/Last-Modified (клиент возьмёт из кэша).
- **307 Temporary Redirect** — то же что 302, но метод сохраняется.
- **308 Permanent Redirect** — то же что 301, но метод сохраняется.

### 5.4 4xx — Client Error

- **400 Bad Request** — invalid syntax / validation.
- **401 Unauthorized** — нет auth или невалидная (плохое название, надо было "Unauthenticated").
- **402 Payment Required** — редко.
- **403 Forbidden** — auth OK, но нет прав.
- **404 Not Found** — ресурс не существует.
- **405 Method Not Allowed** — метод не поддерживается (PUT когда только GET).
- **406 Not Acceptable** — не можем отдать в запрошенном формате (Accept).
- **408 Request Timeout** — клиент не досылал.
- **409 Conflict** — состояние конфликтует (dup key, версия).
- **410 Gone** — было, но нет (навсегда, в отличие от 404).
- **413 Payload Too Large** — тело слишком большое.
- **415 Unsupported Media Type** — Content-Type не поддерживается.
- **422 Unprocessable Entity** — синтаксис OK, но semantically invalid.
- **429 Too Many Requests** — rate limit.

### 5.5 5xx — Server Error

- **500 Internal Server Error** — общая ошибка сервера.
- **501 Not Implemented** — метод не реализован.
- **502 Bad Gateway** — upstream вернул invalid (gateway/proxy).
- **503 Service Unavailable** — сервер overloaded / down.
- **504 Gateway Timeout** — upstream timeout.

### 5.6 Как выбирать

**Стандарт**:
- Ресурс не найден → **404**.
- Валидация упала → **400** или **422**.
- Не аутентифицирован → **401**.
- Нет прав → **403**.
- Дубликат (unique) → **409**.
- Rate limit → **429**.
- Внутренняя ошибка → **500**.
- Downstream down → **503** (или **502**).

**НЕ**: `500 { "error": "not found" }`. Используй правильный код.

---

## 6. URI дизайн

### 6.1 Nouns, not verbs

- ✅ `GET /orders/42`
- ❌ `GET /getOrder?id=42`

- ✅ `POST /orders`
- ❌ `POST /createOrder`

### 6.2 Plural

- ✅ `/orders/42`
- ❌ `/order/42`

Коллекции — множественное.

### 6.3 Hierarchy

```
/customers/1/orders           — все ордера клиента 1
/customers/1/orders/42        — конкретный ордер
/customers/1/orders/42/items  — items ордера
```

Логично, читабельно.

### 6.4 Kebab-case

- ✅ `/order-items`
- ❌ `/orderItems`
- ❌ `/order_items`

По convention для URLs.

### 6.5 Query parameters

Для фильтрации, sort, pagination:
```
GET /orders?status=NEW&customerId=c1&sort=createdAt,desc&page=0&size=20
```

### 6.6 Verbs как sub-resources

Иногда действие не CRUD:
```
POST /orders/42/cancel      — отменить
POST /orders/42/pay         — оплатить
POST /users/authenticate    — login
```

Не всё вписывается в CRUD — использовать sub-resource с verb-nouns.

### 6.7 Bad practices

- ❌ Смешанные URL: `/orders/42/getStatus` (get не нужен).
- ❌ Технические детали: `/api/v1/db/select/orders`.
- ❌ Файловые расширения: `/orders.json` (используй Accept).

---

## 7. Pagination

### 7.1 Offset-based

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

**Проблема**: OFFSET на больших страницах медленный (см. `29-postgresql-spring-hikaricp.md`). При изменении данных — sliding (страница 5 после INSERT'а покажет часть страницы 4).

### 7.2 Keyset (cursor-based)

```
GET /orders?cursor=eyJjcmVhdGVkQXQiOiIyMDI2LTA5LTA3In0
```

Cursor — encoded position (last created_at + id).

Response:
```json
{
  "content": [...],
  "nextCursor": "eyJjcmVhdGVkQXQiOi..."
}
```

Плюсы: быстрее, стабильно при изменениях.

Минусы: не можешь прыгать сразу на страницу 50.

### 7.3 Link header (RFC 5988)

```
Link: </orders?page=2>; rel="next",
      </orders?page=0>; rel="prev",
      </orders?page=50>; rel="last"
```

GitHub API использует.

---

## 8. Filtering / sorting

### 8.1 Filtering

```
GET /orders?status=NEW
GET /orders?status=NEW,SHIPPED   — множество
GET /orders?minAmount=100&maxAmount=1000
GET /orders?createdAfter=2026-01-01
```

Custom queries сложнее — GraphQL / dedicated search endpoint (`POST /orders/search`).

### 8.2 Sorting

```
GET /orders?sort=createdAt
GET /orders?sort=createdAt,desc
GET /orders?sort=status,asc&sort=createdAt,desc   — множественная сортировка
```

Spring Data JPA `Pageable` парсит автоматически.

### 8.3 Field selection

```
GET /orders/42?fields=id,status,total
```

Возвращает только указанные поля. Экономит трафик.

GraphQL — более гибко, но сложнее.

---

## 9. Versioning

Как менять API без ломки клиентов. Разбирали в `49-microservices-decomposition.md`.

### 9.1 URI versioning

```
/v1/orders
/v2/orders
```

Простой, видимый.

### 9.2 Header versioning

```
Accept: application/vnd.myapi.v2+json
```

Чище URL, менее видимый.

### 9.3 Query parameter

```
/orders?v=2
```

Легко для тестирования, некрасиво.

### 9.4 Правила

- **Adding fields** — safe (backward compatible).
- **Removing / renaming** — breaking.
- **Changing types / semantics** — breaking.
- **Deprecated** — предупредить, потом удалить в major version.

---

## 10. Auth

### 10.1 Basic Auth

```
Authorization: Basic <base64(user:pass)>
```

Всегда с HTTPS. Простой, но раскрывает credentials.

### 10.2 Bearer / JWT

```
Authorization: Bearer eyJ...
```

Стандарт для API. См. `25-oauth2-oidc-theory.md`.

### 10.3 API Key

```
X-API-Key: abc123
```

Для service-to-service, публичных APIs.

### 10.4 OAuth 2.0

Разбирали в `25-oauth2-oidc-theory.md`.

---

## 11. HATEOAS

Level 3 Richardson.

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

Клиент **следует за ссылками** — не хардкодит URL.

Spring HATEOAS:
```java
EntityModel<Order> model = EntityModel.of(order,
    linkTo(methodOn(OrderController.class).get(order.getId())).withSelfRel(),
    linkTo(methodOn(OrderController.class).cancel(order.getId())).withRel("cancel"));
```

**Практика**: большинство REST APIs — Level 2 без HATEOAS. Level 3 сложно, редко нужно.

---

## 12. Content Types

- **`application/json`** — де-факто стандарт.
- **`application/xml`** — legacy / SOAP-подобные.
- **`application/x-www-form-urlencoded`** — HTML forms.
- **`multipart/form-data`** — file uploads.
- **`text/plain`**, **`text/html`**.
- **`application/octet-stream`** — binary.
- **`application/problem+json`** — RFC 7807 errors.
- **`application/vnd.company.resource+json`** — vendor-specific.

`Accept` header — что клиент **хочет получить**.
`Content-Type` — что клиент **шлёт** / сервер **возвращает**.

---

## 13. Caching

### 13.1 Cache-Control

```
Cache-Control: max-age=3600, public
Cache-Control: no-cache, no-store, must-revalidate
Cache-Control: private, max-age=0
```

- `max-age=N` — валидно N секунд.
- `public` — CDN и proxies тоже могут.
- `private` — только клиент.
- `no-cache` — всегда re-validate с сервером.
- `no-store` — вообще не кэшировать.

### 13.2 ETag

Уникальный идентификатор версии ресурса.

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

Экономит трафик — сервер отвечает 304 если не изменилось.

### 13.3 Last-Modified

```
Response:
    Last-Modified: Wed, 07 Sep 2026 12:00:00 GMT

Next request:
    If-Modified-Since: Wed, 07 Sep 2026 12:00:00 GMT

Server:
    Если не изменилось → 304
```

Аналогично ETag, но по времени.

---

## 14. Error responses

Стандартный формат — важно для клиентов.

### 14.1 Простой

```json
{
  "error": "not_found",
  "message": "Order 42 not found"
}
```

### 14.2 Problem+JSON (RFC 7807)

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

Spring 6 / Boot 3 поддерживает `ProblemDetail`.

### 14.3 Validation errors

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

---

## 15. Documentation

### 15.1 OpenAPI / Swagger

Стандарт описания REST APIs (YAML/JSON):

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

### 15.2 Springdoc-openapi

Автогенерирует OpenAPI из Spring контроллеров:
```gradle
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0'
```

URL:
- `/v3/api-docs` — JSON описание.
- `/swagger-ui.html` — интерактивный UI для тестирования.

### 15.3 Аннотации

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

---

## 16. Idempotency-Key

Для POST с retry (см. `51-outbox-inbox-pattern.md`):

```
POST /transfer
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Body: {"from": "c1", "to": "c2", "amount": 100}
```

Server:
- Первый вызов → обработать, сохранить result по key.
- Повторный с тем же key → вернуть сохранённый result.

Защита от дублей при retry. Stripe, банковские APIs.

---

## 17. Best practices

1. **Nouns not verbs**, plural.
2. **HTTP methods** правильно (GET/POST/PUT/PATCH/DELETE).
3. **Правильные статус-коды** (не всё 200 или 500).
4. **JSON** default; XML только legacy.
5. **snake_case** или **camelCase** — выбрать один, не смешивать.
6. **Pagination** обязательна для коллекций.
7. **Versioning** с самого начала.
8. **HTTPS** обязательно.
9. **Auth**: JWT Bearer для APIs.
10. **Rate limiting** на публичных.
11. **OpenAPI** для документации.
12. **Consistent error format** (Problem+JSON).
13. **Idempotency-Key** для критичных POST.
14. **Не возвращай Entity** — DTO.
15. **CORS** правильно (не `*` в prod).

---

## 18. Собесные вопросы

1. **Что такое REST?** — Архитектурный стиль на HTTP; resources, stateless, uniform interface.
2. **Кто автор REST?** — Roy Fielding, диссертация 2000.
3. **Разница REST и SOAP?** — REST = архитектурный стиль на HTTP; SOAP = протокол на XML.
4. **Richardson Maturity Model — уровни?** — 0 POX, 1 Resources, 2 Verbs, 3 HATEOAS.
5. **Safe vs Idempotent methods?** — Safe: не меняют state (GET); Idempotent: многократный = один эффект (PUT, DELETE).
6. **Что вернуть при создании ресурса?** — 201 Created + Location header.
7. **Разница PUT и PATCH?** — PUT полная замена; PATCH частичное обновление.
8. **Разница 401 и 403?** — 401: не аутентифицирован; 403: аутентифицирован, нет прав.
9. **Что такое HATEOAS?** — Response содержит ссылки для дальнейших действий (Level 3).
10. **Как версионировать API?** — URI (/v1), header (Accept), query param. Backward compatible additions.
11. **Что такое ETag?** — Cache validator; If-None-Match → 304 если не изменилось.
12. **Content-Type vs Accept?** — Content-Type что шлёшь; Accept что хочешь получить.
13. **Что такое Idempotency-Key?** — Header для дедупликации POST на server side.
14. **Стандарт для ошибок?** — Problem+JSON (RFC 7807).
15. **Как документировать REST API?** — OpenAPI (Swagger); springdoc-openapi для авто-генерации.

---

## Итог

- **REST** = архитектурный стиль на HTTP.
- **6 принципов**: client-server, stateless, cacheable, uniform, layered, code-on-demand.
- **HTTP methods**: GET (safe/idempotent) / POST (dangerous) / PUT (idempotent full) / PATCH (partial) / DELETE (idempotent).
- **Статус-коды** правильно: 200/201/204/400/401/403/404/409/422/429/500/503.
- **URI design**: nouns, plural, hierarchy.
- **Pagination**, **versioning**, **auth**, **caching** — обязательно.
- **OpenAPI** для документации.
- **Problem+JSON** для ошибок.

Следующий — `63-soap-api.md`.
