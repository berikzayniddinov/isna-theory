# 65. ИШ / Интеграционные шины / ESB

Что такое интеграционная шина, зачем нужна, где ещё используется.

---

## 1. Что такое интеграционная шина

**Интеграционная шина** (ИШ / **ESB** — Enterprise Service Bus) — централизованное middleware для интеграции разных систем.

Идея:
- Много приложений разной природы (Java monoliths, SAP, mainframe, БД, Web-сервисы).
- Все нужно связывать (обмен данными, вызовы).
- Вместо **N × N** point-to-point интеграций — все через **центральную шину**.

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

Плюсы:
- **Loose coupling** — приложения не знают друг о друге.
- Централизованная логика (routing, security, monitoring).
- Легко добавить новое приложение.

Минусы:
- ESB — SPOF (single point of failure).
- Централизованная бизнес-логика ("smart pipe, dumb endpoints") — трудно поддерживать.
- Vendor lock-in.

---

## 2. Классические функции ESB

### 2.1 Routing

Определить куда доставить сообщение. Content-based routing:
```
Если Order.type = "EXPRESS" → shipping-service
Если Order.type = "REGULAR" → warehouse-service
```

### 2.2 Transformation

Одна система шлёт XML, другая ожидает JSON. Одна — camelCase, другая — snake_case.

ESB преобразует **на лету**.

Инструменты: XSLT (XML), XPath, JOLT (JSON), custom mapping.

### 2.3 Mediation

Между несовместимыми протоколами:
- SOAP → REST.
- HTTP → JMS.
- File → Kafka.

### 2.4 Protocol bridging

Клиент SOAP, backend REST. ESB — переводчик.

### 2.5 Message enrichment

Добавить данные из других систем в сообщение по пути:
```
Order → [ESB: спросить у customer-service имя клиента] → order+customerName → dest
```

### 2.6 Security

Централизованная аутентификация / шифрование.

### 2.7 Auditing / Monitoring

Все сообщения через шину — легко логировать / метрики.

### 2.8 Guaranteed delivery

С persistent queues — сообщение не потеряется.

---

## 3. Классические ESB продукты

Enterprise Java экосистема (1990-2010е):
- **Apache ServiceMix** (open source).
- **MuleSoft** (сейчас Salesforce).
- **IBM Integration Bus (IIB / DataPower)**.
- **Oracle Service Bus (OSB)**.
- **TIBCO BusinessWorks**.
- **Microsoft BizTalk**.
- **JBoss ESB** (deprecated).
- **Apache Camel** — не ESB, но integration framework часто в связке.

Все — тяжелые, dorogих enterprise systems.

---

## 4. Пример работы (Camel-style)

**Apache Camel** — популярный integration framework (может работать как lightweight ESB):

