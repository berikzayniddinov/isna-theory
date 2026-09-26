# 23. RabbitMQ: паттерны и troubleshooting в production

## Паттерны использования

RabbitMQ применяется в различных сценариях каждый со своими характеристиками. Понимание базовых паттернов помогает выбирать правильный подход для конкретной задачи.

Work queue паттерн — самый классический. Одна очередь и много consumers обрабатывающих сообщения параллельно. Broker распределяет сообщения между consumers по round-robin с учётом prefetch и ack. Используется для распределённой обработки задач — распечатать тысячу отчётов, обработать пачку изображений, отправить множество email:

```
publisher → work.queue → ┌─► consumer1
                         ├─► consumer2
                         └─► consumer3
```

Каждое сообщение попадает только к одному consumer. Общая throughput растёт линейно с количеством consumers до момента когда broker или подлежащие ресурсы становятся bottleneck. Простой и надёжный паттерн для распараллеливания однотипной работы.

Pub-sub паттерн через fanout exchange реализует broadcast — одно событие доставляется многим независимым подписчикам. Producer публикует в fanout exchange, каждая bound очередь получает копию сообщения, каждый consumer работает со своей независимой копией:

```
                       ┌─► queue1 → consumer1 (audit)
publisher → fanout.ex ─┼─► queue2 → consumer2 (notifications)
                       └─► queue3 → consumer3 (analytics)
```

Использование для broadcast событий вроде инвалидации кэша, распространения новостей, отправки одного события множеству отделов системы. Каждый подписчик обрабатывает сообщение независимо, добавление нового подписчика не влияет на существующих.

Topic routing использует topic exchange для маршрутизации по pattern matching. Producer публикует с иерархическим routing key. Consumers подписываются на конкретные patterns через wildcards:

```
                          ┌─ "orders.kz.new"  → kz-orders queue → kz-consumer
publisher → topic.ex ─────┼─ "orders.uz.new"  → uz-orders queue → uz-consumer
                          └─ "orders.*.new"   → all-orders queue → analytics
```

Использование для routing по бизнес-контексту. Разные consumers могут интересоваться разными подмножествами событий. Wildcards дают гибкость — analytics слушает все создания заказов независимо от страны, регион-specific consumers обрабатывают только свои.

RPC pattern через request-reply queues возможен но обычно нежелателен. Синхронное общение через асинхронный broker сложнее чем прямой HTTP. Использовать редко когда конкретная ситуация оправдывает — обычно routing через exchange для service discovery или сложная топология уже настроена.

Priority queue поддерживает приоритеты через x-max-priority аргумент. Producer устанавливает priority от 0 до максимума, broker отдаёт сообщения с высоким приоритетом первыми. Использование для срочных операций перегоняющих обычные — критическая нотификация приоритетнее рутинного audit.

Delayed messages реализуются через плагин rabbitmq-delayed-message-exchange или через TTL плюс DLX трюк. С плагином — special exchange принимающий x-delay header, публикующий сообщение в bound queue через указанное время. Без плагина — очередь-holder с TTL и DLX указывающим на целевую queue, сообщение висит TTL time и потом через DLX попадает куда нужно. Использование для retry backoff, отложенных напоминаний, scheduled tasks.

Saga pattern через choreography реализует сложные workflow через events. Каждый сервис реагирует на relevant события публикуя свои. Нет центрального дирижёра — координация decentralized:

```
OrderCreated → InventoryService → InventoryReserved →
    PaymentService → PaymentReceived → OrderConfirmed
```

При failure — компенсирующие события отменяющие проделанные шаги. InventoryReleased после PaymentFailed, OrderCancelled после InventoryReleased. Сложно моделировать и debugging но хорошо масштабируется в микросервисной архитектуре без единой точки отказа.

## Кластеризация

RabbitMQ поддерживает кластеризацию для высокой доступности. Несколько узлов образуют кластер, metadata реплицируется между всеми узлами, connections могут идти к любому узлу. Однако сами queues по умолчанию живут только на одном узле — если этот узел падает, queue становится недоступной пока не поднимется.

Classic mirroring был классическим решением для HA до RabbitMQ 3.8. Queue зеркалируется между узлами кластера — master плюс slaves. Каждое сообщение реплицируется на master и slaves. При failure master один из slaves становится новым master. Проблемы включают возможность split-brain при сетевом разделении, медленную синхронную репликацию, риск потери данных при определённых failure scenarios. Deprecated в новых версиях RabbitMQ в пользу quorum queues.

