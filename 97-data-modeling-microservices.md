# 97. Моделирование данных для микросервисов

## Почему это критично

Плохая структура данных — самая дорогая ошибка в жизни системы. Плохой код можно рефакторить постепенно; плохую схему БД — практически невозможно поменять после того как в ней миллионы записей и десятки сервисов зависят от неё. Поэтому решения о моделировании данных, принимаемые в начале проекта, определяют его судьбу на годы.

В классической монолитной архитектуре моделирование сводится к вопросам нормализации: как разбить сущности на таблицы, где нужны foreign keys, где допустима денормализация, как обеспечить целостность через constraints. Это давно и хорошо изученная область, есть books, курсы, methodology.

В микросервисной архитектуре появляется совершенно новое измерение — **как разделить данные между сервисами**. Что становится отдельным сервисом со своей БД? Как сервисы общаются, если foreign keys между их данными невозможны? Как обеспечить консистентность когда одну бизнес-сущность обновляют несколько сервисов? Как быть с shared data — reference tables, user info, каталоги? Эти вопросы не имеют однозначных ответов, зависят от конкретного контекста, но есть проверенные подходы.

В этом файле разберём фундаменты моделирования данных с фокусом на микросервисы. Классические принципы нормализации и когда от них отклоняться. Foreign keys — плюсы и минусы, ограничения в микросервисной архитектуре. Database per service паттерн: полная изоляция vs shared database. Как справляться с shared reference data. Патenty CQRS для разделения read и write моделей. Event Sourcing как альтернативный подход к persistent state.

## Нормализация: что это и зачем

Нормализация — процесс структурирования БД так, чтобы минимизировать redundancy и dependency. Формально существует несколько нормальных форм (1NF до 6NF), но на практике важны первые три.

**Первая нормальная форма (1NF)**: каждое поле атомарно (нет массивов, повторяющихся групп), у каждой строки уникальный ключ.

Не 1NF:

```
| user_id | name | phones          |
| 1       | Bob  | 111, 222, 333  |
```

Поле phones содержит несколько значений. 1NF:

```
| user_id | name |    | phone_id | user_id | phone |
| 1       | Bob  |    | 1        | 1       | 111   |
                       | 2        | 1       | 222   |
                       | 3        | 1       | 333   |
```

**Вторая нормальная форма (2NF)**: 1NF + каждый non-key атрибут полностью зависит от primary key (не от его части).

Актуально когда primary key составной. Пример анти-паттерна:

```
| order_id | product_id | product_name | quantity |
```

product_name зависит только от product_id, не от order_id. Нарушает 2NF. Нормальная форма:

```
| order_id | product_id | quantity |    | product_id | product_name |
```

**Третья нормальная форма (3NF)**: 2NF + non-key атрибуты не зависят транзитивно от primary key.

Пример нарушения:

```
| user_id | name | city | country |
```

country зависит от city, city — от user_id, значит country транзитивно зависит через city. 3NF:

```
| user_id | name | city_id |    | city_id | city | country_id |    | country_id | country |
```

## Когда нормализовать, когда денормализовать

Полная нормализация — это идеал теории. На практике есть trade-off.

Плюсы нормализации. Отсутствие redundancy — обновляешь имя города в одной таблице, эффект везде. Целостность — можно enforcement'ить через FK. Экономия места — не дублируешь одно и то же в миллионе строк.

Минусы. JOIN'ы. Много JOIN'ов. `SELECT user + address + city + country` = 4-way JOIN. При больших таблицах — медленнее, чем читать одну денормализованную. При очень больших — существенно медленнее.

Стандартная стратегия — **normalize until it hurts, denormalize until it works**. Начинать с нормальной формы. Профилировать. Когда конкретные запросы становятся узким местом — селективно денормализовать.

Типичные обоснованные денормализации.

**Дублирование часто читаемых значений**. Хранить city_name в таблице users прямо, чтобы не JOIN'ить cities при списке пользователей. Update, если названия городов меняются, — редкое событие.

**Pre-calculated aggregates**. Хранить `orders_count` в таблице users, обновляемое через trigger. Не пересчитывать COUNT каждый раз.

**JSON колонки для sparse attributes**. Если у сущности много опциональных полей, разные для разных типов — JSONB вместо десятков nullable колонок или EAV table.