```java
@Component
class OrderRoute extends RouteBuilder {
    @Override
    public void configure() {
        // из JMS-очереди
        from("jms:queue:orders.incoming")
            // трансформация XML → JSON
            .marshal().json(JsonLibrary.Jackson)
            // enrichment
            .enrich("http://customer-service/api/customers/{customerId}",
                    new CustomerEnricher())
            // routing по content
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

Читабельно: **входящий JMS → transform → enrich → routing → outgoing HTTP/JMS**.

Camel — 300+ компонентов (JMS, HTTP, File, Kafka, Salesforce, S3, ...).

---

## 5. EAI patterns

**Enterprise Integration Patterns** (Hohpe & Woolf, 2003) — классическая книга.

Ключевые паттерны:
- **Message Channel** — очередь между sender и receiver.
- **Message Endpoint** — как приложение подключается к каналу.
- **Message Router** — выбирает канал по content.
- **Message Translator** — трансформирует формат.
- **Content Enricher** — добавляет данные.
- **Content Filter** — убирает лишнее.
- **Aggregator** — собирает связанные сообщения в одно.
- **Splitter** — разбивает одно на много.
- **Wire Tap** — копия для audit.
- **Dead Letter Channel** — куда деть проблемные сообщения.
- **Idempotent Receiver** — избежать дублей.

Все реализуются в ESB / Camel / любом integration tool.

---

## 6. Почему ESB deprecated в новых системах

С развитием микросервисов и cloud-native подход изменился.

### 6.1 "Smart endpoints, dumb pipes"

Мартин Фаулер: логика — в **приложениях**, не в шине. Шина — просто транспорт.

ESB накапливает бизнес-логику → превращается в **integration monolith**. Одну строчку изменить → тестирование всей шины.

### 6.2 Микросервисы вместо интеграции

Раньше приложения были монолитами (SAP + Oracle + custom Java). ESB интегрировала.

Теперь приложения сами — набор микросервисов, общаются через HTTP/events. ESB не нужен.

### 6.3 Cloud-native альтернативы

- **API Gateway** — routing + auth (обычно).
- **Message broker** (Kafka/Rabbit/SQS) — async messaging.
- **Service Mesh** (Istio/Linkerd) — mTLS, retry, observability.
- **Serverless / Cloud Functions** — трансформации.

Каждый инструмент делает одну вещь хорошо. Композиция > монолитный ESB.

### 6.4 DevOps / Continuous Deployment

ESB deploy — недели (изменения в central hub). Микросервисы — минуты.

### 6.5 Vendor lock-in

Проприетарные ESB — тяжело мигрировать.

---

## 7. Где ESB ещё живёт

Не всё умерло. ESB жив в:

### 7.1 Enterprise Java (banking, insurance, gov)

Legacy monoliths + SAP + mainframe требуют интеграции. Нельзя переписать всё.

### 7.2 Регулируемые отрасли

Строгий audit, security, contract compliance. ESB даёт централизованный контроль.

### 7.3 Государственные системы

Как ИСНА в Казахстане.

### 7.4 Modern ESB — iPaaS

**iPaaS (Integration Platform as a Service)** — cloud-based ESB:
- **MuleSoft Anypoint Platform**.
- **Boomi**.
- **Workato**.
- **AWS Step Functions + EventBridge**.
- **Azure Logic Apps**.

Легче chем legacy ESB, но идея та же.

---

## 8. ИШ в контексте ИСНА

### 8.1 ЕСБ — Единая Сервисная Шина

В ИСНА есть **ЕСБ** — центральная integration bus для интеграции с внешними государственными системами.

Через ЕСБ идут:
- Взаимодействие с системами других ведомств.
- Приёмка ФНО / ФО (`save-fno-<code>` SOAP endpoints).
- Синхронизация данных между налоговой и другими системами.
- Проверки ИИН/БИН, регистрационных данных.

Из memory:
- **`knp-fno-outer-sync-esb-dead-route`** — SOAP "Requested service is not found" когда маршрут `BT_OUTER_FNOSYNC_OUTER_TAXREP_SYNC` не поднят на ЕСБ.
- Логика: `KnpOuterSystemFnoSyncService` вызывает ЕСБ → падает если маршрут не настроен → 1586×/13ч ошибок.

Урок: **зависимость от ЕСБ = зависимость от инфры**. Circuit breaker + флаг отключения важны.

### 8.2 Каналы ЛС-ИШ-ОС

Из memory `knp-e2e-prod-smoke-scope-rules`:
> «не смокать мутирующие/опасные API (send/запись/запрос в ЛС-ИШ-ОС под ЭЦП владельца)»

Что означает:
- **ЛС** — **Личный Счёт** налогоплательщика (транзакции, разноска платежей).
- **ИШ** — **Интеграционная Шина** (обмен с внешними системами).
- **ОС** — **Отправка/Обмен Сообщениями** (или Openservice? "Отчётная Система").

Три канала критичных операций требующих ЭЦП владельца.

Правило смока: **не мутировать ничего в эти каналы** (не создавать нагрузки на прод / не влиять на состояние ЛС / не слать в ИШ / не отправлять в ОС).

### 8.3 SOAP vs REST в ИСНА

- **Внешний слой** (интеграция с гос. системами) — SOAP через ЕСБ/ИШ.
- **Внутренний слой** (микросервисы КНП, АРМ) — REST через Feign + Consul.
- **Gateway** — `isna-knp-gateway` (Zuul) на входе, роутит.

Так исторически: SOAP был стандартом gov, REST пришёл позже для user-facing.

---

## 9. ESB vs API Gateway

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

**API Gateway** — edge для internet клиентов.
**ESB** — hub для internal системной интеграции.

Иногда сливаются в одном продукте (MuleSoft умеет оба).

В ИСНА:
- **isna-knp-gateway** — API Gateway (Zuul) для внешних клиентов.
- **ЕСБ/ИШ** — ESB для интеграции с гос-системами.

---

## 10. ESB vs Message Broker

| | ESB | Broker (Rabbit/Kafka) |
|---|---|---|
| Логика | В шине | В consumer'ах |
| Transformation | Да | Нет (в приложении) |
| Routing | Content-based | Простой (queue/topic name) |
| Enrichment | Да | Нет |
| "Smartness" | Smart pipe | Dumb pipe |

**Broker** — простой транспорт. **ESB** — умный посредник.

Современный подход: **broker + smart consumers**.

---

## 11. ESB vs Service Mesh

**Service Mesh** (Istio, Linkerd) — infrastructure layer для sidecar proxy.

| | ESB | Service Mesh |
|---|---|---|
| Уровень | Приложение | Инфраструктура |
| Реализация | Центральный broker | Sidecar на каждом service |
| Функции | Transformation, routing, business logic | Retry, mTLS, tracing, LB |
| Coupling | High (все через ESB) | Low (transparent) |

Service Mesh — **cross-cutting infrastructure** без бизнес-логики.

ESB — **business-level integration**.

Разные слои.

---

## 12. Когда ещё нужен ESB

Не для новых микросервис-проектов. Но:

### 12.1 Legacy integration

10 monolithic apps, каждый со своим API → ESB решает.

### 12.2 B2B integration

Партнёры со своими протоколами (SOAP, EDI, файлы). ESB — точка перевода.

### 12.3 Enterprise workflows

Сложные orchestration (BPEL) — ESB часто включает workflow engine.

### 12.4 Централизованный audit / compliance

Регулятор требует "все сообщения через одну точку".

### 12.5 Gov / regulated

Как ИСНА — исторически SOAP + ESB для gov integration.

---

## 13. Anti-patterns

### 13.1 Business logic в ESB

Расчёты, валидации, decisions — в приложениях, не в шине.

### 13.2 Огромный ESB monolith

Один ESB для всей компании → все зависят от одной команды.

Правильно — **несколько специализированных** (по domain).

### 13.3 God's message

Одно сообщение несёт всё для всех. Sender не знает что receiver'ы возьмут → coupling скрыт.

### 13.4 Sync через async broker

RPC поверх Rabbit/Kafka — ждёшь ответ через тот же broker. Медленно, сложно debug.

---

## 14. Modern подход к integration

**Choreography** (см. `50-saga-pattern.md`):
- Каждый сервис публикует events.
- Другие подписываются.
- Никого центрального посредника.

Инструменты:
- **Kafka + Schema Registry** — event streaming backbone.
- **Rabbit** — task queues.
- **API Gateway** — HTTP edge.
- **Service Mesh** — infrastructure.
- **Event Sourcing / CQRS** — data.

Distributed, cloud-native, микросервисы. ESB не нужен.

---

## 15. Собесные вопросы

1. **Что такое ESB?** — Enterprise Service Bus — централизованное middleware для интеграции систем.
2. **Функции ESB?** — Routing, transformation, mediation, enrichment, protocol bridging.
3. **Плюсы ESB?** — Loose coupling, централизованная логика/security/monitoring.
4. **Минусы ESB?** — SPOF, integration monolith, vendor lock-in, медленные deploys.
5. **ESB vs API Gateway?** — ESB: internal integration hub; Gateway: external entry point.
6. **ESB vs Message Broker?** — ESB smart pipe (transform, route); broker dumb pipe (просто транспорт).
7. **ESB vs Service Mesh?** — ESB business-level integration; mesh infrastructure cross-cutting.
8. **Почему ESB deprecated?** — Микросервисы + broker + gateway + mesh делают то же лучше.
9. **Что такое Apache Camel?** — Java integration framework; часто в связке с ESB.
10. **Что такое EAI patterns?** — Enterprise Integration Patterns (Hohpe/Woolf) — паттерны интеграции.
11. **Где ESB ещё используется?** — Banking, insurance, government, regulated industries, legacy integration.
12. **iPaaS — что?** — Integration Platform as a Service; cloud-based ESB.
13. **"Smart endpoints, dumb pipes" — что значит?** — Логика в приложениях, шина — просто транспорт.
14. **ЕСБ в ИСНА — для чего?** — Интеграция с внешними государственными системами (SOAP).
15. **Anti-patterns ESB?** — Business logic в ESB, огромный monolith, sync-over-async.

---

## Итог

- **ESB** = централизованный middleware для интеграции.
- Функции: **routing, transformation, mediation, enrichment**.
- Deprecated в микросервисах (broker + gateway + mesh лучше).
- Ещё используется в enterprise / government / legacy.
- В **ИСНА**: **ЕСБ/ИШ** для интеграции с внешними гос-системами (SOAP-based).
- **Каналы ЛС-ИШ-ОС** — критичные, не смокать mutating APIs.
- Modern подход: **choreography** через events + smart consumers.

Следующий — `66-sync-vs-async.md`.
