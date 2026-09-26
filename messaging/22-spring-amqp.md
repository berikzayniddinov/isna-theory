# 22. Spring AMQP: RabbitTemplate и @RabbitListener

## Экосистема Spring AMQP

Работа с RabbitMQ в Spring Boot приложении происходит через Spring AMQP — обёртку над низкоуровневой java библиотекой amqp-client предоставляющей идиоматичный Spring API. Подключение через spring-boot-starter-amqp тянет обе библиотеки плюс автоконфигурацию Boot настраивающую основные bean-ы по умолчанию.

Ключевые абстракции которые предоставляет Spring AMQP покрывают все основные потребности. RabbitTemplate для публикации сообщений с автоматической сериализацией. @RabbitListener для декларативного создания consumers через аннотации на методах. RabbitAdmin для автоматического объявления exchanges, queues, bindings при старте приложения. ConnectionFactory для управления соединениями с broker. MessageConverter для преобразования между объектами Java и byte array в сообщениях.

Spring AMQP хорошо интегрируется с существующей экосистемой Spring — dependency injection работает, конфигурация через application.yml, интеграция с Spring Boot Actuator для метрик, интеграция с Spring Retry для error handling. Это делает работу с RabbitMQ существенно продуктивнее чем прямое использование низкоуровневого API.

## Конфигурация подключения

Основная настройка подключения к RabbitMQ выполняется через spring.rabbitmq секцию в application.yml. Host, port, virtual-host, username, password — базовые параметры соединения. Connection-timeout ограничивает время ожидания установления соединения при старте приложения.

Publisher-confirm-type задаёт тип подтверждений публикации. Значение none отключает confirms, simple использует синхронное ожидание через waitForConfirms, correlated включает асинхронные подтверждения с корреляцией через callback. Для production сценариев рекомендуется correlated дающий баланс производительности и надёжности.

Publisher-returns управляет обработкой mandatory returns — сообщений которые не могут быть маршрутизированы ни в одну очередь. Значение true активирует callback на такие возвраты позволяя обнаруживать routing проблемы.

Listener настройки контролируют поведение consumers. Acknowledge-mode задаёт стратегию ack — auto для автоматического подтверждения после успешной обработки, manual для явного контроля из кода, none для отсутствия подтверждений. Prefetch устанавливает количество unacked сообщений которые consumer может держать одновременно. Concurrency и max-concurrency задают минимальное и максимальное количество параллельных consumers на очередь — начинается с минимума, при росте нагрузки увеличивается до максимума.

Retry секция настраивает автоматические повторы при exception в consumer. Max-attempts ограничивает количество попыток. Initial-interval, multiplier, max-interval задают backoff между попытками с экспоненциальным ростом. Default-requeue-rejected определяет поведение по умолчанию для rejected сообщений — при false нерасхавываемое сообщение уходит в DLX что предотвращает redelivery loop.

## RabbitTemplate для публикации

RabbitTemplate это основной класс для отправки сообщений в Spring AMQP. Использование обычно через @Autowired RabbitTemplate в компонентах где нужна публикация.

Простая публикация выполняется через convertAndSend с указанием exchange, routing key и объекта для отправки. Spring автоматически сериализует объект через настроенный MessageConverter — по умолчанию через Java serialization, но обычно настраивают Jackson для JSON. Result — сообщение с payload сериализованного объекта отправляется в указанный exchange с указанным routing key.

Публикация с дополнительными свойствами сообщения возможна через MessagePostProcessor как третий аргумент. В нём можно установить messageId для идемпотентности, expiration для TTL, custom headers для маршрутизации или метаданных, delivery mode для persistent vs transient. Гибкость постобработки позволяет тонко контролировать все аспекты сообщения не привязываясь к специфике low-level API.

Confirm callback регистрируется через setConfirmCallback на RabbitTemplate. Callback вызывается broker для каждого опубликованного сообщения — ack означает успешное принятие broker, nack говорит о проблеме. Correlation data передаваемая при публикации помогает связать confirm с оригинальным сообщением. Практическое использование — логирование ошибок публикации, retry для nack, метрики публикации.

Returns callback обрабатывает mandatory returns когда сообщение не может быть маршрутизировано. Broker возвращает сообщение через этот callback, приложение обрабатывает возврат — обычно log warning и alert поскольку routing проблема обычно указывает на конфигурационную ошибку.

