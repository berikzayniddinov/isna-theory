# 70. XSD и WSDL детально

XML Schema Definition + Web Services Description Language. Подробное объяснение.

---

## 1. Зачем это нужно

Обмен данными между системами через XML — нужен **строгий контракт**:
- Какие поля обязательны?
- Какие типы?
- Максимальная длина?
- Формат даты?

**XSD** описывает **структуру XML данных**.
**WSDL** описывает **web service** (какие операции, где, как вызывать).

WSDL использует XSD для типов.

В ИСНА (интеграция с ЕСБ, гос-системами, налоговыми формами) — активно.

---

## 2. XSD — XML Schema Definition

### 2.1 Что это

**XSD** — стандартный W3C-язык описания структуры XML документов.

Файл `.xsd`, сам является XML.

Аналог: `JSON Schema` для JSON, `.proto` для Protobuf, класс в Java.

### 2.2 Простой пример

**XML данные**:
```xml
<order>
  <id>42</id>
  <customerId>C001</customerId>
  <amount>100.50</amount>
</order>
```

**XSD**:
```xml
<?xml version="1.0"?>
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema">

  <xsd:element name="order">
    <xsd:complexType>
      <xsd:sequence>
        <xsd:element name="id" type="xsd:long"/>
        <xsd:element name="customerId" type="xsd:string"/>
        <xsd:element name="amount" type="xsd:decimal"/>
      </xsd:sequence>
    </xsd:complexType>
  </xsd:element>

</xsd:schema>
```

Читается: «order = complex type; sequence из id/customerId/amount с типами».

### 2.3 Namespace

`xmlns:xsd="http://www.w3.org/2001/XMLSchema"` — стандартный XSD namespace.

Свой namespace:
```xml
<xsd:schema
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    xmlns:tns="http://example.com/orders"
    targetNamespace="http://example.com/orders">
```

Отсюда XML должен указать namespace:
```xml
<order xmlns="http://example.com/orders">
  ...
</order>
```

Namespace — URI (не URL — не обязательно должен резолвиться). Уникальный идентификатор схемы.

---

## 3. Simple types

**Simple type** — не имеет child элементов и атрибутов. Только значение.

### 3.1 Встроенные

Стандартные:
- **`xsd:string`** — текст.
- **`xsd:int`** — 32-bit signed integer.
- **`xsd:long`** — 64-bit signed.
- **`xsd:short`** — 16-bit.
- **`xsd:byte`** — 8-bit.
- **`xsd:decimal`** — точное десятичное (для денег!).
- **`xsd:float`**, **`xsd:double`** — floating-point.
- **`xsd:boolean`** — `true` / `false`.
- **`xsd:date`** — `2026-09-07`.
- **`xsd:time`** — `10:30:00`.
- **`xsd:dateTime`** — `2026-09-07T10:30:00`.
- **`xsd:duration`** — `PT1H30M`.
- **`xsd:base64Binary`** — binary в base64.
- **`xsd:hexBinary`** — binary в hex.
- **`xsd:anyURI`** — URI.
- **`xsd:QName`** — qualified name с namespace.

### 3.2 Restrictions (ограничения)

Custom simple type = встроенный + restrictions:

```xml
<!-- ограниченный string -->
<xsd:simpleType name="RegNumber">
  <xsd:restriction base="xsd:string">
    <xsd:minLength value="12"/>
    <xsd:maxLength value="12"/>
    <xsd:pattern value="[0-9]{12}"/>
  </xsd:restriction>
</xsd:simpleType>

<!-- ограниченный int -->
<xsd:simpleType name="Percent">
  <xsd:restriction base="xsd:int">
    <xsd:minInclusive value="0"/>
    <xsd:maxInclusive value="100"/>
  </xsd:restriction>
</xsd:simpleType>

<!-- enum -->
<xsd:simpleType name="OrderStatus">
  <xsd:restriction base="xsd:string">
    <xsd:enumeration value="NEW"/>
    <xsd:enumeration value="PROCESSING"/>
    <xsd:enumeration value="SHIPPED"/>
    <xsd:enumeration value="CANCELLED"/>
  </xsd:restriction>
</xsd:simpleType>

<!-- decimal с precision -->
<xsd:simpleType name="Money">
  <xsd:restriction base="xsd:decimal">
    <xsd:totalDigits value="12"/>
    <xsd:fractionDigits value="2"/>
    <xsd:minInclusive value="0"/>
  </xsd:restriction>
</xsd:simpleType>
```

