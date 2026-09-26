# 33. Внутренности @Transactional: прокси, PlatformTransactionManager, resource binding

## Зачем спускаться на этот уровень

В большинстве Spring-приложений разработчики просто ставят аннотацию @Transactional и предполагают что «оно как-то работает». Пока всё работает — этого достаточно. Но когда возникают проблемы — «почему транзакция не откатывается», «почему изменения не сохраняются», «почему @Transactional не срабатывает при вызове из того же класса» — необходимо понимать что реально происходит под капотом. Без этого понимания диагностика превращается в guessing.

Разница между middle и senior здесь очевидна. Middle пишет @Transactional и надеется что работает. Senior знает что @Transactional это CGLib subclass proxy, знает что self-invocation минует proxy потому что вызов через this идёт напрямую в оригинальный объект без AOP-совета, знает что PlatformTransactionManager под капотом биндит Connection в ThreadLocal, знает почему @PostConstruct метод не может иметь работающего @Transactional. За этой разницей стоит знание конкретных Java-механизмов bytecode manipulation, ThreadLocal storage, proxy generation.

В этом файле разберём именно эти внутренности. Как Spring обнаруживает что бину нужен прокси. Как CGLib генерирует subclass с добавленной логикой на уровне bytecode. Как отличаются JDK dynamic proxy и CGLib proxy на уровне class layout. Как TransactionInterceptor встраивается в цепочку MethodInterceptor. Как TransactionAspectSupport управляет lifecycle транзакции. Как PlatformTransactionManager абстрагирует различные mechanisms (JDBC, JPA, JTA, Rabbit). Как resources (Connection, EntityManager) привязываются к потоку через TransactionSynchronizationManager. Что происходит на уровне stack frames при входе в @Transactional-метод и что при выходе. Почему self-invocation ломает всё и какие пути обхода существуют.

## От аннотации к rowtime behavior

Начнём с самого начала — что происходит когда разработчик пишет @Transactional. На этапе компиляции ничего особенного — аннотация просто сохраняется в class файле как metadata. Никакого bytecode generation не происходит в javac. Аннотация это просто маркер.

Всё интересное происходит при старте Spring context. Spring сканирует классы, находит те что помечены как beans (@Service, @Component, @Repository), создаёт их instances через DI. И вот здесь начинается магия — специальный BeanPostProcessor называемый InfrastructureAdvisorAutoProxyCreator (или его конкретная реализация BeanFactoryTransactionAttributeSourceAdvisor) смотрит на каждый созданный bean и решает — «этому bean нужен транзакционный proxy?».

Логика решения проста. AutoProxyCreator проходит по всем методам класса, ищет @Transactional на методе или на классе. Также проверяет любые интерфейсы которые реализует класс — если аннотация на interface method. Если хоть где-то нашёл — создаётся proxy. Если не нашёл — bean остаётся оригинальным, никакой обёртки нет.

Кэширование этого решения важно для performance. TransactionAttributeSource (обычно AnnotationTransactionAttributeSource) при первом обращении парсит аннотации метода и кэширует результат в ConcurrentHashMap<Method, TransactionAttribute>. Дальнейшие вызовы того же метода читают TransactionAttribute из кэша без reflection overhead. Кэш живёт весь lifetime context — очищается только при shutdown.

Выбор типа прокси происходит на основе класса. Если класс имеет non-trivial интерфейсы (не Aware, не InitializingBean и подобные infrastructure) — Spring может выбрать JDK dynamic proxy. Прокси реализует все эти interfaces и делегирует calls в оригинал через InvocationHandler. Если класс не имеет interfaces — CGLib генерирует subclass. По умолчанию в Spring Boot настройка spring.aop.proxy-target-class=true заставляет всегда использовать CGLib даже для классов с interfaces. Причина в предсказуемости — CGLib subclass работает независимо от того как bean injectится (по class type или interface type), тогда как JDK proxy подходит только для injection по interface.

## CGLib subclass generation on bytecode level