Message converter определяет формат сериализации. По умолчанию SimpleMessageConverter использует Java serialization что не рекомендуется по нескольким причинам — jar скачки версий классов, безопасность, невозможность interop с не-Java системами. Jackson2JsonMessageConverter даёт JSON что стандартно, читаемо, поддерживается любой платформой. Регистрация как @Bean заменяет default converter во всём приложении.

Template-level retry настраивается через spring.rabbitmq.template.retry для повторных попыток самой публикации при исключениях. Настройки аналогичны listener retry — max-attempts, initial-interval, multiplier. Полезно для transient проблем сети или broker.

## Декларация топологии через @Bean

Объявление exchanges, queues, bindings делается через @Bean декларации в конфигурационном классе. Spring RabbitAdmin автоматически создаёт эти сущности на broker при старте приложения. Если уже существуют — проверяет совместимость аргументов и типов.

Exchange создаётся через TopicExchange, DirectExchange, FanoutExchange, HeadersExchange классы. Конструктор принимает имя, durable флаг, autoDelete флаг. Durable true означает что exchange переживёт перезапуск broker. AutoDelete обычно false для named exchanges.

Queue создаётся через QueueBuilder с fluent API. Метод durable задаёт имя и включает durability. withArgument добавляет специфические свойства — x-dead-letter-exchange для DLX, x-dead-letter-routing-key для указания как маршрутизировать в DLX, x-message-ttl для времени жизни сообщений, x-max-length для ограничения размера, x-max-priority для priority queue.

Binding связывает queue с exchange через BindingBuilder. Метод bind принимает queue, to принимает exchange, with принимает routing key или pattern для topic exchange. Один queue может быть bound к нескольким exchanges или к одному exchange с разными routing keys.

Централизованное объявление всей топологии в одном конфиг классе даёт полный контроль и явную документацию структуры. Изменение топологии видно в одном месте, code review покрывает изменения, версионирование через git легко отслеживает эволюцию. Альтернатива декларировать топологию через @RabbitListener queuesToDeclare компактнее но менее прозрачна.

## @RabbitListener для приёма

@RabbitListener это декларативный способ создания consumer через аннотацию на методе. Spring автоматически создаёт SimpleMessageListenerContainer под капотом, подписывается на указанные queues, десериализует сообщения через настроенный converter, вызывает метод для каждого сообщения.

Простой consumer выглядит как метод помеченный @RabbitListener с параметром — типом ожидаемого объекта. При получении сообщения Spring десериализует payload через converter, вызывает метод передавая объект. При successful возврате из метода Spring отправляет ack. При exception в методе — Spring запускает retry logic и в конце recovery.

Полные аргументы позволяют доступ к метаданным сообщения. @Payload аннотация явно маркирует основной payload. @Header аннотация с именем header даёт доступ к специфическим заголовкам — messageId, correlationId, custom headers. Channel параметр даёт доступ к AMQP каналу для manual ack сценариев. Message параметр даёт полный AMQP Message объект со всеми свойствами.

Concurrency контролируется на уровне application или per listener. Application уровень через spring.rabbitmq.listener.simple.concurrency и max-concurrency. Per listener через параметр concurrency на @RabbitListener в формате «min-max». Даёт гибкость — разные listeners могут иметь разные требования к параллелизму.

Manual ack режим для случаев когда нужен явный контроль над acknowledgment. Метод должен принимать Channel и delivery tag через @Header. Внутри метода — try/catch с basicAck при успехе и basicNack при ошибках с явным управлением requeue. Обычно auto ack плюс правильно настроенные retry и recovery достаточны, manual ack используется редко для специфических сценариев.

## Retry и recovery

Автоматический retry настраивается через spring.rabbitmq.listener.simple.retry секцию. При включении Spring оборачивает listener в retry advice который перехватывает exceptions и повторяет вызов согласно настройкам. Enabled true активирует retry. Max-attempts ограничивает количество попыток. Initial-interval, multiplier, max-interval задают backoff.

Механизм работы retry — при exception в listener Spring не отправляет nack сразу, вместо этого повторяет вызов после указанного интервала. Первая попытка мгновенно, вторая через initial-interval, третья через initial-interval умноженное на multiplier и так далее до max-interval. Если все попытки исчерпаны — запускается recovery.