**История через отдельные таблицы**. Активные записи в основной таблице, старые/архивные в отдельной (archive_orders). Основная маленькая и быстрая.

Правило: денормализация — это оптимизация, применяемая когда есть конкретная проблема. Не преждевременная.

## Foreign keys: плюсы и минусы

Foreign keys — механизм гарантии referential integrity. Строка в orders ссылается на существующий user через `user_id`. PostgreSQL проверяет: нельзя вставить order с user_id, которого нет в users. Нельзя удалить user, если есть orders (или CASCADE удаляет их тоже).

Плюсы FK. Гарантия целостности на уровне БД — независимо от bug'ов в приложении. Явное documentation связей — схема сама себя объясняет. Optimizer использует для лучших планов (знает cardinality связей).

Минусы. Overhead на INSERT/UPDATE (проверка FK — extra queries). Ограничения на bulk operations (сложно delete parent если есть orphaned children). Cross-database FK невозможны — в микросервисах становятся проблемой.

В монолите с одной БД FK — почти всегда правильный выбор. Проверки быстрые (индексы), overhead минимален, надёжность больше кода приложения.

В микросервисах ситуация меняется. Если Order Service имеет orders, а User Service — users, между ними технически можно сделать FK, только если они делят одну БД. Но это нарушает принцип independent deployment: каждый сервис владеет своей БД, никто другой не должен туда лезть.

Три варианта.

**Share database**. Несколько сервисов используют одну БД, FK работают. Плюсы: consistency, простота. Минусы: сервисы связаны через схему БД, изменения ломают других, не независимы. По сути анти-паттерн для настоящих микросервисов.

**Database per service, no FK across services**. Каждый сервис — своя БД. FK только внутри одной БД сервиса. Между сервисами — soft references: user_id в orders, но без FK constraint. Целостность через application logic и eventual consistency.

**Shared reference data через каталоги**. Некоторые справочные данные (страны, валюты, категории) shared. Один сервис owns их, остальные читают через API или replicate локально.

## Database per service

Стандартный микросервисный паттерн. Каждый bounded context (в терминах DDD) — свой сервис со своей БД. Никто другой не имеет прямого access к чужой БД.

Пример архитектуры для КНП-подобной системы:

**User Service** — users, auth, profiles. Своя PostgreSQL с users table.

**Order Service** — orders, order_items. Своя PostgreSQL с orders. В order записывается user_id (soft reference, без FK constraint к User Service).

**Payment Service** — payments, invoices. Своя PostgreSQL. В payment — order_id (soft reference).

**Notification Service** — notifications, delivery status. Своя PostgreSQL.

Плюсы database per service. Independent deployment — каждый сервис можно менять без координации. Technology choice — один сервис на PostgreSQL, другой на MongoDB, третий на Redis. Team autonomy — команды не блокируют друг друга. Blast radius — сбой одной БД не валит всех.

Минусы. Distributed queries — не можешь JOIN между базами. Distributed transactions — нужны Saga или Outbox (файл 93). Data duplication — user_id в orders хранится, но не проверяется через FK, если user удалён — orphaned orders. Ops complexity — управлять N БД сложнее чем одной.

Практика показывает: strict database per service дорого поддерживать. Есть компромиссы.

**Bounded context, но одна БД instance**. Разные сервисы могут physically работать с одной PostgreSQL instance, но со своими схемами. Не share tables. Не FK через схемы. Ops дешевле (одна железка), логическая изоляция сохраняется.

**Some sharing, но контролируемое**. Некоторые reference tables shared через отдельную схему. All read-only для всех сервисов кроме owner'а. Изменения через специальный протокол.

Balance зависит от размера команды, зрелости, реальной необходимости в независимости. Startup из 20 человек с 5 микросервисами — одна БД instance, несколько схем — правильный выбор. Enterprise с 100+ инженерами и 30+ сервисами — настоящая database per service оправдана.

## Shared reference data

Проблема. Есть таблица countries (страны). Используется во всех сервисах. Как её организовать?

Вариант 1: **shared table в общей БД**. Все сервисы читают. Изменения контролируются одним service owner (например Reference Data Service). Работает, но связывает сервисы через схему.