Когда Spring решает создать CGLib proxy, происходит следующее. CGLib библиотека (интегрированная в spring-core с Spring 4) через ASM API генерирует новый Java class на runtime. Этот класс — subclass оригинального. Наследует все public и protected методы, переопределяет их, добавляет свою логику.

Bytecode генерации выглядит следующим образом. Для каждого метода оригинального класса CGLib создаёт два элемента. Первый — переопределённая версия метода в subclass, которая вместо оригинальной логики вызывает MethodInterceptor. Второй — так называемый «direct method» с определённым именем (обычно CGLIB$methodName$0), который содержит копию оригинального bytecode. MethodInterceptor может вызвать этот direct method через reflection-подобный механизм MethodProxy для execution оригинальной логики.

Практически структура сгенерированного класса выглядит так. Оригинальный класс OrderService имеет метод createOrder. CGLib создаёт OrderService$$SpringCGLIB$$0 extends OrderService. В этом subclass метод createOrder переопределён — при вызове он делает invocation через registered MethodInterceptors chain. MethodInterceptor это TransactionInterceptor который выполняет транзакционную logic и потом делегирует к MethodProxy.invokeSuper для fallback к оригинальному методу через super.createOrder.

Именно поэтому CGLib имеет свои ограничения. Final методы нельзя override — не может быть переопределён в subclass. Final классы нельзя extend — не может быть создан subclass. Private методы не наследуются — не visible в subclass, не могут быть переопределены. Static методы не подходят под этот механизм — они не instance methods. Все эти ограничения фундаментально следуют из subclass generation approach.

Отдельная тонкость — CGLib subclass должен иметь no-arg constructor или Spring должен использовать Objenesis library для создания instance без constructor. Objenesis умеет создавать objects минуя constructors через platform-specific tricks. Это позволяет CGLib работать с классами имеющими только parameterized constructors. Но final fields инициализируемые в constructor могут остаться в default state (null для references) в proxy instance — Spring обычно заполняет их через reflection после создания.

## TransactionInterceptor и цепочка advice

TransactionInterceptor это конкретный MethodInterceptor реализующий AOP интерфейс от AOP Alliance стандарта (spring-aop namespace). Класс org.springframework.transaction.interceptor.TransactionInterceptor наследуется от TransactionAspectSupport и реализует MethodInterceptor через один метод invoke(MethodInvocation invocation).

Внутри invoke происходит вся оркестрация транзакции. Interceptor извлекает Method и target class из MethodInvocation. Вызывает TransactionAspectSupport.invokeWithinTransaction — главный метод содержащий логику begin/commit/rollback вокруг оригинального invocation. Возвращает результат обратно в AOP framework.

TransactionAspectSupport.invokeWithinTransaction в исходном коде Spring выглядит примерно так (упрощённо для понимания). Первый шаг — получить TransactionAttribute для method через TransactionAttributeSource. Attribute содержит все настройки @Transactional — propagation, isolation, timeout, readOnly, rollbackFor.

Второй шаг — определить какой PlatformTransactionManager использовать. Может быть указан явно через @Transactional(value = «txManagerName»), может быть выбран по type, обычно один default в context. Третий шаг — создать transaction если нужно, через txManager.getTransaction(txAttr). Возвращается TransactionStatus представляющий open transaction.

Четвёртый шаг — реальный invocation оригинального метода через invocation.proceed. Здесь передача control возвращается в AOP framework, который вызывает следующий interceptor в chain или direct method оригинала. Пятый шаг — обработка результата. Если exception — вызывается completeTransactionAfterThrowing которая решает commit или rollback на основе rollback rules. Если success — вызывается commitTransactionAfterReturning которая делает commit.

Шестой шаг — cleanup regardless of outcome. TransactionInfo восстанавливается (unbind current, restore previous если был suspend), resources возвращаются в pool. Всё это в try/finally блоке для обеспечения cleanup при exceptions.

