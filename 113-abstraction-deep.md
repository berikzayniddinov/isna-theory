# 113. Абстракция в программировании: что, зачем, когда не надо

## Зачем это знать

Абстракция — центральное понятие программирования, но одно из самых расплывчатых. Джуниоры пишут «слишком конкретно» — методы по 200 строк, повторяющаяся логика в трёх местах, изменения бизнес-правил требуют менять пять файлов. Мидлы часто перегибают в другую сторону — интерфейсы для всего, четыре слоя абстракций поверх простого CRUD, `AbstractGenericBaseFactoryStrategy<T extends BaseEntity>` где хватило бы одного метода. Обе крайности одинаково болезненны в поддержке, только по-разному.

Разница между «делаю абстракции» и «понимаю абстракцию» — это способность отвечать на вопрос **зачем именно эта абстракция в этом месте**. Хорошая абстракция скрывает то, что реально должно быть скрыто (чтобы одно изменилось без остального), и открывает то, что реально должно быть открыто. Плохая — либо утекает (клиенты знают детали реализации), либо неправильно проведена (границы не совпадают с реальными осями изменений), либо преждевременна (сделана «на всякий случай» до того как стало известно что реально нужно варьировать).

Разберём: базу — что такое абстракция фундаментально (сокрытие деталей + выделение существенного), уровни абстракции в программировании от машинных инструкций до бизнес-домена. Ключевые понятия — encapsulation, information hiding по Парнасу, ADT (abstract data types). Стоимость абстракции — indirection, cognitive load, performance overhead. Что такое **правильная** и **неправильная** абстракция — leaky abstractions Joel Spolsky, wrong abstraction Sandi Metz («duplication is far cheaper than the wrong abstraction»). Anti-patterns: over-abstraction, premature abstraction, God interface. SOLID как принципы вокруг абстракции — SRP, OCP, LSP, ISP, DIP разобранные детально с примерами. Паттерны абстракции: interface / abstract class, template method, strategy, adapter, bridge, layers, ports & adapters (hexagonal), repository. Domain-driven abstraction — bounded context, aggregate. Практические правила: когда абстрагировать, когда inlining лучше. Специфика в Java: extends vs implements, sealed classes для controlled abstraction, records для immutable value abstractions.

Facade и Proxy разобраны в 85. Design patterns Facade/Proxy подробно там же. Здесь — концепция абстракции как таковая, шире одного паттерна.

## Что такое абстракция

Абстракция — процесс **выделения существенного и сокрытия несущественного** для конкретной задачи. Оба движения важны, и оба должны быть осознанными.

**Выделение существенного** — определить что важно для потребителя абстракции. `List<T>` из Java Collections важно: `add`, `get(i)`, `size`, `iterator`. Неважно (клиент не должен знать): под капотом это ArrayList или LinkedList, какая начальная емкость, как реализована реаллокация массива.

**Сокрытие несущественного** — детали реализации не должны просачиваться в интерфейс. Клиент кода видит только то, что ему нужно. Может писать код против абстракции, не зная реализации. Реализация может меняться (замена ArrayList на ArrayDeque для performance) без переписывания клиентов.

Классическое определение Парнаса (1972, «On the Criteria to Be Used in Decomposing Systems into Modules»): **information hiding** — модули должны скрывать design decisions, которые могут поменяться. Разделение на модули по осям потенциальных изменений, а не по функциональной декомпозиции. Это фундаментальный труд, поменявший как мы думаем про архитектуру ПО.

## Уровни абстракции

Программирование — вложенные слои абстракций. Каждый слой скрывает нижний.

**Транзисторы** — базовый уровень. Не думаем.

**Логические вентили** (AND, OR, NOT) — на уровне hardware дизайна.

**Машинные инструкции** — CPU понимает opcodes. `MOV`, `ADD`, `JMP`. Инженер обычно не думает.

**Assembler** — mnemonics для машинных инструкций. Реже сейчас.

**C / системный язык** — переменные, функции, указатели. Абстракция над регистрами и адресами памяти.

**Java bytecode** — виртуальная машина. Абстракция над CPU (одинаковый bytecode работает на x86 и ARM).

**Java язык** — классы, объекты, generic'и. Абстракция над bytecode + type system.

**JDK библиотеки** — Collections, IO, concurrency. Абстракция над примитивами языка.

**Фреймворки (Spring, Hibernate)** — dependency injection, ORM, MVC. Абстракция над JDK для конкретных задач.