Вариант 2: **API access**. Countries service предоставляет REST API, остальные обращаются per запрос. Простой, но медленный (network hop на каждое использование).

Вариант 3: **Local cache**. Countries service предоставляет API, каждый сервис кэширует локально (in-memory или в своей БД). Обновляется периодически или через events.

Вариант 4: **Event-driven replication**. Countries service публикует CountriesChanged events, каждый сервис держит свою локальную копию, обновляемую через события. Быстро (локальный доступ), масштабируется.

Для типичных reference data (справочники, каталоги, редко меняющиеся lookups) вариант 3 или 4 — правильный. Данные близко к запросу, никаких network hops, изменения eventually распространяются.

Для КНП: справочники налоговых форм, кодов, статусов — replicate через events во все сервисы. Каждый сервис имеет свою локальную копию, читает мгновенно.

## Аggregates и bounded contexts

Domain-Driven Design (DDD) даёт концептуальные инструменты для разбиения на сервисы.

**Aggregate** — кластер объектов, которые обрабатываются как единое целое. Order + OrderItems — aggregate. Все операции на этот aggregate атомарны. Все invariants (например, order.total = sum(items.price)) проверяются внутри aggregate.

Правило: **одна транзакция — один aggregate**. Не пытайся изменить несколько aggregates в одной транзакции — это ведёт к coupling. Между aggregates — eventual consistency через events.

**Bounded context** — граница модели. Одно и то же понятие «Order» может значить разное в разных контекстах: в Order Service Order — это заказ пользователя со списком items, статусами. В Payment Service Order — это просто идентификатор с суммой. В Shipping Service Order — это отправление с адресом. Каждый контекст имеет свою модель, свои инварианты.

Каждый bounded context — кандидат в отдельный микросервис со своей БД.

DDD даёт framework для правильного разделения. Начни с event storming — соберите команду, обозначьте бизнес-события (OrderPlaced, PaymentProcessed, OrderShipped). Обнаружь aggregates (какие сущности вокруг каких events). Найди bounded contexts (какие aggregates логически объединены). Микросервисы естественно вырастают из этого анализа.

## CQRS

**Command Query Responsibility Segregation** (CQRS) — паттерн разделения read и write моделей. Идея: часто оптимизации для чтения и записи расходятся. Write требует normalized structure для integrity. Read хочет denormalized для быстрого поиска. CQRS предлагает: имей две модели, одна для write, другая для read.

Классическая монолитная модель. Одна БД, одна схема, приложение и пишет, и читает.

CQRS. Write model: normalized, protected by constraints, оптимизирована для transactions. Read model(s): denormalized, оптимизированы под конкретные query patterns, могут быть в другой БД (даже другого типа — например Elasticsearch для search-heavy reads).

Синхронизация. Write model изменяется, публикуются events. Read models обновляются по events. Eventual consistency.

Пример. Order Service имеет write model в PostgreSQL: normalized tables orders, order_items, customers. Для read scenarios — read model в Elasticsearch с денормализованными order records включающими имя клиента, полный список items, статус — всё в одном document. UI reads из Elasticsearch (быстро, нет JOINов). Write goes в PostgreSQL, event OrderUpdated обновляет Elasticsearch document.

Плюсы CQRS. Оптимизация под нагрузку — writes и reads scale независимо. Read models могут быть тяжело кэшированы. Разные технологии для разных задач (PG для writes, ClickHouse для analytics).

Минусы. Complexity — две модели вместо одной, синхронизация. Eventual consistency — read может отставать от write. Debug сложнее.

Не применяй CQRS везде. Применяй когда:
- Reads существенно чаще writes (10:1 или больше).
- Read queries сильно отличаются по optimization требованиям от writes.
- Есть конкретные performance проблемы, которых нельзя решить обычной оптимизацией.

Для типичного CRUD приложения CQRS — overkill.

## Event Sourcing

Ещё один паттерн, часто идущий вместе с CQRS. Обычные модели хранят **текущее состояние**: users имеет запись Bob с текущим email. При update email — старое значение перезаписывается.

Event Sourcing переворачивает: хранишь **последовательность событий**, приведших к текущему состоянию. Bob создан — UserCreated event. Email изменён — UserEmailChanged event. Bob деактивирован — UserDeactivated event. Текущее состояние вычисляется как fold всех events.