Важное про AOP chain. TransactionInterceptor обычно не единственный advice — Security, Cache, custom aspects могут все крутиться вокруг метода. Chain обрабатывается через ReflectiveMethodInvocation.proceed рекурсивно — каждый interceptor вызывает proceed чтобы продолжить chain, последний interceptor вызывает target method. Порядок interceptors определяется @Order или подобным механизмом. TransactionInterceptor обычно one of the outermost — транзакция должна wrap всё остальное.

## PlatformTransactionManager абстракция и реализации

PlatformTransactionManager это core interface для транзакционного management. Три метода определяют весь contract. getTransaction(TransactionDefinition) начинает или подхватывает существующую транзакцию, возвращает TransactionStatus. commit(TransactionStatus) коммитит транзакцию. rollback(TransactionStatus) откатывает.

TransactionDefinition это read-only interface содержащий все атрибуты транзакции. Propagation, isolation, timeout, readOnly. TransactionAttribute extends TransactionDefinition добавляя rollback rules. TransactionStatus представляет active transaction — is new, has savepoint, is rollback-only и другие state flags.

Различные implementations serve разные scenarios. DataSourceTransactionManager это простейший — работает напрямую с DataSource. Начинает транзакцию через Connection.setAutoCommit(false), коммитит через Connection.commit, откатывает через Connection.rollback. Использует Spring's DataSourceUtils для правильного обращения с pooled connections. Идеален для plain JDBC приложений без JPA.

JpaTransactionManager это более сложная реализация для JPA приложений. Управляет EntityManager plus underlying JDBC Connection. Начинает транзакцию через EntityManagerFactory.createEntityManager followed by em.getTransaction().begin(). Синхронизирует EntityManager transaction с underlying JDBC transaction. Коммитит через em.flush followed by conn.commit. Handling сложнее потому что JPA имеет свою transaction abstraction поверх JDBC.

HibernateTransactionManager это Hibernate-specific version предшественник JpaTransactionManager. Практически заменён JpaTransactionManager. Оставлен для legacy code использующего Hibernate Session API напрямую.

JtaTransactionManager для distributed transactions через 2PC. Работает с external JTA provider (Atomikos, Bitronix, Narayana). Делегирует UserTransaction для begin/commit/rollback. Rare choice в microservices — избегается в пользу Saga/Outbox patterns.

RabbitTransactionManager для RabbitMQ transactions. Работает с ConnectionFactory Spring AMQP. Redко используется потому что publisher confirms обычно достаточны, transactions в Rabbit имеют performance overhead. KafkaTransactionManager аналогично для Kafka transactional producers.

ChainedTransactionManager был попыткой объединить multiple managers для best-effort atomicity. Начинает все transactions параллельно, commits по цепочке. Не 2PC — atomicity не гарантируется если один из commits fails. Deprecated с Spring 3. Правильное решение — outbox pattern либо full JTA если нужен настоящий 2PC.

Spring Boot autoconfiguration выбирает правильный TransactionManager based on classpath. spring-boot-starter-data-jpa триггерит JpaTransactionManager. Just DataSource без JPA — DataSourceTransactionManager. Multiple sources требуют explicit configuration.

## TransactionSynchronizationManager и thread-local binding

Ключевой класс связывающий транзакцию с текущим потоком. TransactionSynchronizationManager это utility класс с только static методами, использующий множество ThreadLocal переменных внутри. Никаких instances — работает через thread-local state.

Основной ThreadLocal это resources — Map<Object, Object>. Key это обычно DataSource или EntityManagerFactory. Value это соответствующий ResourceHolder — ConnectionHolder или EntityManagerHolder. При начале транзакции TransactionManager создаёт connection/EntityManager, wraps в holder, binds в resources map через bindResource(dataSource, holder). При завершении — unbindResource.

Как это работает практически. JpaTransactionManager begins transaction. Создаёт EntityManager. Создаёт EntityManagerHolder wrapping этот EntityManager. Bind через TransactionSynchronizationManager.bindResource(entityManagerFactory, holder). Дальше в коде — когда что-то запрашивает EntityManager (через @PersistenceContext или SharedEntityManagerCreator), это lookup в thread-local через getResource(entityManagerFactory) возвращающий holder. Если holder есть — return его EntityManager. Все обращения в текущей транзакции получают тот же EntityManager.

