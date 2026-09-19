# 63. SOAP API: envelope, WSDL, XSD, WS-standards, JAX-WS

## Зачем понимать SOAP

Разработчик работающий preimущestvenno с REST APIs может думать — SOAP это legacy, не важно. Реальность enterprise Java окружения приносит SOAP в жизнь неожиданно. Government integration требует SOAP. Banking partners использует SOAP. Regulatory compliance systems SOAP-based. Даже modern applications interfacing с legacy systems часто encounter SOAP.

Разница между разработчиком «избегающим SOAP» и «понимающим SOAP» проявляется в capability engaging enterprise integrations. Первый видит SOAP endpoint plus не знает где начать. Второй знает SOAP envelope structure — Header plus Body. Знает WSDL как contract describing operations, types, endpoints. Знает XSD для strict type definitions. Знает WS-Security для message-level signing/encryption. Знает JAX-WS annotations для Spring integration. Может generate client code из WSDL через wsimport. Может configure SOAP endpoint в Spring через Apache CXF или Spring Web Services.

В этом файле разберём SOAP глубоко. Что такое SOAP fundamentally. SOAP envelope structure. Пример request/response cycles. SOAP Fault error format. WSDL — Web Services Description Language. XSD — XML Schema Definition. WS-* standards семейство. RPC vs Document style. Java стек (JAX-WS, Apache CXF, Spring Web Services). Kalkan ЭЦП в казахстанском context. Инструменты (SoapUI, wsimport). Почему SOAP ещё существует. Проблемы SOAP. Реальный ИСНА workflow с ЕСБ.

## Что такое SOAP

SOAP = Simple Object Access Protocol. Разработан Microsoft в 1998, стандартизирован W3C. Не совсем «simple» — на практике довольно тяжёлый.

Ключевые черты. Протокол (не архитектурный стиль как REST). XML для всего. Envelope (Header plus Body). Transport-agnostic — обычно HTTP но может быть SMTP, JMS. Contract-first — сначала WSDL, потом код. Enterprise features — WS-Security, WS-Transaction, WS-Addressing.

Historical context. SOAP emerged когда XML был dominant data format. Web services concept driven by enterprise integration needs. WS-* ecosystem grew из committee-driven standardization. Complex specifications создавали competitive advantage для enterprise vendors.

## SOAP Envelope

Всё сообщение — XML с фиксированной структурой:
```xml
<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">

  <soap:Header>
    <auth:Token xmlns:auth="...">abc123</auth:Token>
    <wsa:MessageID>uuid:12345</wsa:MessageID>
  </soap:Header>

  <soap:Body>
    <ord:CreateOrder xmlns:ord="http://example.com/orders">
      <ord:CustomerId>c1</ord:CustomerId>
      <ord:Amount>100.00</ord:Amount>
    </ord:CreateOrder>
  </soap:Body>

</soap:Envelope>
```

Всегда. Envelope — корневой элемент. Header (optional) — метаданные. Body — данные операции.

Namespace-heavy. soap: namespace для envelope structure. Custom namespaces для application-specific elements. WS-* namespaces для standard extensions.

Verbose но strict. Every element namespaced. Types validated against XSD. No implicit conversions.

## Пример SOAP request/response

Request (POST):
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

Response (200):
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

Обрати внимание. HTTP status всегда 200 (даже при ошибке — в body SOAP fault). SOAPAction header уточняет операцию.

SOAP ignores HTTP semantics. HTTP просто transport. Everything в SOAP layer. Cannot leverage HTTP caching, methods, status codes.

## SOAP Fault

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

Fault codes. soap:Sender — ошибка клиента (аналог 4xx). soap:Receiver — ошибка сервера (5xx). soap:VersionMismatch. soap:MustUnderstand — Header с mustUnderstand не понят.

HTTP status обычно всё равно 200 (или 500). Errors indicated в SOAP body not HTTP status. Different error handling model от REST.

Structured error details. Application-specific fault types в Detail element. Enables typed exception handling in client code.

## WSDL: Web Services Description Language

XML-описание SOAP-сервиса. Аналог OpenAPI для REST но mandatory not optional.

Пример (упрощённо):
```xml
<?xml version="1.0"?>
<definitions xmlns="http://schemas.xmlsoap.org/wsdl/"
             xmlns:tns="http://example.com/orders"
             targetNamespace="http://example.com/orders">

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

  <message name="CreateOrderRequest">
    <part name="body" element="tns:CreateOrder"/>
  </message>
  <message name="CreateOrderReply">
    <part name="body" element="tns:CreateOrderResponse"/>
  </message>

  <portType name="OrdersPortType">
    <operation name="CreateOrder">
      <input message="tns:CreateOrderRequest"/>
      <output message="tns:CreateOrderReply"/>
    </operation>
  </portType>

  <binding name="OrdersBinding" type="tns:OrdersPortType">
    <soap:binding style="document" transport="http://schemas.xmlsoap.org/soap/http"/>
    <operation name="CreateOrder">
      <soap:operation soapAction="http://example.com/orders/create"/>
      <input><soap:body use="literal"/></input>
      <output><soap:body use="literal"/></output>
    </operation>
  </binding>

  <service name="OrdersService">
    <port name="OrdersPort" binding="tns:OrdersBinding">
      <address location="http://example.com/orders"/>
    </port>
  </service>

</definitions>
```