Restrictions:
- **minLength / maxLength** — длина строки.
- **length** — фиксированная длина.
- **pattern** — regex.
- **enumeration** — enum.
- **minInclusive / maxInclusive / minExclusive / maxExclusive** — диапазон.
- **totalDigits / fractionDigits** — для decimal.
- **whiteSpace** — preserve / replace / collapse.

Валидация: XML парсер проверяет все ограничения.

### 3.3 Использование

```xml
<xsd:element name="regNum" type="tns:RegNumber"/>
<xsd:element name="status" type="tns:OrderStatus"/>
<xsd:element name="amount" type="tns:Money"/>
```

---

## 4. Complex types

**Complex type** — с child элементами и/или атрибутами.

### 4.1 Sequence

Порядок фиксирован:
```xml
<xsd:complexType name="Order">
  <xsd:sequence>
    <xsd:element name="id" type="xsd:long"/>
    <xsd:element name="customerId" type="xsd:string"/>
    <xsd:element name="amount" type="xsd:decimal"/>
  </xsd:sequence>
</xsd:complexType>
```

XML должен идти в том же порядке.

### 4.2 Choice

Один из вариантов:
```xml
<xsd:complexType name="Payment">
  <xsd:choice>
    <xsd:element name="cardNumber" type="xsd:string"/>
    <xsd:element name="bankAccount" type="xsd:string"/>
    <xsd:element name="cryptoAddress" type="xsd:string"/>
  </xsd:choice>
</xsd:complexType>
```

Только один из cardNumber/bankAccount/cryptoAddress в XML.

### 4.3 All

Все элементы, порядок неважен:
```xml
<xsd:complexType name="Address">
  <xsd:all>
    <xsd:element name="street" type="xsd:string"/>
    <xsd:element name="city" type="xsd:string"/>
    <xsd:element name="zip" type="xsd:string"/>
  </xsd:all>
</xsd:complexType>
```

Все обязательны, но в любом порядке.

### 4.4 minOccurs / maxOccurs

```xml
<xsd:element name="middleName" type="xsd:string" minOccurs="0"/>
<!-- необязательный -->

<xsd:element name="phone" type="xsd:string" minOccurs="0" maxOccurs="unbounded"/>
<!-- 0 или больше -->

<xsd:element name="item" type="tns:OrderItem" minOccurs="1" maxOccurs="100"/>
<!-- от 1 до 100 -->
```

Default:
- `minOccurs=1` (обязательный).
- `maxOccurs=1` (не массив).

### 4.5 Массивы

`maxOccurs > 1` = массив.

XML:
```xml
<phones>
  <phone>+7-555-0100</phone>
  <phone>+7-555-0200</phone>
</phones>
```

XSD:
```xml
<xsd:element name="phones">
  <xsd:complexType>
    <xsd:sequence>
      <xsd:element name="phone" type="xsd:string" minOccurs="0" maxOccurs="unbounded"/>
    </xsd:sequence>
  </xsd:complexType>
</xsd:element>
```

### 4.6 Attributes

Атрибуты элемента:
```xml
<order id="42" version="1">
  <customerId>C001</customerId>
</order>
```

XSD:
```xml
<xsd:complexType name="Order">
  <xsd:sequence>
    <xsd:element name="customerId" type="xsd:string"/>
  </xsd:sequence>
  <xsd:attribute name="id" type="xsd:long" use="required"/>
  <xsd:attribute name="version" type="xsd:int" use="optional" default="1"/>
</xsd:complexType>
```

`use`: **required** / **optional** / **prohibited**.

### 4.7 Nested types

```xml
<xsd:complexType name="Order">
  <xsd:sequence>
    <xsd:element name="id" type="xsd:long"/>
    <xsd:element name="customer">
      <xsd:complexType>
        <xsd:sequence>
          <xsd:element name="id" type="xsd:string"/>
          <xsd:element name="name" type="xsd:string"/>
        </xsd:sequence>
      </xsd:complexType>
    </xsd:element>
    <xsd:element name="items">
      <xsd:complexType>
        <xsd:sequence>
          <xsd:element name="item" type="tns:OrderItem" maxOccurs="unbounded"/>
        </xsd:sequence>
      </xsd:complexType>
    </xsd:element>
  </xsd:sequence>
</xsd:complexType>
```