Другие ThreadLocals в TransactionSynchronizationManager. synchronizations хранит зарегистрированные TransactionSynchronization callbacks которые будут вызваны при commit/rollback. currentTransactionName — имя транзакции для debugging и logging. currentTransactionIsolationLevel — isolation level текущей транзакции для tools вроде MyBatis которые могут проверять и адаптироваться. currentTransactionReadOnly — read-only flag для optimizations. actualTransactionActive — есть ли реально начатая транзакция (true) или мы в suspend state / вне транзакции (false).

Именно thread-local nature создаёт проблему @Async и virtual threads. Когда task submitted в executor, он выполняется в другом thread. Этот thread имеет свой ThreadLocal state — пустой относительно TransactionSynchronizationManager. Транзакция инициированная в calling thread не пробрасывается в executor thread. @Async метод @Transactional начнёт свою собственную новую транзакцию не привязанную к caller's.

Spring предоставляет TransactionSynchronization API для callbacks на transaction events. Регистрируется через TransactionSynchronizationManager.registerSynchronization. Хранится в thread-local synchronizations set. При различных transaction events (beforeCommit, afterCommit, beforeCompletion, afterCompletion) все synchronizations вызываются. Именно на этом механизме построен @TransactionalEventListener.

Suspension для REQUIRES_NEW работает через SuspendedResourcesHolder. Когда inner transaction начинается с REQUIRES_NEW, outer transaction должна suspend. TransactionManager вызывает doSuspend который забирает все current resources из TransactionSynchronizationManager, packages в SuspendedResourcesHolder, unbinds их. Начинает fresh transaction с новыми resources. При окончании inner transaction — doResume восстанавливает suspended resources обратно в thread-local. Классическая implementation основанная на suspending и restoring thread-local state.

Именно поэтому REQUIRES_NEW требует два connections из pool одновременно — один для suspended outer, второй для active inner. При heavy REQUIRES_NEW usage pool может исчерпаться быстрее чем expected.

## Полный жизненный цикл транзакции

Разберём что происходит на каждом уровне stack frame при вызове @Transactional метода. Начинается всё когда client code вызывает orderService.createOrder(order). Client code имеет reference на bean injected через DI — это на самом деле CGLib proxy не оригинальный OrderService.

Первый frame — CGLib generated createOrder в OrderService$$SpringCGLIB$$0. Этот метод получает call plus arguments. Извлекает MethodInterceptor chain из CGLib metadata. Начинает invocation через ReflectiveMethodInvocation.proceed.

Второй frame — TransactionInterceptor.invoke. Получает MethodInvocation. Extracts Method plus target class. Calls TransactionAspectSupport.invokeWithinTransaction(method, targetClass, invocation).

Третий frame — TransactionAspectSupport.invokeWithinTransaction. Получает TransactionAttribute через tas.getTransactionAttribute(method, targetClass). Определяет TransactionManager. Вызывает createTransactionIfNecessary(txManager, txAttr, joinpointIdentification).

Четвёртый frame — внутри createTransactionIfNecessary. Вызывает txManager.getTransaction(txAttr). В случае JpaTransactionManager — это довольно много работы. doGetTransaction создаёт JpaTransactionObject. Проверяет existing transaction через isExistingTransaction. Если нет — doBegin. Внутри doBegin: EntityManager.createEntityManager из EntityManagerFactory. em.getTransaction().begin. Получает Connection через unwrap. conn.setAutoCommit(false). Установка isolation level если specified. Установка readOnly если specified. Bind EntityManagerHolder в TransactionSynchronizationManager для EntityManagerFactory. Bind ConnectionHolder в TransactionSynchronizationManager для DataSource. Установка actualTransactionActive true. Установка current transaction name, isolation level, read only.