Ключевые элементы. types — типы данных (через XSD). message — SOAP-сообщения. portType — операции (что можно). binding — как (протокол, encoding). service — где (URL).

Comprehensive service description. Type-safe. Enables tooling для automatic code generation.

## XSD: XML Schema Definition

Строгая типизация XML:
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

Типы. Простые. xsd:string, xsd:int, xsd:decimal, xsd:boolean, xsd:date, xsd:dateTime, xsd:base64Binary. Complex — с sequence, choice, all. Ограничения — minOccurs, maxOccurs, pattern (regex), enumeration.

Плюсы. Строгая валидация (сервер / клиент проверяют). Автогенерация Java-классов.

Минусы. Многословно. Сложно писать вручную. Verbose XML syntax.

Trade-off strong typing vs verbosity. Enterprise values type safety enough to accept verbosity.

## WS-* стандарты

WS- prefix — целое семейство стандартов расширения SOAP.

WS-Security. Безопасность на уровне сообщения (не транспорта). Encryption body. Signature. Username tokens. SAML tokens. Позволяет end-to-end безопасность через intermediary'ов (в отличие от TLS который прерывается на прокси).

Example header:
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

Message-level security. Sensitive parts can be encrypted while others plain. Fine-grained control.

WS-Addressing. Routing и correlation. MessageID, RelatesTo. To, ReplyTo, FaultTo. Для async messaging. Enables message flow patterns beyond simple request-response.

WS-ReliableMessaging. Гарантии доставки (at-least-once, exactly-once). Protocol-level reliability. Similar semantics к Kafka acks но SOAP-based.

WS-Transaction (WS-AT, WS-BA). Distributed transactions (2PC поверх SOAP). Rarely used practically. Enterprise systems сложно координировать transactions across boundaries.

WS-Policy. Описание требований (security, reliability) в WSDL. Declarative policy expression.

WS-Trust, WS-Federation. Federated identity (Single Sign-On). Enterprise SSO patterns predating OAuth.

UDDI. Реестр веб-сервисов. Deprecated, никто не использует. Failed vision of automatic service discovery.

BPEL. Business Process Execution Language — оркестрация SOAP-сервисов. Workflow definition language. Executed by BPEL engines.

Реальность. Спецификаций сотни. На практике реально используются. WS-Security, WS-Addressing, иногда WS-ReliableMessaging. Остальное — enterprise legacy.

## Style: RPC vs Document

Два стиля SOAP.

RPC. Body содержит имя операции plus параметры:
```xml
<soap:Body>
  <createOrder>
    <customerId>c1</customerId>
    <amount>100</amount>
  </createOrder>
</soap:Body>
```

Похоже на вызов метода. Method name explicit в body.

Document. Body содержит XML-документ (по XSD schema):
```xml
<soap:Body>
  <Order xmlns="http://example.com/orders">
    <customerId>c1</customerId>
    <amount>100</amount>
  </Order>
</soap:Body>
```

Document/literal — модерный стандарт. Более гибкий (schema-driven). Recommended style.

Difference subtle но important. RPC binds operation name in message. Document allows schema evolution более гибко.

## Java стек

JAX-WS стандарт Java EE. Стандарт для SOAP в Java.

Producer:
```java
@WebService
public class OrderService {

    @WebMethod
    public Long createOrder(
        @WebParam(name = "customerId") String customerId,
        @WebParam(name = "amount") BigDecimal amount) {
        return orderId;
    }
}
```

Publish:
```java
Endpoint.publish("http://localhost:8080/orders", new OrderService());
```

Simple embedded SOAP endpoint. Development convenience.

Consumer генерируется из WSDL через wsimport:
```bash
wsimport -keep -d target/generated -p com.example.client http://server/orders?wsdl
```

Сгенерируется Java-код (клиент plus все DTO по WSDL/XSD).

Использование:
```java
OrdersService service = new OrdersService();
OrdersPortType port = service.getOrdersPort();
Long orderId = port.createOrder("c1", BigDecimal.valueOf(100));
```

Type-safe Java client generated from WSDL. Compilation errors для incompatible API changes.

Apache CXF. Популярная реализация JAX-WS plus расширения (WS-Security и т.д.). Feature-rich enterprise Java SOAP stack. Alternative to reference JAX-WS.

Spring Web Services. Spring-based framework для SOAP:
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

Более удобно чем чистый JAX-WS. Spring integration plus contract-first approach.

## Kalkan ЭЦП в Казахстане

В ИСНА / КНП SOAP используется активно.

Из memory. SOAP-шина — центральная integration bus. ЕСБ (единая система баланса) — SOAP-based. save-fno-<code> — SOAP endpoint для приёмки ФНО. BT_OUTER_FNOSYNC_OUTER_TAXREP_SYNC — маршрут в ЕСБ. Kalkan — библиотека ЭЦП, тоже часто через SOAP.

