# 63. SOAP API теория

Что такое SOAP, WSDL, XSD, WS-* стандарты, зачем ещё существует.

---

## 1. Что такое SOAP

**SOAP** = **S**imple **O**bject **A**ccess **P**rotocol.

Разработан Microsoft в 1998, стандартизирован W3C.

Не совсем "simple" — на практике довольно тяжёлый.

Ключевые черты:
- **Протокол** (не архитектурный стиль как REST).
- **XML** для всего.
- **Envelope** (Header + Body).
- **Transport-agnostic** — обычно HTTP, но может быть SMTP, JMS.
- **Contract-first** — сначала WSDL, потом код.
- **Enterprise features** — WS-Security, WS-Transaction, WS-Addressing.

---

## 2. SOAP Envelope

Всё сообщение — XML с фиксированной структурой:

```xml
<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">

  <soap:Header>
    <!-- Метаданные: security tokens, correlation, routing -->
    <auth:Token xmlns:auth="...">abc123</auth:Token>
    <wsa:MessageID>uuid:12345</wsa:MessageID>
  </soap:Header>

  <soap:Body>
    <!-- Полезная нагрузка -->
    <ord:CreateOrder xmlns:ord="http://example.com/orders">
      <ord:CustomerId>c1</ord:CustomerId>
      <ord:Amount>100.00</ord:Amount>
    </ord:CreateOrder>
  </soap:Body>

</soap:Envelope>
```

Всегда:
- Envelope — корневой элемент.
- Header (optional) — метаданные.
- Body — данные операции.

---

## 3. Пример SOAP-запрос/ответ

**Request** (POST):
```
POST /orders HTTP/1.1
Host: example.com
Content-Type: text/xml; charset=utf-8
SOAPAction: "http://example.com/orders/create"

<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <ord:CreateOrder xmlns:ord="http://example.com/orders">
      <ord:CustomerId>c1</ord:CustomerId>
      <ord:Amount>100.00</ord:Amount>
    </ord:CreateOrder>
  </soap:Body>
</soap:Envelope>
```

**Response** (200):
```xml
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

Обрати внимание:
- HTTP status всегда **200** (даже при ошибке — в body SOAP fault).
- `SOAPAction` header — уточняет операцию.

---

## 4. SOAP Fault

Ошибки — тоже в SOAP body:

```xml
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <soap:Fault>
      <soap:Code>
        <soap:Value>soap:Sender</soap:Value>
      </soap:Code>
      <soap:Reason>
        <soap:Text>Customer not found</soap:Text>
      </soap:Reason>
      <soap:Detail>
        <ord:CustomerNotFoundFault>
          <ord:CustomerId>c999</ord:CustomerId>
        </ord:CustomerNotFoundFault>
      </soap:Detail>
    </soap:Fault>
  </soap:Body>