Можно inline или named types (для переиспользования).

---

## 5. Наследование (extension / restriction)

### 5.1 Extension (добавить)

```xml
<xsd:complexType name="Person">
  <xsd:sequence>
    <xsd:element name="name" type="xsd:string"/>
  </xsd:sequence>
</xsd:complexType>

<xsd:complexType name="Employee">
  <xsd:complexContent>
    <xsd:extension base="tns:Person">
      <xsd:sequence>
        <xsd:element name="employeeId" type="xsd:string"/>
        <xsd:element name="department" type="xsd:string"/>
      </xsd:sequence>
    </xsd:extension>
  </xsd:complexContent>
</xsd:complexType>
```

`Employee` = `Person` + новые поля.

### 5.2 Restriction (сузить)

```xml
<xsd:complexType name="LimitedPerson">
  <xsd:complexContent>
    <xsd:restriction base="tns:Person">
      <xsd:sequence>
        <xsd:element name="name" type="xsd:string" fixed="John"/>
      </xsd:sequence>
    </xsd:restriction>
  </xsd:complexContent>
</xsd:complexType>
```

Реже используется.

### 5.3 Abstract types

```xml
<xsd:complexType name="Vehicle" abstract="true">
  <xsd:sequence>
    <xsd:element name="brand" type="xsd:string"/>
  </xsd:sequence>
</xsd:complexType>

<xsd:complexType name="Car">
  <xsd:complexContent>
    <xsd:extension base="tns:Vehicle">
      <xsd:sequence>
        <xsd:element name="doors" type="xsd:int"/>
      </xsd:sequence>
    </xsd:extension>
  </xsd:complexContent>
</xsd:complexType>
```

`Vehicle` нельзя использовать напрямую — только через subtypes.

### 5.4 substitutionGroup + xsi:type

```xml
<xsd:element name="vehicle" type="tns:Vehicle"/>

<vehicle xsi:type="Car" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <brand>Toyota</brand>
  <doors>4</doors>
</vehicle>
```

Полиморфизм в XML.

---

## 6. Import / include

Схемы в разных файлах.

### 6.1 include

Тот же namespace, разбитие на файлы:
```xml
<xsd:include schemaLocation="common-types.xsd"/>
```

### 6.2 import

Другой namespace:
```xml
<xsd:import namespace="http://example.com/common"
            schemaLocation="common.xsd"/>
```

Часто у гос-схем — много импортов (базовые типы, справочники).

---

## 7. Валидация XML против XSD

### 7.1 Java (Xerces)

```java
SchemaFactory factory = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
Schema schema = factory.newSchema(new File("order.xsd"));
Validator validator = schema.newValidator();

try {
    validator.validate(new StreamSource(new File("order.xml")));
    System.out.println("Valid");
} catch (SAXException e) {
    System.err.println("Invalid: " + e.getMessage());
}
```

### 7.2 CLI (xmllint)

```bash
xmllint --schema order.xsd order.xml --noout
```

### 7.3 В IDE

IntelliJ / Eclipse — авто-валидация XML при открытии если XSD доступен.

---

## 8. Генерация Java из XSD (JAXB)

**JAXB (Jakarta XML Binding)** — генерирует Java-классы из XSD.

### 8.1 xjc (JAXB compiler)

```bash
xjc -d src/main/java -p com.example.orders order.xsd
```

Сгенерирует классы `Order.java`, `Customer.java` и т.д.

С аннотациями:
```java
@XmlAccessorType(XmlAccessType.FIELD)
@XmlType(name = "Order", propOrder = {"id", "customerId", "amount"})
public class Order {

    @XmlElement(required = true)
    protected long id;
    @XmlElement(required = true)
    protected String customerId;
    @XmlElement(required = true)
    protected BigDecimal amount;

    // getters/setters
}
```

### 8.2 Gradle

```gradle
plugins {
    id 'com.github.bjornvester.xjc' version '1.8.2'
}

xjc {
    schemas.from fileTree('src/main/resources/xsd')
    generatedPackage = 'com.example.orders'
}
```

### 8.3 Marshalling / unmarshalling