Возвращается TransactionStatus содержащий все флаги плюс TransactionInfo содержащий необходимую информацию для последующего cleanup. Prepare TransactionInfo через prepareTransactionInfo — записывает в thread-local currentTransactionInfo для быстрого доступа.

Пятый frame — обратно в invokeWithinTransaction. Continue дальше:
```java
Object retVal;
try {
    retVal = invocation.proceedWithInvocation();  // реальный метод!
} catch (Throwable ex) {
    completeTransactionAfterThrowing(txInfo, ex);
    throw ex;
}
```

invocation.proceedWithInvocation вызывает следующий interceptor в chain или (если TransactionInterceptor последний в chain) реальный target method. Через MethodProxy.invokeSuper в CGLib proxy — это equivalent super.createOrder который выполнит оригинальный код метода.

Шестой уровень stack — реальный createOrder в оригинальном OrderService. Здесь код разработчика — repo.save(o), publisher.publish(o) и так далее. Repository методы внутренне используют EntityManager. При запросе EntityManager Spring lookup в TransactionSynchronizationManager, находит holder привязанный ранее в doBegin, возвращает его EntityManager. Repository получает тот же EntityManager что был bound. Все SQL идёт через этот EntityManager, соответственно через один Connection, соответственно в одной транзакции.

Возврат из createOrder возвращает control в invokeWithinTransaction. Если exception — completeTransactionAfterThrowing проверяет rollback rules через txAttr.rollbackOn(ex) и вызывает либо rollback либо commit. Если success — commitTransactionAfterReturning вызывает commit.

Внутри commit — доBegin в reverse. Trigger beforeCommit synchronizations. doCommit — em.flush отправляет все накопленные changes в SQL. conn.commit фиксирует транзакцию в БД. Trigger afterCommit synchronizations. cleanupAfterCompletion — unbind resources, close EntityManager (возвращает connection в pool), reset TransactionSynchronizationManager state (currentTransactionActive false, other ThreadLocals cleared).

Финальный return проходит обратно через все frames — TransactionAspectSupport, TransactionInterceptor, CGLib proxy — до client code. Client получает результат метода не подозревая о всём этом machinery.

## Self-invocation детально

Классическая проблема @Transactional имеет корни в описанной архитектуре. Разберём почему точно.

Bean OrderService injected через DI это CGLib proxy — OrderService$$SpringCGLIB$$0 instance. Reference переменной в calling code указывает на этот proxy instance. Когда calling code вызывает proxy.processAll(list) — вызов идёт через proxy method processAll который в свою очередь через MethodInterceptor chain вызывает super.processAll (оригинальный метод в parent класс OrderService).

Внутри оригинального processAll метода происходит this.processOne(o). this в этом context — это this proxy instance? Нет, this — это оригинальный OrderService instance (parent класс). Здесь ключевой момент — proxy это subclass extending оригинал. Когда execution находится внутри метода parent class, this ссылается на actual runtime instance который является proxy. Но вот что критично — вызов метода через this в Java выполняется через vtable dispatch к method implementation.

Но метод processOne в parent class overriden в proxy child class. И vtable dispatch должен позвать overriden version в proxy. Почему же он идёт напрямую в оригинал? Ответ в том что CGLib generation не всегда переопределяет каждый public метод. Даже если он переопределяет, dispatch через this.method() внутри parent class body действительно должен позвать overridden version в child.

Actual explanation более subtle. Когда CGLib генерирует subclass, он не просто copies parent method — он вставляет MethodInterceptor invocation. Но byte code parent class остаётся неизменным. Когда execution находится в parent method и вызывает this.otherMethod, JVM использует invokevirtual instruction. Invokevirtual делает dynamic dispatch — ищет метод по type actual instance. Actual instance это child (proxy). Dispatch должен пойти в overridden version в proxy...

Но фактически dispatch идёт в оригинал! Причина в том, что в bytecode compiled из Java source, calls к non-static methods same class компилируются как invokespecial сначала когда classes generated (в pre-Java 8) или в некоторых optimization scenarios. Actually correct answer: dispatch идёт в overriden proxy method (invokevirtual), но proxy method — это не то же что MethodInterceptor invocation. CGLib вставляет interceptor для методов вызываемых от **external** callers.