</soap:Envelope>
```

Fault codes:
- **soap:Sender** — ошибка клиента (аналог 4xx).
- **soap:Receiver** — ошибка сервера (5xx).
- **soap:VersionMismatch**.
- **soap:MustUnderstand** — Header с mustUnderstand не понят.

HTTP status обычно всё равно 200 (или 500).

---

## 5. WSDL — Web Services Description Language

XML-описание SOAP-сервиса. Аналог OpenAPI для REST.

Пример (упрощённо):
```xml
<?xml version="1.0"?>
<definitions xmlns="http://schemas.xmlsoap.org/wsdl/"
             xmlns:tns="http://example.com/orders"
             targetNamespace="http://example.com/orders">

  <!-- Типы данных (XSD) -->
  <types>
    <schema xmlns="http://www.w3.org/2001/XMLSchema">
      <element name="CreateOrder">
        <complexType>
          <sequence>
            <element name="customerId" type="xsd:string"/>
            <element name="amount" type="xsd:decimal"/>
          </sequence>
        </complexType>
      </element>
      <element name="CreateOrderResponse">
        <complexType>
          <sequence>
            <element name="orderId" type="xsd:long"/>
            <element name="status" type="xsd:string"/>
          </sequence>
        </complexType>
      </element>
    </schema>
  </types>

  <!-- Сообщения -->
  <message name="CreateOrderRequest">
    <part name="body" element="tns:CreateOrder"/>
  </message>
  <message name="CreateOrderReply">
    <part name="body" element="tns:CreateOrderResponse"/>
  </message>

  <!-- Порт (интерфейс) -->
  <portType name="OrdersPortType">
    <operation name="CreateOrder">
      <input message="tns:CreateOrderRequest"/>
      <output message="tns:CreateOrderReply"/>
    </operation>
  </portType>

  <!-- Binding (протокол) -->
  <binding name="OrdersBinding" type="tns:OrdersPortType">
    <soap:binding style="document" transport="http://schemas.xmlsoap.org/soap/http"/>
    <operation name="CreateOrder">
      <soap:operation soapAction="http://example.com/orders/create"/>
      <input><soap:body use="literal"/></input>
      <output><soap:body use="literal"/></output>
    </operation>
  </binding>

  <!-- Service (адрес) -->
  <service name="OrdersService">
    <port name="OrdersPort" binding="tns:OrdersBinding">
      <address location="http://example.com/orders"/>
    </port>
  </service>

</definitions>
```

Ключевые элементы:
- **types** — типы данных (через XSD).
- **message** — SOAP-сообщения.
- **portType** — операции (что можно).
- **binding** — как (протокол, encoding).
- **service** — где (URL).

---

## 6. XSD — XML Schema Definition

Строгая типизация XML.

```xml
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema">
  <xsd:element name="Order">
    <xsd:complexType>
      <xsd:sequence>
        <xsd:element name="id" type="xsd:long"/>
        <xsd:element name="customerId" type="xsd:string" minOccurs="1"/>
        <xsd:element name="amount" type="xsd:decimal" minOccurs="0"/>
        <xsd:element name="items" type="tns:OrderItems"/>
      </xsd:sequence>
      <xsd:attribute name="version" type="xsd:int" use="required"/>
    </xsd:complexType>
  </xsd:element>
</xsd:schema>
```

Типы:
- Простые: `xsd:string`, `xsd:int`, `xsd:decimal`, `xsd:boolean`, `xsd:date`, `xsd:dateTime`, `xsd:base64Binary`.
- Complex: с sequence, choice, all.
- Ограничения: `minOccurs`, `maxOccurs`, `pattern` (regex), `enumeration`.

Плюсы:
- Строгая валидация (сервер / клиент проверяют).
- Автогенерация Java-классов.

Минусы:
- Многословно.
- Сложно писать вручную.

---

## 7. WS-* стандарты

**WS-** прибавка — целое семейство стандартов расширения SOAP.

### 7.1 WS-Security

Безопасность на уровне сообщения (не транспорта):
- Encryption body.
- Signature.
- Username tokens.
- SAML tokens.

Позволяет **end-to-end** безопасность через intermediary'ов (в отличие от TLS который прерывается на прокси).

```xml
<soap:Header>
  <wsse:Security>
    <wsse:UsernameToken>
      <wsse:Username>alice</wsse:Username>
      <wsse:Password>...</wsse:Password>
    </wsse:UsernameToken>
    <ds:Signature>...</ds:Signature>
  </wsse:Security>
</soap:Header>
```

### 7.2 WS-Addressing

Routing и correlation:
- `MessageID`, `RelatesTo`.
- `To`, `ReplyTo`, `FaultTo`.

Для async messaging.

### 7.3 WS-ReliableMessaging

Гарантии доставки (at-least-once, exactly-once).

### 7.4 WS-Transaction (WS-AT, WS-BA)

Distributed transactions (2PC поверх SOAP).

### 7.5 WS-Policy

Описание требований (security, reliability) в WSDL.

### 7.6 WS-Trust, WS-Federation

Federated identity (Single Sign-On).

### 7.7 UDDI

Реестр веб-сервисов. Deprecated, никто не использует.

### 7.8 BPEL

Business Process Execution Language — оркестрация SOAP-сервисов.

### 7.9 Реальность

Спецификаций **сотни**. На практике реально используются: WS-Security, WS-Addressing, иногда WS-ReliableMessaging.

Остальное — enterprise legacy.

---

## 8. Style: RPC vs Document

Два стиля SOAP.

### 8.1 RPC

Body содержит имя операции + параметры:
```xml
<soap:Body>
  <createOrder>
    <customerId>c1</customerId>
    <amount>100</amount>
  </createOrder>
