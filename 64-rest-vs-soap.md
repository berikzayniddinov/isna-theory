# 64. REST vs SOAP — развёрнутое сравнение

Разбор различий. Когда что выбирать.

---

## 1. Природа

**REST** — **архитектурный стиль**. Набор принципов, не протокол.

**SOAP** — **протокол**. Строгие правила формата и обмена.

Разница фундаментальная: REST — "как строить APIs", SOAP — "как форматировать сообщения".

---

## 2. Основные различия — таблица

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

---

## 3. Сравнение сообщений

Одна и та же операция — создать order.

### 3.1 REST + JSON

**Request**:
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

**Response**:
```
HTTP/1.1 201 Created
Location: /orders/42
Content-Type: application/json

{
  "id": 42,
  "status": "NEW"
}
```

Размер: ~200 bytes.

### 3.2 SOAP + XML

**Request**:
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

**Response**:
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

Размер: ~800-1500 bytes (в 5-10× больше).

---

## 4. Использование HTTP

### 4.1 REST — использует полностью

- **Methods**: GET/POST/PUT/PATCH/DELETE — семантика.
- **Status codes**: 200/201/404/409/500 — разные ситуации.
- **Headers**: Cache-Control, ETag, Location.
- **Content Negotiation** через Accept.

### 4.2 SOAP — только транспорт

- Всё через **POST**.
- Всегда status **200** (или 500 for fault).
- Информация о fault — в **body** (не в HTTP).
- `SOAPAction` header уточняет операцию.

По сути SOAP игнорирует HTTP-возможности. Мог бы быть на любом транспорте.

---

## 5. Contract

### 5.1 REST — обычно code-first

Пишешь Spring controller → генерируется OpenAPI (через springdoc).

Опционально. Многие REST APIs без формального contract.

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

### 5.2 SOAP — обязательно contract-first

1. Пишешь WSDL.
2. Генерируешь код (wsimport → Java classes).
3. Реализуешь методы.

Или code-first:
```java
@WebService
public class OrderService {
    public Long createOrder(...) { }
}
```

Spring генерирует WSDL из аннотаций.

### 5.3 Runtime validation

- **SOAP**: сообщения валидируются против XSD **автоматически**. Invalid = SOAP fault.
- **REST**: валидация ручная (Bean Validation через `@Valid`). Или через OpenAPI validator в middleware.

Строгость SOAP — плюс для enterprise integration (партнёры хотят "гарантию").

---

## 6. Security

### 6.1 REST — Transport-level

Обычно:
- **HTTPS (TLS)** — encryption in-transit.
- **JWT / OAuth 2.0** для authentication.
- **API keys** для service-to-service.

Security **между** клиентом и сервером. Через прокси / gateway — прерывается TLS.

Простой, стандартный.

### 6.2 SOAP — Message-level (WS-Security)

**WS-Security** позволяет:
- Encrypt конкретные части XML.
- Sign отдельные элементы.
- Username tokens в SOAP Header.
- SAML tokens.

Плюс: **end-to-end** через intermediary'ов. Клиент → Gateway → Server — сообщение остаётся signed/encrypted всю дорогу.

Минус: сложно, много boilerplate.

---

## 7. Transactions

### 7.1 REST

- **Локальная tx** — обычная @Transactional.
- **Distributed** — Saga, Outbox, Idempotency (см. `50-saga-pattern.md`, `51-outbox-inbox-pattern.md`).
- **Нет стандарта** для distributed tx через REST.

### 7.2 SOAP

- **WS-AtomicTransaction (WS-AT)** — 2PC поверх SOAP.
- **WS-BusinessActivity (WS-BA)** — long-running compensations (аналог Saga).

Enterprise-ready, но сложно + требует cross-vendor совместимости.

На практике даже в SOAP-мире 2PC редко.

---

## 8. Performance

### 8.1 Bandwidth

REST + JSON — **компактно**.

SOAP + XML — **verbose** (namespaces, envelope, boilerplate). В 5-10× больше.

### 8.2 CPU (parsing)

- **JSON** — быстро парсится (Jackson).
- **XML** — медленнее (SAX/DOM/StAX).

### 8.3 Latency

- **REST** — быстрее (компактнее, меньше overhead).
- **SOAP** — медленнее.

Для high-throughput → REST.

### 8.4 Caching

- **REST** — GET требования кэшируются HTTP-кэшом, CDN.
- **SOAP** — всё POST, no caching.

Big deal для read-heavy APIs.

---

## 9. Developer experience

### 9.1 REST

- Простой: браузер, curl, Postman.
- JSON — читаемо.
- Легко debug.
- Быстрый feedback.
- Онбординг за часы.

### 9.2 SOAP

- Нужен WSDL, XSD tooling.
- XML сложно писать/читать.
- SoapUI для тестирования.
- Дни на первый вызов.
- Онбординг за недели.

Отсюда — популярность REST для web / mobile.

---

## 10. Что где используется — примеры

### 10.1 REST

- **Публичные web APIs**: GitHub, Twitter, Stripe, Slack.
- **Mobile apps**.
- **SPA / React**.
- **Микросервисы** (внутренняя коммуникация).

### 10.2 SOAP

- **Banking** (SWIFT, ISO 20022 XML-based).
- **Government** (Казахстан ИСНА, ЕС ecosystems).
- **Telecom** (OSS/BSS).
- **Enterprise B2B** (EDI, SAP).
- **Legacy integrations**.

---

## 11. Когда что выбрать

### 11.1 REST — большинство случаев