**Бизнес-домен** — Order, Payment, Customer. Абстракция над техническими деталями для описания бизнес-логики.

Хороший код обычно работает **на одном уровне абстракции за раз**. Метод не должен смешивать бизнес-логику (`if order.isValid()`) с низкоуровневыми деталями (`new FileInputStream("config.xml")`). Смешивание — признак утечки уровня абстракции.

## Инструменты абстракции в языках

Разные механизмы, каждый со своим смыслом.

**Функция (метод)** — базовая абстракция: даёт имя последовательности действий, скрывает шаги. Клиент вызывает `sortAscending(list)`, не знает используется ли quicksort или mergesort.

**Класс** — абстракция состояния + поведения. Инкапсулирует поля через private + методы. Клиент видит методы, не поля.

**Interface** — абстракция **чистого** контракта без реализации. Множественная имплементация, полиморфизм. `List` — не знает реализации, только контракт.

**Abstract class** — частичная абстракция: часть готова (общие методы), часть abstract (subclass реализует). Template Method паттерн.

**Module (package)** — абстракция группы связанных сущностей с контролем видимости. Java `module` (JPMS с Java 9), packages с package-private.

**Generic type** — параметризованная абстракция. `List<T>` — «список чего-угодно», конкретный тип задаётся клиентом. Type erasure в Java (в отличие от reified generics в C#) — компромисс совместимости.

**Sealed class / interface** (Java 17+) — контролируемая абстракция: явный список permitted subclasses. Компилятор знает всех implementors, exhaustive pattern matching возможен.

**Record** (Java 16+) — immutable value abstraction. Автоматические accessors, equals, hashCode. Правильная абстракция для DTO / value objects.

## Абстрактные типы данных (ADT)

Теоретическая база — **Abstract Data Type**. Определяется через:

- **Операции** — что можно делать с типом. Для Stack: push, pop, peek, isEmpty.
- **Аксиомы** — как операции взаимодействуют. Для Stack: `pop(push(s, x)) = s`, `peek(push(s, x)) = x`.
- **Никакой реализации** — не важно как хранится (array, linked list). Важно только что операции ведут себя согласно аксиомам.

Java пример: `java.util.Stack` (deprecated, лучше `Deque`) — ADT. Клиент использует push/pop, не знает реализации. Замена ArrayList → LinkedList в основе не должна ломать клиентов если ADT сохранён.

ADT vs class: класс — реализация ADT. `ArrayList` и `LinkedList` — две реализации ADT `List`.

## Cost of abstraction

Абстракция не бесплатна. Три класса стоимости:

**Runtime overhead**. Каждый уровень abstraction добавляет indirection: виртуальный вызов вместо прямого, лишний object allocation, дополнительный stack frame. HotSpot JIT (файл 112) многое устраняет через inlining — но не всегда получается (megamorphic call sites, native methods, синхронизированные блоки).

Пример. `stream.filter(...).map(...).collect(...)` — 3 lambda + 3 wrapper объекта + iterator overhead. Против классического `for` цикла в 5-10 раз медленнее в micro-benchmark. В большинстве случаев не важно (I/O dominates), но для tight loops в hot path — заметно.

**Cognitive load**. Каждый уровень — дополнительная концепция для чтения кода. Читая метод, программист должен помнить: что делает метод, что делают вызываемые методы, что скрыто в интерфейсе, где реализация. Слишком много слоёв — невозможно удержать в голове.

Правило: если чтобы понять простую вещь надо развернуть 5 уровней indirection — абстракция плохая. Хороший код читается сверху вниз, каждый уровень объясняет следующий.

**Rigidity через wrong abstraction**. Если абстракция проведена неправильно — она не только не помогает, а активно мешает. Изменения бизнес-правил требуют менять и абстракцию, и все реализации, и часто клиентов тоже. Хуже чем прямой код без абстракции.

## Хорошая vs плохая абстракция

**Leaky Abstractions** (Joel Spolsky, «The Law of Leaky Abstractions», 2002). Все нетривиальные абстракции в какой-то степени leaky: детали реализации просачиваются наверх, клиенты вынуждены о них знать.

Классические примеры:

- TCP скрывает потерю пакетов, но latency медленных сетей просачивается — клиент чувствует.
- ORM (Hibernate) скрывает SQL, но N+1 problem (файл 104), lazy loading exceptions, batch fetch tuning — детали реализации, о которых нужно знать.
- HTTP клиент абстрагирует сеть, но connection timeouts, DNS resolution, TLS handshake — просачиваются как ошибки клиенту.
- File abstraction скрывает диск, но disk full, slow I/O, network file system latency — клиент чувствует.

Вывод Spolsky: **абстракции экономят время, но не время на обучение**. Чтобы эффективно работать поверх абстракции, всё равно нужно понимать что внутри — на случай когда абстракция потечёт.

**Wrong Abstraction** (Sandi Metz, «The Wrong Abstraction», 2016). Более коварная проблема. Абстракция сделана «правильно» с точки зрения дизайна, но границы проведены **не по тем осям**, по которым реально меняется код. Каждое новое требование требует растягивать абстракцию, добавлять параметры, if'ы, специальные случаи.

Metz формулирует: **duplication is far cheaper than the wrong abstraction**. Три одинаковых куска кода — проблема, но небольшая (можно найти, объединить когда паттерн станет очевидным). Wrong abstraction — большая проблема: она навязывает форму, которая не подходит новым случаям, и разобрать её обратно тяжелее чем начать с дублирования.

Recipe от Metz — **правило трёх**:
1. Пишешь код первый раз — просто конкретно.
2. Второй раз — copy-paste с изменениями. Терпи дублирование.
3. Третий раз — теперь видно паттерн, можно выделить общее.

Три случая дают достаточно информации о реальных осях изменений. Один-два — гадание на кофейной гуще.

## Anti-patterns абстракции

**Over-abstraction / architecture astronaut**. Каждая простая вещь обёрнута в интерфейс + factory + provider + strategy. `UserService`, `UserServiceImpl`, `UserServiceFactory`, `IUserServiceFactory`. Часто в enterprise Java, особенно с уклоном на «на всякий случай, вдруг понадобится другая реализация». Обычно другая реализация не появляется никогда.

Правило: **интерфейс имеет смысл если есть 2+ реальные реализации, или явное требование иметь возможность подмены (тесты через mock — не считается, есть Mockito с mocking конкретных классов)**.

**Premature abstraction**. Дизайним абстракцию до того как поняли что реально нужно варьировать. Обычно ошибаемся — реальная ось изменений оказывается другой, чем предполагали.

Recipe: не создавать абстракцию для потенциальных будущих требований. Создавать когда потребность стала очевидной (правило трёх).

**God interface**. Интерфейс с 40 методами. Все реализации вынуждены имплементировать всё, даже если 30 методов им не нужны. Часто заканчивается `throw new UnsupportedOperationException()` в куче методов.

Recipe: следовать ISP (Interface Segregation Principle, ниже) — маленькие focused интерфейсы, клиенты implement только то что нужно.

**Speculative generality**. `AbstractSomething<T extends BaseEntity, R extends Repository<T>, S extends BaseService<T, R>>`. Три уровня generic'ов, потому что «вдруг понадобится». Никогда не понадобится. Читать невозможно.

**Wrong hierarchy**. Наследование там где нужна композиция. Классика — `class Square extends Rectangle` (Liskov violation, файл ниже).

## SOLID — принципы вокруг абстракции

Пять принципов Robert Martin, направляющих проектирование класс-уровневой абстракции.

**S — Single Responsibility Principle**. Класс должен иметь **одну причину меняться**. Не «одна функция», а один stakeholder изменений. `OrderPersistence` меняется когда меняется схема БД. `OrderCalculator` меняется когда меняются бизнес-правила расчёта. Два разных reason to change → два класса.

Часто нарушается в форме «утилитных» классов с 30 методами про разные предметные области.

**O — Open/Closed Principle**. Классы должны быть **открыты для расширения, закрыты для изменения**. Добавление новой функциональности не должно требовать модификации существующего кода. Достигается через полиморфизм — вместо `if (type == "A") ... else if (type == "B") ...` использовать интерфейс с двумя имплементациями. Новый тип = новый класс, старый код не трогается.

На практике: не превращать в догму. Иногда `switch (enum)` с exhaustive check (sealed types в Java 17+) — правильно. OCP через полиморфизм оправдан когда действительно много типов и они действительно расширяются.

**L — Liskov Substitution Principle**. Objects базового типа должны быть заменяемы объектами subtype без нарушения корректности программы. Если `Rectangle` имеет метод `setWidth(int)` который меняет только width, `Square extends Rectangle` не может менять и height — это нарушение LSP (клиент, работающий с `Rectangle`, ждёт что setWidth не тронет height).

Классический пример «Square extends Rectangle» — иллюстрация что реальная предметная связь (квадрат — частный случай прямоугольника геометрически) не всегда транслируется в правильное наследование в коде.

Recipe: если subtype нарушает контракт supertype — либо изменить дизайн (композиция вместо наследования), либо переопределить абстракцию (нет `setWidth` в базовом типе, оба immutable).

**I — Interface Segregation Principle**. **Много мелких интерфейсов лучше одного большого**. Клиенты не должны зависеть от методов которые не используют.

Пример нарушения — `Worker` интерфейс с `work()`, `eat()`, `sleep()`. `RobotWorker` вынужден имплементировать `eat()` и `sleep()` (throw exception). Правильно — разделить на `Workable`, `Feedable`, `Sleepable`.

В Java практически: если у интерфейса больше 5-7 методов — задуматься о разбиении. Client interfaces (specific to consumer) vs implementation interfaces (all methods) — типовой pattern.

**D — Dependency Inversion Principle**. Модули высокого уровня не должны зависеть от модулей низкого уровня. Оба должны зависеть от абстракций. Абстракции не должны зависеть от деталей — детали должны зависеть от абстракций.

Практически: сервис-слой не импортирует конкретный `HibernateOrderRepository`, а зависит от `OrderRepository` interface. Реализация подключается через DI. Даёт testability (mock repository), заменяемость (можно поменять Hibernate на MyBatis без переписывания сервиса).

DIP — фундамент dependency injection фреймворков (Spring). Также фундамент **hexagonal architecture** (ports & adapters, ниже).

## Паттерны абстракции

**Template Method** — базовый класс определяет скелет алгоритма, subclasses переопределяют шаги. Абстракция общего hollywood principle («don't call us, we'll call you»).

```java
abstract class ReportGenerator {
    public final void generate() {
        fetchData();
        transform();
        render();
        send();
    }
    protected abstract void fetchData();
    protected abstract void transform();
    protected abstract void render();
    protected void send() { /* default impl */ }
}
```

Плюс: общая структура фиксирована в базовом классе. Subclass фокусируется на специфичном.

Минус: жёсткое наследование, LSP-ловушки, тестировать сложнее (не composable).

Альтернатива — **Strategy** через композицию.

**Strategy** — семейство алгоритмов, каждый в отдельном классе, взаимозаменяемые. Клиент выбирает нужный:

```java
interface PaymentStrategy {
    void pay(BigDecimal amount);
}

class CardPayment implements PaymentStrategy { ... }
class PaypalPayment implements PaymentStrategy { ... }
class CryptoPayment implements PaymentStrategy { ... }

class OrderService {
    private final PaymentStrategy payment;  // inject
    void checkout(Order o) { payment.pay(o.total()); }
}
```

Плюс: composable, легко добавить новый способ (OCP), тесты через mock.

Минус: overhead создания объектов, для одноразового кода избыточно.

**Adapter** — переходник между несовместимыми интерфейсами. Есть класс `LegacyPaymentAPI` со старым интерфейсом, нужно использовать через новый `PaymentStrategy`:

```java
class LegacyPaymentAdapter implements PaymentStrategy {
    private final LegacyPaymentAPI legacy;
    
    public void pay(BigDecimal amount) {
        legacy.processPayment(amount.doubleValue(), "USD");
    }
}
```

Классический паттерн интеграции старого кода в новую систему.

**Bridge** — разделение абстракции и реализации так, чтобы менять их независимо. `Shape` (абстракция) и `Renderer` (реализация): `Shape` знает про `Renderer` через композицию, `Circle` использует `Renderer.drawCircle(...)`. Меняем renderer (SVG, Canvas, GL) — все shapes работают.

Похоже на Strategy, но отличается тем что и abstraction, и implementation имеют свои иерархии.

**Repository** — абстракция persistence. Домен работает с `OrderRepository.findById(id)`, не с SQL. Реализация может быть Hibernate, MyBatis, in-memory (для тестов), gRPC к другому сервису. Домен изолирован от способа хранения.

Классический enterprise паттерн (файл 14 про Spring Data JPA — Repository реализация).

**Layers** — горизонтальные слои архитектуры: Presentation → Application → Domain → Infrastructure. Верхний слой зависит от нижнего, не наоборот. Каждый слой — уровень абстракции.

Проблема классических layers: infrastructure (БД, external APIs) традиционно на дне, но домен зависит от них через repository interfaces. Решается либо dependency inversion (repository interface в domain, implementation в infrastructure), либо hexagonal.

**Hexagonal / Ports & Adapters** (Alistair Cockburn, 2005). Домен в центре, вокруг — ports (interfaces которые домен определяет), снаружи — adapters (реализации портов для конкретных технологий).

```
              ┌──── HTTP Adapter ────┐
              │  Kafka Consumer      │
              │  CLI                 │
       ┌──────┴──────┐           ┌───┴──────────┐
       │             │           │              │
   Adapters        Ports      Domain          Ports        Adapters
   (input)       (input)      Core           (output)      (output)
                                                            │
                                                    ┌───────┴──────┐
                                                    │ PostgreSQL   │
                                                    │ Redis Cache  │
                                                    │ Payment API  │
                                                    └──────────────┘
```

Домен ни о ком не знает. Всё общается через ports. Плюсы: тестируемость (mock adapters), заменяемость (новая технология = новый adapter), чёткие границы.

Модель, лежащая в основе современного clean architecture, DDD tactical patterns.

## Domain-Driven Design abstraction

**DDD** (Eric Evans, 2003) — набор принципов для абстракции сложной бизнес-логики.

**Bounded Context** — границы модели. В одном контексте «Customer» может значить одно, в другом — другое. Каждый контекст имеет свою собственную ubiquitous language, свою модель. Не пытаться сделать одну «правильную» модель Customer для всей системы — это провал. Разные контексты — разные модели, интеграция через explicit boundaries.

Пример: в Sales context Customer — с payment history, credit limit. В Shipping context Customer — с address, delivery preferences. Одна и та же сущность в реальности, но разные абстракции для разных задач.

**Aggregate** — кластер объектов, обрабатываемых как единое целое. Имеет aggregate root — единственную точку входа. Инварианты обеспечиваются внутри aggregate transactionally. `Order` (aggregate root) + `OrderItems` — один aggregate. Изменение через `order.addItem(...)`, не напрямую `orderItem.setQuantity(...)`.

**Entity vs Value Object**. Entity — сущность с identity (`Order` — важно какой конкретно, `id` определяет). Value Object — определяется только значением (`Money(100, USD)` — важно только сумма и валюта, два разных объекта с одинаковыми значениями эквивалентны). Записи (records) в Java 16+ — идеальная реализация value objects.

**Domain Service** — операция которая не принадлежит одной entity. `TransferService.transfer(fromAccount, toAccount, amount)` — не логично класть в `Account`, поэтому domain service.

**Domain Event** — важное событие в домене. `OrderPlaced`, `PaymentReceived`. Аналог outbox pattern (файл 51) — публикация событий из домена во внешний мир.

## Практические правила

**Правило трёх** (Sandi Metz): не абстрагируй до третьего повторения. Дублирование терпимо, wrong abstraction — нет.

**Правило симметрии**: если абстракция обещает симметричное поведение (`add` / `remove`), обе стороны должны быть одинаково эффективны и одинаково реализованы. Асимметричная абстракция — плохой знак.

**Правило одного уровня**: метод должен работать на одном уровне абстракции. Смешивание бизнес-логики и низкоуровневых деталей — reader вынужден переключать mental model.

**Правило имени**: имя абстракции должно чётко описывать что она предоставляет клиенту, не как реализована. `OrderRepository` (что) vs `PostgresOrderDAO` (как). Первое лучше — клиент не должен знать о Postgres.

**Правило кабеля**: чем толще интерфейс (много методов, много типов) — тем сложнее его использовать и тестировать. Тонкие focused интерфейсы предпочтительнее.

**Правило один интерфейс — одна реализация**: если единственная имплементация — обычно интерфейс не нужен. Исключение — есть явное требование заменяемости (тесты через integration test с реальной имплементацией — не требование).

**Правило времени жизни**: не абстрагируйся преждевременно. Дождись пока паттерн станет очевидным. Абстракция сделанная до понимания реальных требований почти всегда неправильная.

## Специфика Java

**extends vs implements**. `extends` для наследования (класс от класса, интерфейс от интерфейса). `implements` для реализации интерфейса классом. Множественная реализация интерфейсов разрешена, множественное наследование классов — нет.

**default methods в интерфейсах** (Java 8+) — можно добавлять методы с реализацией без ломания клиентов. Полезно для эволюции интерфейсов без breaking changes.

**Sealed classes / interfaces** (Java 17+) — controlled abstraction: явный список permitted subclasses. Даёт exhaustive pattern matching + предотвращает нежелательные расширения:

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
final class Circle implements Shape { ... }
final class Rectangle implements Shape { ... }
final class Triangle implements Shape { ... }

// Компилятор проверяет exhaustive
String describe(Shape s) {
    return switch (s) {
        case Circle c -> "circle";
        case Rectangle r -> "rect";
        case Triangle t -> "tri";
    };
}
```

Полезно для ADT-style дизайна (алгебраические типы).

**Records** (Java 16+) — immutable value objects с автоматическими accessors, equals, hashCode, toString:

```java
record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount == null || amount.signum() < 0)
            throw new IllegalArgumentException();
    }
}
```

Идеально для value objects в DDD, DTO, event payloads.

**Immutable collections** (`List.of`, `Map.of`) — правильная абстракция коллекции без права модификации. Клиент не может ломать.

**Function<T, R>, Consumer<T>, Supplier<T>, Predicate<T>** — стандартные functional interfaces. Абстракция «функция» как first-class citizen. Passable в методы, composable.

## Заключение

Абстракция — процесс выделения существенного и сокрытия несущественного. Не бесплатна: платишь runtime overhead (indirection), cognitive load (больше уровней для понимания), rigidity (wrong abstraction тяжело переделать). Хорошая абстракция окупается за счёт изоляции изменений, тестируемости, ясности намерения.

**Уровни абстракции** — от машинных инструкций до бизнес-домена. Хороший код работает на одном уровне за раз. Смешивание уровней — признак утечки.

**Инструменты в языках**: функция, класс, interface, abstract class, module, generic type, sealed class (Java 17+), record (Java 16+). Каждый инструмент — своя семантика, выбирать осознанно.

**ADT (Abstract Data Type)** — теоретическая база. Определяется через операции + аксиомы, независимо от реализации.

**Cost of abstraction**: (1) runtime overhead (indirection, allocations, virtual calls — JIT многое устраняет через inlining, но не всегда); (2) cognitive load (больше уровней = сложнее держать в голове); (3) rigidity при wrong abstraction.

**Leaky abstractions** (Spolsky): все нетривиальные абстракции в какой-то степени leaky. Детали реализации просачиваются — TCP latency, ORM N+1, HTTP timeouts. Абстракции экономят время, но не время на обучение.

**Wrong abstraction** (Metz): duplication is far cheaper than the wrong abstraction. Правило трёх — терпи дублирование до третьего повторения, тогда паттерн становится очевидным.

**Anti-patterns**: over-abstraction («на всякий случай»), premature abstraction (до понимания требований), God interface (40 методов), speculative generality (3 уровня generic'ов), wrong hierarchy (наследование там где нужна композиция).

**SOLID** — принципы: **SRP** (одна причина меняться), **OCP** (открыто для расширения, закрыто для изменения), **LSP** (subtypes заменяемы basetypes без нарушения контракта), **ISP** (маленькие focused интерфейсы), **DIP** (зависеть от абстракций, не деталей).

**Паттерны абстракции**: Template Method (общий скелет + subclass steps), Strategy (взаимозаменяемые алгоритмы через композицию), Adapter (несовместимые интерфейсы), Bridge (независимые иерархии abstraction и implementation), Repository (абстракция persistence), Layers (горизонтальные уровни), Hexagonal / Ports & Adapters (домен в центре, adapters вокруг).

**DDD abstraction**: Bounded Context (границы модели, разные контексты — разные абстракции одной реальности), Aggregate (кластер с транзакционными инвариантами через aggregate root), Entity vs Value Object (identity vs value equality — records идеальны для VO), Domain Service, Domain Event.

**Практические правила**: правило трёх (не абстрагируй до третьего повторения); правило одного уровня (метод — один уровень абстракции); правило имени (что предоставляешь, не как реализовано); правило одна реализация → нет interface обычно; правило времени жизни (не преждевременно).

**Java specifics**: sealed для controlled abstractions с exhaustive matching; records для immutable value objects; default methods для evolvable interfaces; functional interfaces как first-class abstraction of function.

Правильная абстракция — не «сделать красиво», а сделать так, чтобы **изменения ложились по границам абстракции**. Если каждое изменение требует трогать и абстракцию, и реализацию, и клиента — граница проведена неверно. Если изменения ложатся на одну сторону границы за раз — абстракция работает.

Facade / Proxy деталь в 85. Design patterns Facade/Proxy там же. Здесь была концепция абстракции: определение, стоимость, хорошая vs плохая, SOLID, паттерны, DDD, practical rules, Java specifics.