Приложение:

```
CREATE TABLE user_events (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    version INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (aggregate_id, version)
);
```

Каждое изменение — INSERT event. UPDATE не бывает. DELETE не бывает.

Плюсы Event Sourcing. Полная история — можешь reconstruct состояние на любой момент времени. Audit trail бесплатно. Debug легче — все изменения записаны с контекстом. Multiple projections — из одного event stream можешь построить разные views.

Минусы. Complexity — приложение думает в events, не в таблицах. Больше storage (события накапливаются). Rehydration — считать N событий чтобы получить текущее состояние. Solved через snapshots (периодически сохранять текущее state, replay только после последнего snapshot).

Event Sourcing — мощный подход для систем где audit trail критичен (финансы, healthcare, regulatory). Для типичного CRUD — overkill.

Часто ES + CQRS вместе. Events — write model. Projections (built from events) — read models. Read models могут быть материализованы в отдельных БД (PostgreSQL, Elasticsearch, Redis) под конкретные query patterns.

Технологии: EventStoreDB (специализированная БД для events), Axon Framework (Java, включает CQRS + ES), Kafka как event log с проекциями. Можно и на обычном PostgreSQL — таблица events с триггерами для проекций.

## Практические принципы

Сводя всё вместе, несколько рабочих принципов.

**Начинай с нормализации**. Проще денормализовать позже когда возникнет конкретная проблема, чем нормализовать после того как денормализованные данные расползлись.

**Foreign keys в пределах сервиса — да**. Между сервисами — soft references с обработкой orphaned data в приложении.

**Database per service — цель, не догма**. Если операционная сложность больше пользы — one instance, multiple schemas. Логическая изоляция важнее физической.

**Aggregates — единица транзакции**. Один Aggregate — одна транзакция. Between aggregates — events, eventual consistency.

**Reference data replicate через events**. Не тяни через network в каждом запросе. Кэшируй локально.

**CQRS только когда есть реальная причина**. Разные optimization requirements для read и write. Не «модно и современно».

**Event Sourcing для audit-critical**. Финансы, healthcare, regulatory. Для обычного CRUD — обычная модель проще.

**Schema evolution думать заранее**. Каждое поле — на всю жизнь. Nullable — обязательно для новых полей. Не переименовывать (expand-contract). Не удалять (deprecated + eventual cleanup через месяцы).

**Data validation в нескольких местах**. Constraint в БД + validation в приложении + validation в API. Redundant, но каждый уровень ловит разные ошибки.

## Заключение

Моделирование данных — самая долгосрочная часть системы. Плохую схему поменять сложно, дорого, рискованно. Хорошая схема служит годами, поддерживает эволюцию бизнеса, масштабируется.

Начинай с нормализации до 3NF, денормализуй селективно когда возникают конкретные проблемы производительности. FK для integrity внутри одной БД. Между сервисами — soft references с eventual consistency.

Database per service — стандартный микросервисный паттерн, но с компромиссами. Полная изоляция дорого; часто one instance, multiple schemas — правильный компромис. Reference data replicate через events в каждый сервис.

DDD даёт framework для правильного разбиения. Aggregates — единицы транзакций и модели. Bounded contexts — кандидаты в микросервисы.

CQRS — разделение read и write моделей когда их optimization requirements существенно отличаются. Не применяй везде.

Event Sourcing — хранение events вместо текущего состояния. Полный audit trail, всё restorable. Complex, применяй когда audit critical.

Для КНП: каждый bounded context (kabinet, arm, tax-rep, forms processing) — свой сервис со своей БД (или отдельной схемой в общей PG). Reference data (справочники, коды, каталоги) — replicate через events в каждый сервис. FK внутри сервиса, между — soft references. CQRS для отчётов (write model normalized, read model materialized). Event Sourcing вероятно для аудита форм (regulatory требование).

Дальше — практика. Возьми существующий сервис в проекте, попробуй explicit'но выделить aggregates и bounded contexts через event storming (даже соло). Посмотри где нарушается aggregate boundary. Замерь запросы, которые крoss-context — правильные ли они. Каждый такой анализ выявляет проблемы моделирования, которые до сих пор жили тихо, ожидая роста.
