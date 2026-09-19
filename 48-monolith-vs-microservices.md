# 48. Монолит vs Микросервисы

Что это, когда что выбирать, как мигрировать.

---

## 1. Монолит

**Монолит** = **одно приложение**, один процесс, одна кодовая база, один deploy.

Всё бизнес-логика — внутри одного приложения. Модули общаются через **вызовы функций** в памяти.

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

Deploy: собрал WAR, положил в Tomcat. Одно приложение — один Tomcat.

---

## 2. Микросервисы

**Микросервисы** = **много маленьких сервисов**, каждый:
- Свой процесс (обычно в контейнере).
- Своя кодовая база / repo.
- Свой deploy.
- Своя (часто) БД.
- Общается с другими через **сеть** (HTTP / gRPC / messaging).

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

Deploy: каждый сервис независимо. `git push` в orders repo → CI собирает orders → K8s deploy только orders.

---

## 3. ИСНА как пример

Из memory:
- `isna-knp-integration` — приёмка ФНО из КНП в tax-rep.
- `isna-knp-user` — пользователи, permissions.
- `isna-knp-fno` — работа с ФНО.
- `isna-knp-notification` — уведомления.
- `isna-knp-fs` — файловое хранилище.
- `isna-knp-gateway` — API-gateway (Zuul, Java 11).
- `isna-fno` — сервис форм налоговой отчётности.
- `isna-fo` — формы отчётности.
- `tax-report`, `tax-rep` — АРМ налогового органа.
- + ещё десятки.

Общаются через:
- **Feign clients** (sync HTTP через Consul).
- **RabbitMQ** (async).

Классическая микросервисная архитектура. Каждый сервис — отдельный git repo, отдельный CI, отдельный K8s Deployment, часто своя БД.

---

## 4. Сравнение: за/против

### 4.1 Монолит — плюсы

- **Простота разработки** — один проект, один IDE-workspace.
- **Простота deploy** — один WAR/JAR.
- **Простота отладки** — стек вызовов в одном процессе, breakpoint работает через все модули.
- **ACID transactions** — можно сделать всё в одной DB-tx.
- **Быстрая коммуникация** — function call ~ 1 нс vs HTTP ~1-10 мс.
- **Меньше инфраструктуры** — одна БД, одна JVM, простой мониторинг.
- **Меньше overhead** для маленькой команды.

### 4.2 Монолит — минусы

- **Scale только целиком** — не можешь скейлить только payment logic.
- **Один падает — всё падает** (single point of failure на процесс).
- **Долгий deploy** — маленькое изменение → пересобрать всё WAR → deploy.
- **Coupling растёт** — с ростом кода модули переплетаются.
- **Один stack** — весь монолит на Java или .NET, нельзя piece написать на Go.
- **Долгий CI/CD** — тесты всей системы часами.
- **Ограничение команды** — 100 разработчиков в одном repo = хаос.

### 4.3 Микросервисы — плюсы

- **Независимый scale** — payment под нагрузкой → 10 pods; остальные — 1.
- **Независимый deploy** — команды не мешают друг другу.
- **Fault isolation** — упал notification → user login работает.
- **Разные stacks** — orders на Java, ML на Python, gateway на Go.
- **Меньшие команды на сервис** — 2-8 человек, ownership.
- **Быстрее CI** — тесты только своего сервиса.
- **Легче переписать сервис** — один маленький, а не весь монолит.

### 4.4 Микросервисы — минусы

- **Distributed system complexity** — сеть ненадёжна, latency, partial failure.
- **Distributed transactions** — нет ACID через несколько сервисов; Saga, Outbox, compromise.
- **Debugging сложнее** — traceId по сервисам, множество логов.
- **Много инфраструктуры** — kubernetes, service mesh, message broker, monitoring.
- **Overhead коммуникации** — HTTP round-trip vs function call.
- **Data consistency** — eventual, не strong.
- **Сложность deploy** — нужен CI/CD per сервис, разные версии.
- **Testing сложнее** — интеграция много сервисов.
- **Больше стоимость** — больше подов, больше compute.
- **Не для маленьких команд** — 5 человек в 20 сервисах = боль.

---

## 5. Distributed monolith — анти-паттерн

**Distributed monolith** = микросервисы, но:
- Все жёстко связаны (нельзя поменять один без всех).
- Deploy одновременно.
- Общая БД.
- Sync HTTP chain (A → B → C → D → E, если один упал, все упали).

Худшее из обоих миров: **сложность микросервисов + coupling монолита**.