```java
JAXBContext ctx = JAXBContext.newInstance(Order.class);

// Java → XML
Marshaller m = ctx.createMarshaller();
m.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);
m.marshal(order, new File("order.xml"));

// XML → Java
Unmarshaller u = ctx.createUnmarshaller();
Order o = (Order) u.unmarshal(new File("order.xml"));
```

---

## 9. WSDL — Web Services Description Language

**WSDL** описывает **SOAP web service**:
- Какие операции.
- Какие сообщения (input/output).
- Какие типы данных (через XSD).
- Где endpoint.
- Как вызывать (SOAP через HTTP).

Аналог **OpenAPI** для REST.

### 9.1 Структура WSDL 1.1

Пять основных секций:

1. **`<types>`** — типы данных (XSD).
2. **`<message>`** — сообщения (input/output).
3. **`<portType>`** — операции (интерфейс).
4. **`<binding>`** — как (протокол, encoding).
5. **`<service>`** — где (URL endpoint).

### 9.2 Полный пример

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions
    xmlns="http://schemas.xmlsoap.org/wsdl/"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"
    xmlns:tns="http://example.com/orders"
    targetNamespace="http://example.com/orders">

  <!-- ═══ 1. Types (XSD inline или import) ═══ -->
  <types>
    <xsd:schema targetNamespace="http://example.com/orders">

      <xsd:element name="CreateOrderRequest">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="customerId" type="xsd:string"/>
            <xsd:element name="amount" type="xsd:decimal"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>

      <xsd:element name="CreateOrderResponse">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="orderId" type="xsd:long"/>
            <xsd:element name="status" type="xsd:string"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>

      <xsd:element name="OrderFault">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="code" type="xsd:string"/>
            <xsd:element name="message" type="xsd:string"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>

    </xsd:schema>
  </types>

  <!-- ═══ 2. Messages ═══ -->
  <message name="CreateOrderRequestMessage">
    <part name="parameters" element="tns:CreateOrderRequest"/>
  </message>

  <message name="CreateOrderResponseMessage">
    <part name="parameters" element="tns:CreateOrderResponse"/>
  </message>

  <message name="OrderFaultMessage">
    <part name="fault" element="tns:OrderFault"/>
  </message>

  <!-- ═══ 3. Port Type (интерфейс) ═══ -->
  <portType name="OrdersPortType">
    <operation name="createOrder">
      <input message="tns:CreateOrderRequestMessage"/>
      <output message="tns:CreateOrderResponseMessage"/>
      <fault name="orderFault" message="tns:OrderFaultMessage"/>
    </operation>
  </portType>

  <!-- ═══ 4. Binding (протокол) ═══ -->
  <binding name="OrdersSoapBinding" type="tns:OrdersPortType">
    <soap:binding style="document" transport="http://schemas.xmlsoap.org/soap/http"/>

    <operation name="createOrder">
      <soap:operation soapAction="http://example.com/orders/createOrder"/>
      <input>
        <soap:body use="literal"/>
      </input>
      <output>
        <soap:body use="literal"/>
      </output>
      <fault name="orderFault">
        <soap:fault name="orderFault" use="literal"/>
      </fault>
    </operation>
  </binding>

  <!-- ═══ 5. Service (endpoint) ═══ -->
  <service name="OrdersService">
    <port name="OrdersPort" binding="tns:OrdersSoapBinding">
      <soap:address location="http://example.com/orders"/>
    </port>
  </service>

</definitions>
```

Читается сверху вниз:
1. Есть типы (Request/Response/Fault).
2. Собираем в messages.
3. Определяем operations (createOrder: input → output/fault).
4. Как транспортировать (SOAP 1.1 over HTTP, document/literal).
5. Где (URL).

### 9.3 Styles: RPC vs Document

**RPC style** — body содержит имя операции + параметры:
```xml
<soap:Body>
  <createOrder>
    <customerId>c1</customerId>
    <amount>100</amount>
  </createOrder>
</soap:Body>
```

**Document style** — body содержит XML документ (по XSD):
```xml
<soap:Body>
  <CreateOrderRequest xmlns="http://example.com/orders">
    <customerId>c1</customerId>
    <amount>100</amount>
  </CreateOrderRequest>