- **Микросервисы**.
- **Публичные APIs**.
- **Mobile / SPA backends**.
- **Simple CRUD**.
- **High-throughput read-heavy**.

По default — REST.

### 11.2 SOAP — специфичные

- **Требование партнёра** (банк / gov).
- **Legacy integration** (существующий SOAP).
- **Строгий contract** обязателен.
- **WS-Security** end-to-end требуется.
- **Distributed transactions** (2PC — редко).
- **Complex enterprise workflows** (BPEL).

---

## 12. Гибридные подходы

Часто **оба** в одной системе:
- **Внешние партнёры** — SOAP (по их требованию).
- **Внутренние сервисы** — REST.
- **Adapter** переводит SOAP ↔ REST.

Так в ИСНА:
- **ЕСБ / SOAP-шина** для интеграции с внешними гос-системами.
- **Внутренние микросервисы** — REST + Feign.
- **Gateway (Zuul)** — L7 маршрутизация.

---

## 13. Modern alternatives

Не только REST vs SOAP.

### 13.1 gRPC

- Protobuf binary.
- HTTP/2.
- Contract-first (.proto).
- Быстрее REST.
- Streaming.

Плюсы SOAP (strong contract, performance) без XML overhead.

Использование: internal microservices, high-throughput.

### 13.2 GraphQL

- Query language.
- Client запрашивает какие поля нужны.
- Один endpoint.

Плюсы: гибкие read (no over/under fetching).

Использование: BFF для сложных UIs.

### 13.3 JSON-RPC / XML-RPC

Legacy RPC.

### 13.4 WebSocket / SSE

Для real-time.

**Правило современности**: **REST** — стандарт, **gRPC** — internal, **GraphQL** — flexible reads, **SOAP** — только когда обязательно.

---

## 14. Миграция SOAP → REST

Если legacy SOAP и хочешь на REST:

1. **Не переписывай сразу** — Strangler pattern.
2. **Facade** — REST-фасад перед SOAP-backend.
3. **Постепенно** — новые features REST, старые остаются SOAP.
4. **Deprecate SOAP** — 6-12 месяцев параллельно.
5. **Delete** SOAP.

---

## 15. Реальный кейс ИСНА

Из memory:
- **`knp-fno-outer-sync-esb-dead-route`** — SOAP интеграция с ЕСБ через `BT_OUTER_FNOSYNC_OUTER_TAXREP_SYNC` маршрут.
- **`knp-fno-reception-api-mgu-vs-knp`** — приёмка ФНО через MGU (REST) или КНП (SOAP).

Т.е. **гибрид**:
- Внешние гос-системы — SOAP-шина.
- Внутренние микросервисы КНП — REST + Feign.

---

## 16. Собесные вопросы

1. **REST vs SOAP — главное отличие?** — REST архитектурный стиль на HTTP + JSON; SOAP protocol на XML.
2. **Формат сообщений?** — REST: JSON (обычно); SOAP: XML always.
3. **Кто использует HTTP-методы?** — REST (GET/POST/PUT/DELETE); SOAP всегда POST.
4. **Кто использует status codes?** — REST (200/201/404/500); SOAP всегда 200 (fault в body).
5. **Кто быстрее / компактнее?** — REST (JSON меньше XML в 5-10×).
6. **Contract-first vs code-first?** — SOAP: contract-first (WSDL); REST: обычно code-first (или OpenAPI).
7. **Runtime validation?** — SOAP: XSD автоматически; REST: ручная через @Valid.
8. **Security?** — REST: HTTPS + JWT; SOAP: WS-Security (message-level end-to-end).
9. **Caching?** — REST: HTTP-кэш; SOAP: плохо.
10. **Когда SOAP?** — Требование партнёра, legacy, banking, government, strong contract.
11. **Когда REST?** — Микросервисы, публичные APIs, mobile, SPA, high-throughput.
12. **WS-Security vs OAuth?** — WS-Security на уровне сообщения; OAuth token в HTTP header.
13. **Что такое ESB?** — Enterprise Service Bus — центральная шина для SOAP.
14. **Modern alternatives?** — gRPC (fast, contract-first), GraphQL (flexible reads).
15. **Гибрид SOAP + REST?** — Facade REST перед SOAP-backend; постепенная миграция.

---

## Итог

- **REST** = архитектурный стиль на HTTP + JSON. **Default для новых проектов**.
- **SOAP** = protocol на XML + WS-*. Legacy / enterprise / government.
- **REST**: простой, быстрый, кэшируется, HTTP semantics.
- **SOAP**: verbose, contract-first, WS-Security, WS-Transaction.
- **В ИСНА**: гибрид (SOAP для ЕСБ, REST для микросервисов КНП).
- **Modern**: gRPC для internal, GraphQL для flexible reads.

---

## Финальный итог блоков 32-64

- **@Transactional** (32-35).
- **Spring Cloud** (36).
- **Nodes** (37).
- **Логирование** (38).
- **Kafka** (39-42).
- **Java memory + Prometheus + Elastic + nginx + инфра** (43-47).
- **Микросервисы** (48-52).
- **Тестирование** (53-56).
- **Config + Helm + Consul + Annotations + Controllers + REST + SOAP** (57-64).

**Итого 64 файла в `isna-theory\`**.

Дальше можно: **Redis** (in-memory кэш), **распределённые системы** (CAP теорема, eventual consistency), **DevOps / GitLab CI** углубленно, **криптография / ЭЦП** (Kalkan для ИСНА), **AWS/GCP** cloud базы, **алгоритмы + структуры данных** для собесов, **system design** interview. Скажи что.