</soap:Body>
```

Похоже на вызов метода.

### 8.2 Document

Body содержит XML-документ (по XSD schema):
```xml
<soap:Body>
  <Order xmlns="http://example.com/orders">
    <customerId>c1</customerId>
    <amount>100</amount>
  </Order>
</soap:Body>
```

**Document/literal** — модерный стандарт. Более гибкий (schema-driven).

---

## 9. Java стек

### 9.1 JAX-WS (стандарт Java EE)

Стандарт для SOAP в Java.

**Producer**:
```java
@WebService
public class OrderService {

    @WebMethod
    public Long createOrder(
        @WebParam(name = "customerId") String customerId,
        @WebParam(name = "amount") BigDecimal amount) {
        // ...
        return orderId;
    }
}
```

Publish:
```java
Endpoint.publish("http://localhost:8080/orders", new OrderService());
```

**Consumer** — генерируется из WSDL через **`wsimport`**:
```bash
wsimport -keep -d target/generated -p com.example.client http://server/orders?wsdl
```

Сгенерируется Java-код (клиент + все DTO по WSDL/XSD).

Использование:
```java
OrdersService service = new OrdersService();
OrdersPortType port = service.getOrdersPort();
Long orderId = port.createOrder("c1", BigDecimal.valueOf(100));
```

### 9.2 Apache CXF

Популярная реализация JAX-WS + расширения (WS-Security и т.д.).

### 9.3 Spring Web Services

Spring-based framework для SOAP:
```java
@Endpoint
public class OrderEndpoint {

    @PayloadRoot(namespace = "http://example.com/orders", localPart = "CreateOrder")
    @ResponsePayload
    public CreateOrderResponse create(@RequestPayload CreateOrder request) {
        // ...
    }
}
```

Более удобно чем чистый JAX-WS.

---

## 10. Klass "Kalkan" ЭЦП в Казахстане

В ИСНА / КНП SOAP используется активно.

Из memory:
- **SOAP-шина** — центральная integration bus.
- **ЕСБ** (единая система баланса) — SOAP-based.
- `save-fno-<code>` — SOAP endpoint для приёмки ФНО.
- `BT_OUTER_FNOSYNC_OUTER_TAXREP_SYNC` — маршрут в ЕСБ.
- **Kalkan** — библиотека ЭЦП, тоже часто через SOAP.

Реальный кейс `knp-fno-outer-sync-esb-dead-route` — SOAP "Requested service is not found" (маршрут не поднят).

Причина: интеграция с гос. системами — исторически SOAP + WSDL + WS-Security.

---

## 11. Инструменты для SOAP

### 11.1 SoapUI

Классика для тестирования SOAP APIs. GUI + XML editor.

### 11.2 Postman

Тоже поддерживает SOAP (просто отправить XML).

### 11.3 curl

```bash
curl -X POST \
  -H "Content-Type: text/xml" \
  -H "SOAPAction: create" \
  -d @request.xml \
  http://example.com/orders