Quorum queues заменили classic mirroring как рекомендуемый подход для HA начиная с 3.8. Основаны на Raft consensus algorithm — тот же алгоритм что используется в etcd, Consul, других distributed системах. Обычно 3 или 5 replicas, quorum большинства записывает и подтверждает сообщение перед ack producer. Automatic failover без split-brain. Более надёжные и производительные чем classic mirroring в failure scenarios.

Активация quorum queue при создании через QueueBuilder метод quorum. Практическое правило — для новых очередей всегда quorum, для важных данных обязательно. Classic queues остаются для temporary или non-critical сценариев где реплицирование overhead неоправдан.

Federation и Shovel решают задачи multi-datacenter или asymmetric топологии между отдельными кластерами. Federation создаёт «мосты» между кластерами для обмена сообщениями — сообщение из exchange одного кластера автоматически передаётся в связанный exchange другого. Shovel настраивает fixed transfer из queue одного кластера в queue или exchange другого — более гибкий но менее «прозрачный».

Использование для disaster recovery когда основной DC теряется и трафик переключается на резервный. Для геораспределённых систем где локальные consumers обрабатывают локальные события с eventual синхронизацией. Для migration сценариев при переезде на новый кластер.

## Мониторинг

Комплексный мониторинг критически важен для production RabbitMQ. Основные метрики которые необходимо отслеживать разделяются на несколько категорий.

Queue metrics показывают состояние отдельных очередей. Messages ready — сколько сообщений ждёт обработки, рост означает отставание consumer. Messages unacknowledged — сколько в работе, чрезмерное значение может указывать на медленный consumer или отсутствие ack. Publish rate и consume rate — балансировка publish и consumption, если publish значительно превышает consume в течение времени queue будет расти.

Consumer metrics отслеживают активность consumers. Consumer count per queue — сколько активных consumers, значение 0 при наличии сообщений критическая проблема. Redeliver rate — частота повторных доставок, рост означает проблемы с обработкой (consumers падают или nack).

Connection metrics касаются состояния сети. Connection count — количество активных соединений, слишком высокое значение может указывать на leak. Channel count — каналы внутри соединений, аналогично. Blocked connections — producers заблокированные из-за flow control, признак memory или disk давления.

Broker resource metrics показывают состояние самого broker. Memory usage — общее использование памяти, приближение к vm_memory_high_watermark активирует flow control. Disk free — свободное место на диске, приближение к порогу активирует disk alarm. File descriptors и socket descriptors — лимиты OS которые могут стать bottleneck при большом количестве connections.

Cluster metrics для кластеризованных установок. Network partitions — split-brain индикаторы, требуют немедленного внимания. Node availability — какие узлы кластера live. Queue leader distribution — балансировка queue leaders между узлами.

Management UI на порту 15672 предоставляет визуальный интерфейс для ad-hoc мониторинга. Показывает все обсуждённые метрики в реальном времени, историю за последние периоды, детальную информацию по exchanges, queues, connections, channels. Полезно для explorаторной диагностики.

Prometheus интеграция через rabbitmq_prometheus плагин экспортирует метрики в стандартном формате. Grafana dashboards визуализируют тренды. Alerting rules настраиваются на превышение критических порогов. Стандартный setup для production observability.

Alerting должен покрывать критические сценарии. Queue depth превышает N (например 10000) в течение времени — indicates отставание consumer или другую проблему. Consumer count zero при наличии сообщений — критическая ситуация, обработка остановилась. Memory alarm active — broker в состоянии back-pressure. DLQ размер растёт — систематические ошибки обработки требуют внимания команды.

## Частые проблемы

Реальные production проблемы обычно сводятся к нескольким типичным сценариям диагностируемым по определённым симптомам.

«Сообщения теряются» — самая тревожная жалоба. Первый чеклист включает проверку durability на всех уровнях. Queue durable true? Message persistent через deliveryMode 2? Consumer использует manual ack и ack вызывается только после успешной обработки? Publisher confirms включены? Mandatory плюс returns callback для обнаружения routing проблем? В большинстве случаев проблема — один из этих пунктов не выполнен и создаёт брешь в цепочке надёжности.

«Consumer стоит, ничего не берёт» диагностируется через несколько проверок. Подписался ли consumer на очередь через basic.consume? Prefetch не равен 0 (accidentally 0 означает что broker не отправляет ничего)? Все сообщения in-flight (unacknowledged) — возможно предыдущие обработки зависли не отправив ack? Connection не упал молча — spring-rabbit обычно переподключается но иногда с ошибками? Case autoAck при exception сообщения теряются молча без alert.

«Broker медленный» может иметь несколько причин. Memory alarm означает что очереди в RAM переполняют лимит — необходимо lazy queues или увеличить watermark. Disk alarm — WAL журнал растёт быстрее чем очищается. Слишком много каналов на connection превышают channel_max лимит. Слишком много connections упираются в connection_max. Flow control замедляет producers что видно в блокировке publish operations.