</soap:Body>
```

**Document/literal** — стандарт modern SOAP. Body — XML документ, валидируется по XSD.

Используй document/literal, не RPC.

### 9.4 Encoding: literal vs encoded

- **literal** — body — валидный XML по XSD.
- **encoded** — SOAP-специфичное кодирование (устарело).

**literal** всегда.

### 9.5 Bindings

- **SOAP binding** — via SOAP (HTTP, SMTP).
- **HTTP binding** — обычный HTTP (POST/GET) без SOAP.

Обычно только SOAP binding.

---

## 10. WSDL 2.0 (редко)

W3C стандартизировал **WSDL 2.0** в 2007. Более чистый, но **никто не использует** — WSDL 1.1 победил.

Все существующие tools и клиенты — на 1.1.

---

## 11. Генерация Java из WSDL

### 11.1 wsimport (JAX-WS)

```bash
wsimport -keep -d target/generated -p com.example.client \
    http://server/orders?wsdl
```

Сгенерирует:
- Java-классы DTO (по XSD внутри WSDL).
- Интерфейс `OrdersPortType`.
- Класс сервиса `OrdersService`.
- Client stub.

Использование:
```java
OrdersService service = new OrdersService();
OrdersPortType port = service.getOrdersPort();

CreateOrderRequest req = new CreateOrderRequest();
req.setCustomerId("c1");
req.setAmount(BigDecimal.valueOf(100));

CreateOrderResponse resp = port.createOrder(req);
System.out.println(resp.getOrderId());
```

Java-код, SOAP внутри — скрыт.

### 11.2 cxf-codegen (Apache CXF)

Аналог с Apache CXF stack.

```xml
<plugin>
  <groupId>org.apache.cxf</groupId>
  <artifactId>cxf-codegen-plugin</artifactId>
</plugin>
```

Более гибкий, поддерживает WS-*.

### 11.3 Gradle wsimport plugin

```gradle
plugins {
    id 'com.github.bjornvester.wsdl2java' version '1.2'
}

wsdl2java {
    wsdlFiles.from fileTree('src/main/resources/wsdl')
    generatedSourceDir = layout.buildDirectory.dir('generated/wsdl')
}
```

---

## 12. Генерация WSDL из Java (server side)

**Code-first**: пишешь Java + аннотации → WSDL генерируется автоматом.

```java
@WebService(targetNamespace = "http://example.com/orders")
public class OrdersService {

    @WebMethod
    public CreateOrderResponse createOrder(
        @WebParam(name = "request") CreateOrderRequest request) {
        // ...
        return new CreateOrderResponse(orderId, "NEW");
    }
}
```

Deploy → сервис доступен по `?wsdl` — автогенерированный WSDL.

---

## 13. Реальные кейсы в ИСНА

Из memory и опыта:

### 13.1 XSD для ФНО

Формы налоговой отчётности (ФНО 200.00, ФНО 100.00, ФНО 270 и т.д.) описаны XSD.

Каждая версия формы — свой XSD (`FNO_270_2024_v1.xsd` и т.д.).

Из memory:
- **`knp-fno-nearest-revision-fallback`** — если ФНО за старый год без контента → ближайшая ревизия (по XSD schema).
- **`fno150-app03-print-split`** — 150.03 разбито на application_03_01..05 в контенте (XSD-подкоды).

Валидация XML ФНО против XSD — на приёмке в АРМ.

### 13.2 WSDL для ЕСБ

Гос-системы связаны через ЕСБ (SOAP-шина). Каждый сервис — свой WSDL.

Из memory `knp-fno-outer-sync-esb-dead-route` — SOAP-маршрут `BT_OUTER_FNOSYNC_OUTER_TAXREP_SYNC` в ЕСБ. Клиенты используют WSDL для генерации Java client.

Из memory `knp-fno-reception-api-mgu-vs-knp` — SOAP endpoint для КНП (WSDL от ЕСБ), REST от МГУ.

### 13.3 Kalkan ЭЦП

Kalkan — казахстанская библиотека ЭЦП. XSD описывает формат подписи (WS-Security XML Signature).

Обычно генерируются классы для работы с ЭЦП через xjc.

---

## 14. Проблемы и антипаттерны

### 14.1 Overly complex XSD

Гос-схемы часто с сотнями типов, глубокая иерархия extension. Тяжело поддерживать.

Fix: modular schemas + imports.

### 14.2 Изменение XSD ломает клиентов

Изменение схемы = re-generation client + redeploy.

Fix: **backward compatible** изменения:
- Добавление optional полей — OK.
- Удаление / rename полей — breaking.

Versioning: `orders_v1.xsd`, `orders_v2.xsd`.

### 14.3 Разные namespaces у похожих типов

Разные версии/системы = разные namespaces = не совместимы.

Fix: продумать namespace strategy.

### 14.4 xsd:anyType

```xml
<xsd:element name="data" type="xsd:anyType"/>
```

Отказ от типизации. Побочный эффект — никакой валидации.

Fix: конкретные типы.

### 14.5 Смешанный контент

```xml
<xsd:complexType mixed="true">
  <xsd:sequence>
    ...
  </xsd:sequence>