Важное ограничение — retry держит thread consumer через Thread.sleep во время backoff. Это блокирует thread на всё время retry что уменьшает throughput доступный для других сообщений. Для медленных retry предпочтительнее retry topic pattern через выделенные queues с TTL, где thread не блокируется, а сообщение ждёт в отдельной очереди.

MessageRecoverer определяет что делать когда все retries исчерпаны. RejectAndDontRequeueRecoverer nack сообщение с requeue false что при настроенном DLX на очереди отправит его туда. Простой и рекомендуемый подход для большинства случаев. RepublishMessageRecoverer явно публикует сообщение в другой exchange с указанным routing key — даёт больше контроля над routing в failure сценариях, полезно для custom DLQ логики. Custom recoverer реализующий MessageRecoverer интерфейс позволяет arbitrary логику.

## ConnectionFactory настройка

CachingConnectionFactory это дефолтная фабрика соединений в Spring Boot. Кэширует каналы внутри одного connection что даёт хорошую производительность для большинства сценариев. Channel-cache-size контролирует размер кэша, дефолт 25.

Тонкая настройка через @Bean переопределяет defaults когда стандартной конфигурации недостаточно. Параметры включают connection timeout, channel cache size, publisher confirm type, publisher returns, request heartbeat interval для keepalive и другие AMQP специфические настройки.

PooledChannelConnectionFactory альтернатива для очень высокой нагрузки. Поддерживает pool каналов вместо простого cache что может дать лучшую производительность в сценариях с очень частой публикацией из многих threads. Реже используется — CachingConnectionFactory обычно достаточен.

## Error handler и batch listeners

ErrorHandler на container factory позволяет глобальную обработку ошибок consumer вне обычного retry/recovery flow. Регистрируется через setErrorHandler и вызывается для всех failures listener. Полезно для централизованного alerting, метрик, специфической логики error handling которую сложно выразить через recoverer.

Batch listeners поддерживают обработку сообщений пачками для очень высокой throughput. @RabbitListener с параметром List<T> вместо одного T получает пачку сообщений сразу. Настройка через container factory — setBatchListener true, setBatchSize размер пачки, setConsumerBatchEnabled true. Полезно для сценариев где отдельные сообщения обрабатываются очень быстро и overhead per-message становится significant.

Batch processing даёт лучшую производительность но усложняет error handling. Ошибка при обработке одного сообщения из пачки требует решения — reject всю пачку, попытаться обработать оставшиеся, ret only failed. Стандартные recovery mechanisms работают с пачками, но custom логика может потребоваться в зависимости от требований.

## RPC и reply-to

RabbitMQ поддерживает RPC pattern когда producer публикует запрос и синхронно ждёт ответа. Через convertSendAndReceive метод RabbitTemplate producer отправляет сообщение с reply-to header указывающим на temporary queue, ждёт ответа в этой queue, возвращает результат вызывающему коду.

Под капотом producer создаёт temporary exclusive queue для reply, устанавливает correlationId для связывания запроса и ответа, публикует запрос с reply-to header, блокируется на чтение reply queue до получения ответа или timeout. Consumer обрабатывающий запрос читает reply-to из полученного сообщения, публикует ответ в указанный queue.

Практическое использование ограниченное. RPC поверх asynchronous message broker сложнее HTTP по нескольким причинам. Latency обычно выше из-за double round-trip через broker. Ошибки сложнее обрабатывать — timeout может означать что запрос не дошёл или ответ потерян. Отсутствие явных correlation в log при debugging сложных сценариев.

Для синхронного request-response обычно лучше использовать HTTP напрямую с retry и circuit breaker. RabbitMQ RPC оправдан в специфических сценариях где нужно routing через exchange для service discovery или где инфраструктура message broker уже настроена и добавление HTTP endpoint нежелательно.

## Тестирование

TestContainers предоставляет удобный способ интеграционного тестирования с реальным RabbitMQ. RabbitMQContainer запускает docker контейнер с broker перед тестами. DynamicPropertySource динамически устанавливает spring.rabbitmq.host и port в свойства Spring Boot указывая на запущенный контейнер. Тесты работают с реальным broker получая полную интеграцию.

Преимущество TestContainers — тесты работают с реальным поведением broker включая все edge cases. Недостаток — требуется Docker, время старта контейнера, ресурсы CPU и памяти. Для быстрых unit тестов иногда используется TestRabbitTemplate из Spring AMQP который записывает вызовы для последующей проверки без реального broker.