**Симптомы**:
- Все сервисы deployment'ся одновременно.
- Изменение одного API ломает 5 других.
- Общая БД, все читают/пишут.
- Circular dependencies между сервисами.

**Fix**: правильные boundaries (DDD), event-driven communication, database per service.

---

## 6. Modular monolith — компромисс

**Modular monolith** = монолит с **чёткими границами модулей внутри**.

- Один процесс, один deploy.
- Но модули изолированы (пакетная структура, не общие модели).
- Общение между модулями через **явные интерфейсы** (не прямые вызовы random классов).

Плюсы:
- Простота монолита.
- Готов к разделению на микросервисы если понадобится.

Минусы:
- Требует дисциплины (легко нарушить границы).

**Рекомендация большинства** (Martin Fowler, Sam Newman): **начинать с модульного монолита**, разделять только когда реально нужно.

---

## 7. Когда что выбирать

### 7.1 Стартап / MVP

**Монолит**. Быстрая разработка, малая команда, ещё не знаешь домен.

### 7.2 Маленькая команда (< 10 человек)

**Монолит**. Overhead микросервисов не окупается.

### 7.3 Средний продукт с большой командой

**Modular monolith** или **несколько крупных сервисов** (не микросервисов). 3-5 сервисов по бизнес-доменам.

### 7.4 Большая система, много команд

**Микросервисы**. Каждая команда — свой сервис.

### 7.5 Разные требования к scale

**Микросервисы**. Payment под жёсткой нагрузкой, admin — редкий → раздельный scale.

### 7.6 Legacy монолит с постоянными inc'ов

**Разделение через Strangler pattern** (см. §9).

---

## 8. Коммуникация между микросервисами

### 8.1 Sync

**HTTP / REST** (JSON) — стандарт.
**gRPC** — быстрее, типизированно (Protobuf).
**GraphQL** — гибкие запросы (для UI backend).

Плюсы: простота, request-response.
Минусы: caller зависит от callee (upstream down → downstream упал).

### 8.2 Async

**Message broker** (Rabbit, Kafka).

Producer публикует событие. Consumer(s) обрабатывают. Producer не знает про consumers.

Плюсы: decoupling, buffering, retry.
Минусы: eventual consistency, сложность отладки, need broker.

### 8.3 Правило

**Sync** — когда нужен immediate response (например, «покажи данные пользователя»).

**Async** — для side effects, notifications, event propagation («заказ создан — отправь email»).

**Идеал**: sync только через API Gateway; между backend-сервисами — async events.

---

## 9. Strangler pattern (migration)

Как из монолита сделать микросервисы **постепенно**.

Идея: **не переписывать всё за раз**. Наоборот:
1. Поставить API Gateway перед монолитом.
2. Выделить одну функциональность в новый микросервис.
3. Роутить эту функциональность через gateway → в новый сервис.
4. Остальное — старый монолит.
5. Постепенно выделять другие модули.
6. Через 2-5 лет монолит «задушен» (strangled) — все функции в микросервисах.

Плюсы:
- Работает всё время (нет big bang).
- Можно откатить любой шаг.
- Обучение команды параллельно.

Минусы:
- Долго.
- Двойная поддержка (монолит + микросервисы).
- Требует четкого API Gateway.

---

## 10. Data management

### 10.1 Database per service

**Правило**: каждый микросервис — своя БД.

Плюсы:
- Изоляция schema.
- Свой tuning.
- Свободный refactoring.

Минусы:
- Дубли данных (customer info в orders + payments).
- Сложность консистентности (см. Saga, Outbox).

### 10.2 Shared database — анти-паттерн

Если все сервисы читают/пишут общую БД → **distributed monolith**.

Проблемы:
- Изменение schema ломает все сервисы.
- Кто-то держит lock — все ждут.
- Coupling через данные.

Иногда допустимо для legacy миграции с постепенным разделением.

### 10.3 CQRS

**Command Query Responsibility Segregation** — разделение write и read моделей.

- Command side — write model (нормализованная).
- Query side — read model (денормализованная, оптимизированная).
- Sync через events / CDC.

Полезно когда read/write очень разные требования (много reads, complex analytics).

### 10.4 Event Sourcing

Хранить **события** (что произошло), а не текущее состояние.

- `OrderCreated`, `OrderPaid`, `OrderShipped` — записи.
- Current state = replay всех events.

Плюсы:
- Audit из коробки.
- Time-travel debug.
- Естественно для event-driven.