Redelivery loop — классическая проблема infinite reprocessing одного сообщения. Consumer падает при обработке, сообщение возвращается в queue через requeue, снова падает, снова возвращается. Причины включают retry без max-attempts, default-requeue-rejected равный true, deserialization error который никогда не пройдёт. Лечение через max-attempts плюс RejectAndDontRequeueRecoverer отправляющий в DLX после исчерпания попыток и default-requeue-rejected false как безопасный default.

«Producer отправил но не пришло» может произойти по нескольким причинам. Publisher confirms не включены — producer не знает дошло ли. Exchange не существует или имя опечатано — сообщение отбрасывается broker. Routing key не сматчился с binding — сообщение silently потеряно (mandatory спасает от этого случая). Consumer вообще не подписан — проверить через management UI. Каждая из причин имеет свой fix, но начинать диагностику стоит с проверки publisher confirms поскольку без них многие проблемы остаются невидимыми.

Дубли сообщений появляются в предсказуемых ситуациях. Consumer не acknowledge и broker переотправил после timeout — normal behavior at-least-once semantics. Publisher retry без idempotency key на consumer стороне. Multiple bindings queue к одному exchange с overlapping patterns — сообщение попадает в очередь несколько раз. Всегда идемпотентный consumer обязательное требование.

Порядок нарушен — типично при concurrent consumers на одной очереди. Concurrency больше 1 означает parallel обработку без гарантии порядка. Nack requeue возвращает в head что нарушает порядок. Sharding по бизнес-ключу через consistent routing даёт порядок внутри shard.

## Backpressure

Что делать когда producer быстрее consumer? Классическая проблема которая требует явного решения на архитектурном уровне.

Queue растёт при дисбалансе publish и consume rates. Варианты решения включают увеличение количества consumers через scaling — новые instances consumer добавляются пока queue depth не стабилизируется. Batch processing позволяет consumer обрабатывать несколько сообщений за один цикл увеличивая throughput. Оптимизация consumer через profiling — часто bottleneck не в CPU обработки а в external calls к database или сторонним API.

Drop old policy через x-max-length и overflow drop-head активно ограничивает размер queue сбрасывая старые сообщения при переполнении. Полезно для сценариев где старые сообщения перестают быть актуальными — real-time уведомления, high frequency updates.

Reject publish policy через x-max-length и overflow reject-publish не принимает новые сообщения когда queue полна создавая backpressure на producer. Producer получает nack на publisher confirms и должен решать что делать — retry позже, буферизация локально, dropping.

Rate limiting на producer стороне — приложение сама ограничивает publish rate. Полезно для сценариев где известна максимальная capacity downstream. Проактивное ограничение предотвращает переполнение вместо реактивного разбирательства.

Ограничение по размеру через x-max-length-bytes задаёт лимит суммарного объёма сообщений в очереди. Полезно для очередей с большими сообщениями где количество не отражает реальный memory footprint.

## Безопасность

Base security практики для RabbitMQ включают несколько обязательных элементов для production установок.

Users и passwords — никогда guest/guest в production. Guest пользователь работает только с localhost по default, но всё равно опасен если случайно открыт. Отдельные пользователи per service с уникальными паролями. Password rotation при уходе персонала или подозрении на компрометацию.

Permissions per vhost ограничивают доступ пользователей. Каждый service имеет свой vhost и user только с правами на этот vhost. Read permissions ограничивают какие очереди можно читать. Write permissions какие exchange можно публиковать. Configure permissions какие сущности можно создавать/удалять.

TLS для encrypted connections через порт 5671 вместо plain 5672. Обязательно для connections через недоверенную сеть — public internet, cross-datacenter. Certificates управляются через standard PKI infrastructure. Rotation certificates перед истечением.

Sensitive data в сообщениях требует внимания. Payload обычно не encrypted внутри broker — administrator с доступом видит содержимое. Для чувствительных данных — client-side encryption перед публикацией. Alternative — ограничение доступа к broker infrastructure на operational уровне.

## HA hitrosti

Idempotent consumer это фундаментальное требование. Реализация через таблицу processed_messages с индексом на message_id и TTL cleanup старых записей. Каждый consumer перед обработкой проверяет — уже обработано? Если да — skip. Иначе — process и record. В одной транзакции чтобы избежать гонки.

Transactional outbox решает проблему атомарности database commit и message publish. Обычная последовательность — commit транзакции в БД потом publish в Rabbit — не атомарна. Если между ними падение то inconsistent state — БД имеет изменения но событие не опубликовано (или наоборот при обратном порядке).