```

### 11.4 wsimport / cxf-codegen

Java tools для генерации клиента из WSDL.

---

## 12. Почему SOAP ещё существует

Где-то используется:
- **Banking** — legacy интеграции.
- **Government** — legacy (Казахстан ИСНА пример).
- **Enterprise B2B** (EDI, SAP integrations).
- **Telecom** (OSS/BSS).
- **Financial exchanges** (FIX частично).

Причины:
- **Legacy** — работает годами, переписывать дорого.
- **Contract-first** — WSDL/XSD даёт строгую типизацию.
- **WS-Security** — end-to-end signing/encryption.
- **Требование** партнёров.

Новые проекты — **REST + JSON**. SOAP только когда обязательно.

---

## 13. Проблемы SOAP

### 13.1 Verbose

XML вдесятеро больше JSON. Bandwidth waste.

### 13.2 Сложность

WSDL + XSD + WS-Security — недели на настройку.

Отладка кошмар — сложные XML namespaces, невнятные faults.

### 13.3 Contract-first

Изменение WSDL → генерация нового клиента → deploy → тестирование. Медленный цикл.

### 13.4 Плохо кэшируется

Всегда POST, всегда XML — HTTP-кэш не работает.

### 13.5 Нет из browser'а

JavaScript плохо работает с SOAP (нужны parsing XML libraries).

### 13.6 Tooling

REST — любой язык, любой инструмент. SOAP — enterprise Java/.NET/mid-tier.

---

## 14. Пример полного SOAP-цикла в ИСНА

Приёмка ФНО через ЕСБ:

1. **АРМ** (АРМ инспектора) вызывает `save-fno-<code>` SOAP:
   ```xml
   <soap:Envelope>
     <soap:Header>
       <wsse:Security>
         <ds:Signature>...</ds:Signature>   ← ЭЦП
       </wsse:Security>
     </soap:Header>
     <soap:Body>
       <SaveFno>
         <RegNum>...</RegNum>
         <Fno>...</Fno>
       </SaveFno>
     </soap:Body>
   </soap:Envelope>
   ```

2. **ЕСБ** маршрутизирует на `isna-fno`.

3. `isna-fno`:
   - Валидирует ЭЦП (Kalkan).
   - Парсит XML → Java-объект.
   - Сохраняет в БД.
   - Возвращает SOAP-response.

4. АРМ показывает результат.

---

## 15. Собесные вопросы

1. **Что такое SOAP?** — Protocol на XML для web services; envelope с Header/Body.
2. **Разница REST и SOAP?** — REST архитектурный стиль на HTTP + JSON; SOAP protocol на XML.
3. **Что такое WSDL?** — XML-описание сервиса (types, operations, endpoint).
4. **Что такое XSD?** — XML Schema; типизация XML документов.
5. **SOAP Envelope структура?** — Envelope → Header (optional) + Body.
6. **SOAP Fault — что?** — Стандартный формат ошибок в SOAP Body.
7. **Какой HTTP status при SOAP fault?** — Обычно 200 (fault в body) или 500.
8. **WS-Security — что даёт?** — End-to-end безопасность на уровне сообщения (encryption, signature).
9. **RPC vs Document style?** — RPC: имя операции + params; Document: XML документ по schema.
10. **JAX-WS — что?** — Java стандарт для SOAP; `@WebService`, `wsimport`.
11. **Как сгенерировать SOAP-клиент из WSDL?** — `wsimport -keep -d target -p pkg URL_TO_WSDL`.
12. **SOAPAction header — зачем?** — Уточняет какая операция вызывается.
13. **Почему SOAP verbose vs REST?** — XML boilerplate, envelope, namespaces, WS-* headers.
14. **Где SOAP ещё используется?** — Banking, government, enterprise B2B, legacy.
15. **Что такое ESB?** — Enterprise Service Bus; централизованная шина для SOAP-сервисов.

---

## Итог

- **SOAP** = protocol на XML; envelope Header + Body.
- **WSDL** описывает сервис; **XSD** — типы.
- **JAX-WS** — Java стандарт.
- **WS-*** — enterprise расширения (WS-Security главный).
- **Verbose + сложно**, но **строгий contract**.
- Используется в legacy / enterprise / government.
- В **ИСНА**: SOAP-шина ЕСБ + Kalkan ЭЦП.

Следующий — `64-rest-vs-soap.md`.