Реальный кейс knp-fno-outer-sync-esb-dead-route — SOAP «Requested service is not found» (маршрут не поднят).

Причина. Интеграция с гос. системами — исторически SOAP plus WSDL plus WS-Security. Government regulations и existing infrastructure sustained SOAP usage.

Kalkan — казахстанская crypto library для ЭЦП (электронная цифровая подпись). Integrated into SOAP через WS-Security signatures. Government-approved cryptographic operations.

## Инструменты

SoapUI. Классика для тестирования SOAP APIs. GUI plus XML editor. Loads WSDL, generates sample requests, executes calls, validates responses.

Postman. Тоже поддерживает SOAP (просто отправить XML). Not SOAP-specialized но works для basic scenarios.

curl:
```bash
curl -X POST \
  -H "Content-Type: text/xml" \
  -H "SOAPAction: create" \
  -d @request.xml \
  http://example.com/orders
```

Command-line testing. Scripting-friendly.

wsimport / cxf-codegen. Java tools для генерации клиента из WSDL. Compile-time code generation.

## Почему SOAP ещё существует

Где-то используется. Banking — legacy интеграции. Government — legacy (Казахстан ИСНА пример). Enterprise B2B (EDI, SAP integrations). Telecom (OSS/BSS). Financial exchanges (FIX частично).

Причины. Legacy — работает годами, переписывать дорого. Contract-first — WSDL/XSD даёт строгую типизацию. WS-Security — end-to-end signing/encryption. Требование партнёров.

Новые проекты — REST plus JSON. SOAP только когда обязательно. Compatibility with existing infrastructure главная driver.

## Проблемы SOAP

Verbose. XML вдесятеро больше JSON. Bandwidth waste. Network costs multiplied.

Сложность. WSDL plus XSD plus WS-Security — недели на настройку. Learning curve steep.

Отладка кошмар. Сложные XML namespaces, невнятные faults. XML tooling required.

Contract-first. Изменение WSDL — генерация нового клиента — deploy — тестирование. Медленный цикл. Slower iteration чем REST.

Плохо кэшируется. Всегда POST, всегда XML — HTTP-кэш не работает. Read-heavy scenarios inefficient.

Нет из browser'а. JavaScript плохо работает с SOAP (нужны parsing XML libraries). Frontend integration awkward.

Tooling. REST — любой язык, любой инструмент. SOAP — enterprise Java/.NET/mid-tier. Ecosystem более narrow.

## Пример полного SOAP-цикла в ИСНА

Приёмка ФНО через ЕСБ demonstrates practical usage.

АРМ (АРМ инспектора) вызывает save-fno-<code> SOAP:
```xml
<soap:Envelope>
  <soap:Header>
    <wsse:Security>
      <ds:Signature>...</ds:Signature>   ← ЭЦП через Kalkan
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

ЕСБ маршрутизирует на isna-fno.

isna-fno. Валидирует ЭЦП (Kalkan). Парсит XML — Java-объект. Сохраняет в БД. Возвращает SOAP-response.

АРМ показывает результат.

Complete flow demonstrates. ЭЦП integration через WS-Security. Routing через integration bus. Enterprise SOAP patterns в government context.

## Итоги

SOAP protocol на XML для web services. Contract-first через WSDL. Transport-agnostic (обычно HTTP).

SOAP Envelope structure. Header optional (metadata, security, addressing). Body required (operation data). Namespace-heavy.

SOAP Fault для errors в body. HTTP status обычно 200 regardless of application-level outcomes. Different from REST error model.

WSDL описание сервиса. Types (XSD). Messages. PortType (operations). Binding (protocol details). Service (endpoint).

XSD schema definitions. Строгая типизация. Complex types. Ограничения. Автогенерация Java classes через tooling.

WS-* стандарты семейство. WS-Security для message-level protection. WS-Addressing для routing/correlation. WS-Transaction для distributed tx (редко). WS-Federation для SSO. Многие others.

RPC vs Document style. Document/literal — modern standard. Schema-driven flexibility.

Java стек. JAX-WS стандарт. Apache CXF popular implementation. Spring Web Services для Spring integration.

Kalkan ЭЦП в казахстанском context. Government-approved cryptographic library. Integrated через WS-Security signatures.

Инструменты. SoapUI standard для testing. wsimport для client generation.

SOAP ещё существует в banking, government, enterprise B2B, telecom. Legacy plus compliance requirements sustained usage.

Проблемы. Verbose XML. Сложность WSDL/XSD. Slow debugging. Slow iteration cycle. Poor caching. Weak browser support. Narrow tooling ecosystem.

В ИСНА. SOAP-шина ЕСБ plus Kalkan ЭЦП для government integration. Legacy sustained through regulations.

Дальше — REST vs SOAP развёрнутое comparison. Когда что выбирать. Modern alternatives (gRPC, GraphQL).