Outbox pattern использует таблицу outbox в БД для промежуточного хранения событий. В основной транзакции — сохранить бизнес-данные плюс запись в outbox таблицу. Атомарность гарантирована транзакцией БД. Отдельный job (scheduled task или CDC listener) читает outbox → публикует в Rabbit → удаляет обработанные записи. Гарантия end-to-end — если БД коммит прошёл то outbox запись есть и рано или поздно опубликуется.

Consumer graceful shutdown критически важен для не потерять сообщения при deployment. При получении SIGTERM consumer должен прекратить принимать новые сообщения из broker, дообработать уже полученные in-flight сообщения, ack всё успешно завершённое, close channel и connection корректно. Spring AMQP делает это через SmartLifecycle интерфейс, но необходимо проверить что shutdown-timeout в OS достаточен для завершения in-flight работы. Слишком короткий timeout — kill 9 обрывает in-flight обработку и сообщения возвращаются через redelivery mechanism что приводит к дубли обработки при необидентичных consumer.

## Реальные кейсы КНП

Memory кейс knp-fo-sync-notification-bugs содержит шесть багов NotificationSyncService связанных с надёжной обработкой очередей. ProcessedDate микросекунды молча теряются из-за неправильной обработки timestamp precision в конверсии. @Transactional мёртв из-за self-invocation что даёт LazyInit exceptions при обращении к lazy полям в неисправно управляемой транзакции. Тихий skip на определённых ошибках вроде MAX offset без явного логирования проблемы. printStackTrace вместо log.error означает что ошибки не попадают в ELK и остаются невидимыми для мониторинга. Контекст на цикл создаёт memory leak. PeriodValue равный нулю обрабатывается некорректно.

Фикс через merge request 369 включал явные транзакции с правильной изоляцией, structured logging через log.error с exception, правильный retry mechanism. Общий урок — правильный consumer это несколько дисциплин соблюдаемых одновременно и одна пропущенная деталь ломает всё.

Кейс knp-fno-outer-sync-esb-dead-route демонстрирует другой класс проблем. Sync сервис для OUTER_FNO_INFO падал 1586 раз за 13 часов из-за отсутствующего SOAP маршрута. Формально не проблема RabbitMQ, но урок общий — retry на не-фиксимую проблему создаёт шум и лишние потери. Каждая retry попытка стоит ресурсов, генерирует alerts, потенциально усугубляет проблемы через дополнительную нагрузку. Правильное решение включает circuit breaker для остановки retry при systematic ошибках, feature flag через @ConditionalOnProperty для явного отключения проблемного функционала до починки, четкая escalation процедура для не-фиксимых проблем чтобы не оставлять их в noise.

Общие уроки для consumers в КНП. Все consumers должны быть идемпотентными через messageId и processed messages таблицу. Structured logging через SLF4J и log.error вместо printStackTrace. Явные транзакции без self-invocation ловушек. DLQ на все критические очереди с alerting на его рост. Метрики per queue включая rate ack, nack, redeliver.

## Итоги

RabbitMQ поддерживает множество паттернов использования от простого work queue до сложных saga workflow. Выбор паттерна определяется требованиями к routing, гарантиям доставки, ordering, масштабированию.

Quorum queues заменяют classic mirroring как рекомендуемый подход для HA. Federation и Shovel решают задачи multi-datacenter и migration сценариев.

Мониторинг критически важен для production. Queue depth, consumer count, publish/consume rate, redeliver rate, resource usage broker — основные метрики. Prometheus плюс Grafana стандартный setup. Alerting на пороговые значения обязательно.

Типичные проблемы имеют предсказуемые причины и стандартные fixes. Missing durability, missing publisher confirms, redelivery loop без max-attempts, backpressure без rate limiting — все встречаются регулярно и решаются известными подходами.

Backpressure требует явного решения — scaling, batching, TTL, max-length, rate limiting. Игнорирование ведёт к падению системы при пиковых нагрузках.

Безопасность включает users/passwords, permissions per vhost, TLS, careful handling чувствительных данных. Guest/guest в production никогда.

Идемпотентность consumer, transactional outbox, graceful shutdown — HA-хитрости отделяющие toy proof-of-concept от production-ready системы.

Реальные кейсы КНП подтверждают что теоретически правильная архитектура ломается на dozens мелких деталей — printStackTrace, self-invocation, missing DLX, retry без circuit breaker. Правильный consumer это дисциплина всех этих деталей.

Итог блока RabbitMQ — файлы 20-23 покрыли AMQP основы, гарантии доставки, Spring AMQP реализацию, production паттерны и troubleshooting. Дальше начинается блок Spring Security и Keycloak с файла 24 spring-security-basics.