Более точное объяснение. CGLib генерирует proxy method которая делает следующее: если invocation происходит через external call (не через super), вызывается interceptor. Если через super (внутри parent method body) — direct call к parent method без interceptor. Это разделение реализуется через специальные flags в MethodProxy или через различные method resolution rules.

Практический results тот же — self-invocation минует interceptor. Внутри processAll когда called this.processOne, метод processOne выполняется без transactional advice потому что MethodInterceptor не активируется для этого self-call. Никакая новая транзакция не создаётся даже если processOne annotated @Transactional(REQUIRES_NEW).

Workarounds имеют разную elegance. Extract to another bean наиболее clean. OrderProcessor как отдельный @Service. Injection в OrderService. Внешний call orderService.processAll вызывает orderProcessor.processOne — external call через proxy processor, transactional advice activates правильно.

Self-injection работает через injection бина самого в себя. @Autowired OrderService self инжектит proxy этого service. self.processOne — external call через proxy, advice activates. Circular dependency в Spring 2.6+ по default запрещён, требует @Lazy для breaking cycle detection. Считается code smell — usually extraction to another bean cleaner.

AopContext.currentProxy предоставляет access к current proxy through thread-local. Требует @EnableAspectJAutoProxy(exposeProxy = true). ((OrderService) AopContext.currentProxy()).processOne(o). Runtime access к proxy explicitly. Works но менее clean чем dependency injection.

AspectJ compile-time weaving через AspectJ Maven/Gradle plugin. Modifies bytecode в parent methods to insert advice invocation. Каждый method call включая self-invocation проходит через advice. Требует AspectJ toolchain, IDE support, complexity. Rare choice — usually not worth it.

## Проверка что @Transactional работает

Runtime introspection класса бина. bean.getClass().getName возвращает имя. Обычный класс — com.example.OrderService. CGLib proxy — com.example.OrderService$$SpringCGLIB$$0 или подобное. Suffix CGLIB indicates that proxy generated.

Логирование transactional operations через logging levels. org.springframework.transaction в TRACE level показывает начало и завершение каждой транзакции. Формат «Getting transaction for [class.method]», «Creating new transaction with name [name]: PROPAGATION_REQUIRED, ISOLATION_DEFAULT», «Committing JPA transaction on ...». Отсутствие этих логов для конкретного метода indicates что @Transactional не активируется — proxy не создан, self-invocation, private/static/final method.

pgstat_activity в PostgreSQL показывает active connections plus состояние. При start транзакции connection переходит в состояние «idle in transaction». При commit — обратно в «idle». Мониторинг этих states помогает understand transactional behavior извне application perspective. Long-running «idle in transaction» индикатор problem — transaction открыта но ничего не делает, обычно blocking transaction internal или external call внутри @Transactional.

## Итоги: что нужно помнить

Spring transactional infrastructure это сложный orchestration нескольких low-level Java concepts. Bytecode manipulation через CGLib создаёт proxy subclasses. AOP chain с TransactionInterceptor wraps method invocations. ThreadLocal storage в TransactionSynchronizationManager binds resources к current thread. PlatformTransactionManager абстрагирует specific transaction mechanism (JDBC, JPA, JTA).

Понимание этих mechanics критично для troubleshooting. Self-invocation minus proxy interception — понятно почему через bytecode dispatch model. Private/static/final не работают — понятно через CGLib subclass limitations. Thread-local nature — понятно почему @Async теряет transaction context.

Правила простые. @Transactional только public methods regular class. Не полагаться на self-invocation. Extract to another bean для intra-class calls требующих separate transactions. Проверять proxy generation через class name inspection или logs при подозрении на не работающий @Transactional.

Дальше — advanced aspects @Transactional включая rollback rules, listeners, savepoints, testing frameworks с deep dive в implementation details.