Минусы:
- Сложно (не для CRUD-приложений).
- Snapshots для performance.
- Migration event-schema.

---

## 11. Deployment

Микросервисы **предполагают**:
- **Docker** контейнеризация.
- **K8s** оркестрация.
- **CI/CD** per service.
- **Blue-green / Canary** deployment.
- **Service mesh** для traffic (Istio, Linkerd) — опционально.
- **Observability** (Prometheus, ELK, Jaeger).

Все это = **infrastructure investment**. Не начинай с микросервисов если нет ресурсов на platform team.

---

## 12. Ошибки при выборе

### 12.1 «Микросервисы = современно»

Нет, микросервисы = инструмент. Стартап с 5 человек → пиши монолит.

### 12.2 «Разделим на 50 сервисов сразу»

Начни с 3-5. Разделяй только когда чувствуешь боль монолита.

### 12.3 «Каждая сущность — свой сервис»

Нет, **по бизнес-домену / bounded context** (см. `49-microservices-decomposition.md`).

### 12.4 «Common library для всех»

Опасно. Легко превратить микросервисы в distributed monolith. Общий код — только stable утилиты (в ИСНА — `isna-commons`, `isna-global`).

### 12.5 «Каждая команда — свой стек»

Плюс: свобода. Минус: нет переиспользования, migration новых людей сложна.

Обычно — договориться на 1-2 stacks (Java + Node, или Java only).

---

## 13. Real-world: ИСНА-опыт

Микросервисы в ИСНА:
- Много сервисов (~30-50).
- Java stack.
- Общие библиотеки (isna-commons, isna-global) — держим единый стиль.
- **API Gateway** — Zuul (legacy, Java 11 only).
- **Consul** для discovery.
- **RabbitMQ** для async.
- **PostgreSQL** — своя БД у большинства сервисов.
- **Keycloak** для auth.
- **K8s** для orchestration.
- **ELK** для logs.

Каветы (из memory):
- Гейт остался на Java 11 (`knp-gateway-no-java21`) — миграция трудна.
- Общие библиотеки создают coupling (`isna-commons-dual-keep`) — держим дважды для Java 11 и 21.
- Много sync HTTP → risk каскадных отказов → нужны Circuit Breaker + timeouts.

---

## 14. Собесные вопросы

1. **Разница монолит и микросервисы?** — Монолит: один процесс/deploy/DB; микросервисы: много сервисов, свои DB, сеть между ними.
2. **Плюсы монолита?** — Простота, ACID, быстрая коммуникация, меньше инфры, легче debug.
3. **Плюсы микросервисов?** — Независимый scale/deploy, fault isolation, разные stacks, меньшие команды.
4. **Минусы микросервисов?** — Distributed complexity, eventual consistency, много инфры, сложный debug.
5. **Что такое distributed monolith?** — Микросервисы, но deploy'ятся вместе + tight coupling; худшее из обоих.
6. **Что такое modular monolith?** — Монолит с чёткими границами модулей; часто оптимальный старт.
7. **Что такое Strangler pattern?** — Постепенная миграция монолит→микросервисы, через API Gateway.
8. **Database per service — зачем?** — Изоляция schema, свободный refactor, но требует Saga/Outbox для consistency.
9. **Sync vs async communication — когда что?** — Sync для immediate response; async для side effects/notifications/decoupling.
10. **Когда микросервисы плохой выбор?** — Стартап, маленькая команда, нет infrastructure resources.
11. **Как избежать distributed monolith?** — Database per service, event-driven, независимый deploy, четкие boundaries.
12. **CQRS?** — Separation read и write моделей; для сложных read-паттернов.
13. **Event sourcing?** — Хранить события, не состояние; audit + time-travel; сложнее CRUD.
14. **API Gateway pattern?** — Единая точка входа для внешних клиентов; routing, auth, rate limit.
15. **Как начать миграцию монолита?** — Strangler: API Gateway + выделять функциональности одну за раз.

---

## Итог

- **Монолит**: простота, ACID, но scale/deploy как одно.
- **Микросервисы**: масштаб/independence, но distributed complexity.
- **Distributed monolith** — избегай.
- **Modular monolith** — оптимальный старт.
- **Database per service** — правило микросервисов.
- **Sync sparingly**, **async предпочтительно** между сервисами.
- **Strangler** для миграции.
- В **ИСНА** — микросервисы + Consul + Rabbit + K8s + общие commons библиотеки.

Следующий — `49-microservices-decomposition.md`.