MockMvc и подобные инструменты не подходят для тестирования Rabbit-based функциональности потому что реальное messaging поведение сложно замокировать корректно. Timing acknowledgments, ordering, redelivery, DLX — все эти aspects требуют реального broker для правильного тестирования.

## Полная production конфигурация

Собранная воедино конфигурация production ready listener и publisher включает несколько компонентов. Конфиг класс с @Bean декларациями exchange, queue, DLQ, DLX, bindings. RabbitTemplate с настроенным JSON converter, mandatory true, confirm и returns callbacks. Listener container factory с prefetch, concurrency, error handling, retry interceptor.

Ключевые элементы правильной конфигурации. Durable exchange и queue для persistence. DLX и DLQ для error isolation. Jackson JSON converter вместо Java serialization. Publisher confirms в correlated режиме. Mandatory для обнаружения routing ошибок. Prefetch согласованный с характером обработки. Concurrency диапазон с автомасштабированием. Retry с backoff и recoverer в DLX. Default-requeue-rejected false для предотвращения redelivery loop.

Publisher должен устанавливать messageId для идемпотентности consumer, deliveryMode persistent для сохранности, timestamp для отладки. Correlation data через CorrelationData объект для связывания publisher confirm с оригинальным сообщением.

Consumer должен проверять идемпотентность по messageId перед обработкой, использовать structured logging вместо printStackTrace, полагаться на автоматический retry и recovery Spring вместо ручного управления thread. При успехе Spring автоматически acknowledge, при exception автоматически retry, после exhaust автоматически отправляет в DLX.

## Реальные грабли из практики

Забытый message converter приводит к падениям при первой попытке отправить или получить сложный объект. Дефолтный SimpleMessageConverter не умеет Jackson, только Java serialization. Регистрация Jackson2JsonMessageConverter как bean и настройка на template плюс listener factory решает проблему.

Missing publisher confirms настройка означает что producer не знает дошли ли сообщения до broker. Publisher-confirm-type none даёт fire and forget поведение. Настройка correlated даёт async подтверждения. Забыть настроить может привести к silent потерям при broker failures.

Default-requeue-rejected true создаёт infinite loop при не-фиксимой ошибке. Сообщение возвращается в queue, снова падает, снова возвращается. Настройка false плюс DLX останавливает loop и изолирует проблему.

Ordering ожидания при multiple consumers приводят к misunderstandings. Concurrency больше 1 плюс prefetch больше 1 означает parallel обработку без гарантии порядка. Если порядок важен — concurrency 1 или sharding по бизнес ключу.

Missing idempotency в consumer приводит к дублированию side effects при redelivery. Каждый consumer должен либо использовать идемпотентные операции либо проверять messageId в processed messages таблице. При отсутствии идемпотентности любое падение может привести к дубли side effect включая финансовые операции или уведомления.

## Итоги

Spring AMQP предоставляет высокоуровневый API для работы с RabbitMQ. RabbitTemplate для публикации, @RabbitListener для приёма, автоматическая конфигурация exchange/queue/binding через RabbitAdmin, интеграция с dependency injection и другими Spring возможностями.

Publisher confirms в correlated режиме с mandatory true обеспечивают надёжность publish. Jackson JSON converter стандартный выбор для сериализации. Correlation data связывает confirm с оригинальным сообщением.

@RabbitListener с автоматическим retry и recovery покрывает большинство сценариев. Manual ack режим для специфических случаев. Concurrency и prefetch настраиваются согласно характеру обработки. Batch listeners для высокой throughput с независимыми сообщениями.

RetryInterceptor настраивает автоматические повторы с backoff. MessageRecoverer определяет действия после исчерпания retries. RejectAndDontRequeue плюс DLX стандартный подход для error isolation.

Тестирование через TestContainers с реальным RabbitMQ даёт полную integration. Mock подходы обычно недостаточны для правильного тестирования message flow.

Production конфигурация требует комбинации всех элементов — durable topology, JSON converter, publisher confirms, prefetch, concurrency, retry, DLX, идемпотентность consumer. Каждый элемент решает свою часть проблемы, отсутствие любого создаёт слабое звено в цепочке надёжности.

Дальше рассматриваются паттерны использования RabbitMQ в production, кластеризация, мониторинг и troubleshooting типичных проблем.