</xsd:complexType>
```

Разрешает текст между элементами (HTML-like). Сложно обрабатывать.

Fix: избегать.

---

## 15. Инструменты

### 15.1 SoapUI

Классика для тестирования SOAP + WSDL. Импорт WSDL → генерация requests → отправка.

### 15.2 Postman

Тоже поддерживает SOAP.

### 15.3 XMLSpy (Altova)

Enterprise IDE для XML / XSD / WSDL.

### 15.4 IntelliJ / Eclipse

Встроенная поддержка XSD, WSDL (validation, autocomplete).

### 15.5 wsdl4j

Java library для parsing WSDL.

---

## 16. Best practices

### 16.1 XSD

1. **Отдельные namespaces** для разных domain.
2. **Named types** (не inline) для переиспользования.
3. **Явные restrictions** (pattern, enum).
4. **`decimal` для денег**, не `float/double`.
5. **`dateTime` с timezone**.
6. **Documentation** через `<xsd:annotation><xsd:documentation>`.
7. **Modular** — imports/includes.
8. **Versioning** через namespaces.

### 16.2 WSDL

1. **Document/literal** style.
2. **Named messages** для reuse.
3. **Fault** объявлен для error cases.
4. **`soapAction`** явно указан.
5. **Полная документация** operations.
6. **Backward compatible** изменения.

---

## 17. Собесные вопросы

1. **Что такое XSD?** — XML Schema Definition; описывает структуру XML документов.
2. **Simple vs complex type?** — Simple: только значение (string, int); complex: с child элементами / атрибутами.
3. **Что такое namespace?** — URI идентификатор schema; уникальный между разными схемами.
4. **targetNamespace vs xmlns?** — targetNamespace — namespace определяемый в этой схеме; xmlns — используемые namespaces.
5. **Restrictions в XSD?** — pattern (regex), enumeration, minInclusive/maxInclusive, minLength/maxLength.
6. **minOccurs / maxOccurs?** — Минимум и максимум вхождений; unbounded = без ограничения.
7. **sequence vs choice vs all?** — Sequence: порядок фиксирован; choice: один из; all: все, любой порядок.
8. **Extension vs restriction?** — Extension добавляет поля; restriction сужает.
9. **Что такое WSDL?** — Web Services Description Language; описывает SOAP web service.
10. **5 секций WSDL?** — types, message, portType, binding, service.
11. **RPC vs Document style?** — RPC: имя операции + params в body; Document: XML документ по XSD.
12. **literal vs encoded?** — literal: валидный XML по XSD; encoded: SOAP-специфичное (устарело).
13. **Как сгенерировать Java из WSDL?** — `wsimport -keep -d target -p pkg URL_TO_WSDL`.
14. **Как сгенерировать Java из XSD?** — JAXB `xjc -d target -p pkg schema.xsd`.
15. **Backward compatible изменения XSD?** — Добавление optional полей OK; удаление / rename — breaking.

---

## Итог

- **XSD** = формальный контракт для XML данных.
- **Simple types** (базовые + restrictions), **complex types** (sequence/choice/all).
- **minOccurs/maxOccurs**, **attributes**, **extension/restriction**.
- **Namespaces** обязательны для isolation.
- **WSDL** = описание SOAP service (types + messages + portType + binding + service).
- **Document/literal** = modern стандарт SOAP.
- **wsimport** — генерация Java client из WSDL.
- **xjc / JAXB** — генерация Java DTO из XSD.
- В **ИСНА**: XSD для ФНО, WSDL для ЕСБ SOAP-services, Kalkan ЭЦП.

Итого **70 файлов в `isna-theory\`**.
