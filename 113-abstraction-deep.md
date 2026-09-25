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

## SOLID: каждый принцип с реальными сценариями

Пять принципов Robert Martin, направляющих проектирование класс-уровневой абстракции. Разберём каждый глубоко — с типичными нарушениями, реальными prod-последствиями, edge cases когда «правильное» решение по SOLID оказывается неправильным.

### Single Responsibility Principle: одна причина меняться

Формулировка Martin'а часто цитируется неточно. Оригинал: «A class should have only **one reason to change**». Не «класс должен делать одну вещь» и не «у класса должна быть одна ответственность» — а именно одна причина меняться. Позднее сам Martin уточнил: причина меняться связана с одним **stakeholder'ом** или одной **осью изменений**.

Пример. Класс `OrderReport` генерирует отчёт по заказам. Кажется одна ответственность — «отчёт по заказам». Реально три:

```java
class OrderReport {
    public String generate() {
        List<Order> orders = fetchFromDatabase();          // (1) как достать данные
        BigDecimal total = calculateBusinessMetrics(orders); // (2) какие метрики
        return renderAsHtml(orders, total);                // (3) какой формат
    }
}
```

Три причины меняться. Схема БД поменялась — правь fetch. Бизнес добавил новую метрику (average order size) — правь calculate. Верстальщик поменял шаблон — правь render. Три разных stakeholder'а (DBA, business analyst, UI designer) → три разных reason to change → три класса:

```java
class OrderRepository { List<Order> findAll(); }
class OrderMetricsCalculator { OrderMetrics compute(List<Order> orders); }
class HtmlOrderRenderer { String render(List<Order> orders, OrderMetrics metrics); }
class OrderReport {   // orchestrator
    private final OrderRepository repo;
    private final OrderMetricsCalculator metrics;
    private final HtmlOrderRenderer renderer;
    
    public String generate() { ... }
}
```

Теперь изменение схемы БД трогает только `OrderRepository`. Новая метрика — только `OrderMetricsCalculator`. Новый формат — только renderer (или добавить `PdfOrderRenderer` рядом).

**Классические нарушения SRP на практике**:

- **Utility classes** с 30 методами про разные домены (`StringUtils`, `Helpers`, `Common`). Меняется каждый раз когда где угодно нужно что-то новое.
- **Fat controllers** — `@RestController` содержит бизнес-логику, валидацию, orchestration, форматирование ответа. Меняется при изменении API, при изменении бизнес-правил, при изменении формата.
- **Anemic domain entities с бизнес-логикой в service layer** — Entity меняется при изменении полей (persistence concern), Service — при изменении логики. Разделение по горизонтали (data / logic) вместо по вертикали (feature).

**Where SRP гнётся**. Иногда «правильное» разделение даёт 10 крошечных классов на одну простую фичу — cognitive load растёт больше чем выигрыш. Правило: разделять когда **уже есть** два разных reason to change или **скоро будет** (по roadmap команды). Не разделять «на случай если понадобится».

### Open/Closed Principle: что реально означает

Принцип: «Software entities should be **open for extension, closed for modification**». Читается странно — если entity закрыто для изменения, как её extend'ить?

Ответ через полиморфизм. Абстракция (interface, abstract class) закрыта для изменения (не меняем сам интерфейс). Расширение — через новые имплементации, которые plug'аются через DI. Существующий код клиента не трогается при добавлении новой имплементации.

Классический пример. Обработка платежей:

**Плохо** (нарушение OCP):
```java
class PaymentProcessor {
    public void process(String method, BigDecimal amount) {
        if (method.equals("CARD")) {
            processCard(amount);
        } else if (method.equals("PAYPAL")) {
            processPaypal(amount);
        } else if (method.equals("CRYPTO")) {
            processCrypto(amount);
        }
        // Новый способ оплаты → новый else if → изменение существующего кода
    }
}
```

**Хорошо** (следует OCP):
```java
interface PaymentMethod {
    void process(BigDecimal amount);
}

class CardPayment implements PaymentMethod { ... }
class PaypalPayment implements PaymentMethod { ... }
class CryptoPayment implements PaymentMethod { ... }

class PaymentProcessor {
    private final Map<String, PaymentMethod> methods;  // DI
    
    public void process(String methodName, BigDecimal amount) {
        methods.get(methodName).process(amount);
    }
}

// Новый способ — новый класс, PaymentProcessor не меняется
class ApplePayPayment implements PaymentMethod { ... }
```

**Где OCP гнётся**. Не всегда полиморфизм лучше `switch`. Когда типов **мало и они стабильны** — sealed interface + exhaustive switch читается лучше и позволяет компилятору проверить полноту:

```java
sealed interface Shape permits Circle, Square, Triangle {}

double area(Shape s) {
    return switch (s) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Square sq -> sq.side() * sq.side();
        case Triangle t -> 0.5 * t.base() * t.height();
    };
}
```

Здесь `switch` явный и exhaustive check компилятора гарантирует что при добавлении нового `Shape` (например Rectangle) все switch-выражения не скомпилируются пока не добавишь case. Это **лучше** полиморфизма для мелких стабильных иерархий — вся логика в одном месте, не разбросана по 3 классам.

**Правило**: OCP через полиморфизм когда типов много и они действительно часто добавляются (open set). Sealed + switch когда типов мало и они стабильны (closed set с известными вариантами).

### Liskov Substitution Principle: почему Square ≠ Rectangle в коде

Формулировка Барбары Лисков (1987): «Если S — subtype of T, объекты type T можно заменить объектами type S без изменения желательных свойств программы». Проще: **subclass должен выполнять контракт superclass**.

Контракт включает не только сигнатуры методов, но и:
- **Предусловия** — что должно быть true до вызова. Subclass **не может усиливать** предусловия (требовать больше от caller'а).
- **Постусловия** — что гарантируется после вызова. Subclass **не может ослаблять** постусловия (обещать меньше).
- **Инварианты** — что true всегда. Subclass должен сохранять.
- **История поведения** — subclass не может ломать историю (нельзя добавить mutability в immutable).

Классический пример нарушения — `Square extends Rectangle`:

```java
class Rectangle {
    protected int width, height;
    public void setWidth(int w) { this.width = w; }   // контракт: меняет только width
    public void setHeight(int h) { this.height = h; } // контракт: меняет только height
    public int area() { return width * height; }
}

class Square extends Rectangle {
    @Override
    public void setWidth(int w) { this.width = w; this.height = w; }   // ломает контракт!
    @Override
    public void setHeight(int h) { this.width = h; this.height = h; }
}

// Клиент работает с Rectangle:
void doubleWidth(Rectangle r) {
    int oldHeight = r.height;
    r.setWidth(r.width * 2);
    assert r.height == oldHeight;   // ломается для Square!
}
```

Геометрически квадрат — частный случай прямоугольника. В коде — нет: `Square` не может корректно выполнить контракт `Rectangle.setWidth`. Наследование неправильное.

**Fix'ы**: (1) сделать оба immutable (`Rectangle` без setter'ов, вместо этого `withWidth(int)` возвращающий новый Rectangle); (2) не наследовать вовсе — два разных класса без общего супертипа; (3) общий интерфейс `Shape` только с методами, которые оба поддерживают (`area()`, `perimeter()`), без setter'ов.

**Реальный prod пример нарушения LSP**. Часто встречается в hierarchies с исключениями:

```java
interface UserRepository {
    User findById(Long id);   // контракт: возвращает User или бросает UserNotFoundException
}

class CachedUserRepository implements UserRepository {
    public User findById(Long id) {
        User cached = cache.get(id);
        if (cached != null) return cached;
        return delegate.findById(id);   // может бросить UserNotFoundException
    }
}

class RemoteUserRepository implements UserRepository {
    public User findById(Long id) {
        try {
            return httpClient.getUser(id);
        } catch (IOException e) {
            throw new RemoteServiceException(e);   // НАРУШЕНИЕ LSP! новый тип исключения
        }
    }
}
```

Клиент кода написан против `UserRepository` и catch'ит `UserNotFoundException`. С `RemoteUserRepository` вдруг летит `RemoteServiceException` которую клиент не ждал — необработанное исключение, 500 в контроллере.

**Recipe**: если subtype не может честно выполнить контракт — либо изменить дизайн (нет `setWidth` в базовом типе, immutable API), либо не использовать наследование (разные типы без общего супертипа), либо расширить контракт супертипа так чтобы включить всех subtypes честно.

### Interface Segregation Principle: тонкие интерфейсы

Формулировка: «Clients should not be forced to depend upon interfaces that they do not use». Иными словами — **много мелких focused интерфейсов лучше одного большого**.

Каноническое нарушение — `Worker`:

```java
interface Worker {
    void work();
    void eat();
    void sleep();
}

class HumanWorker implements Worker {
    public void work() { ... }
    public void eat() { ... }
    public void sleep() { ... }
}

class RobotWorker implements Worker {
    public void work() { ... }
    public void eat() { 
        throw new UnsupportedOperationException("Robots don't eat");
    }
    public void sleep() {
        throw new UnsupportedOperationException("Robots don't sleep");
    }
}
```

`RobotWorker` вынужден реализовать методы которые не имеет смысла — `throw new UnsupportedOperationException()`. Хуже — клиент кода получил `Worker robot` и вызвал `robot.eat()` — runtime crash вместо compile-time ошибки.

Разделение на маленькие интерфейсы:

```java
interface Workable { void work(); }
interface Feedable { void eat(); }
interface Sleepable { void sleep(); }

class HumanWorker implements Workable, Feedable, Sleepable { ... }
class RobotWorker implements Workable { ... }

// Клиент запрашивает только то что реально нужно:
void assignJob(Workable w) { w.work(); }   // работает для обоих
void feedBreak(Feedable f) { f.eat(); }    // только для тех кто ест
```

**Java конкретные проявления ISP**:

- **`java.util.Collection` — большой интерфейс** с ~30 методами. Кому нужен только `add()` вынужден зависеть от всего. Разделение хотели сделать при добавлении Streams (Java 8), но обратная совместимость сохранила статус-кво.
- **`java.io.ObjectInputStream` / `ObjectOutputStream`** — толстые интерфейсы, требуют реализации serialize/deserialize. Многие клиенты хотели только чтение или только запись — отсюда сepуarate `Serializer` и `Deserializer` в современных библиотеках (Jackson, Kryo).
- **Repository интерфейсы Spring Data**. `JpaRepository<T, ID>` даёт save/findAll/delete/count и ещё десяток методов. Часто конкретному use case нужен только `findById(ID)`. Правильный подход — свои узкие интерфейсы: `interface UserFinder { User findById(Long id); }`, реализация делегирует в `JpaRepository`.

**Правило для Java**: если интерфейс имеет >5-7 методов — задуматься о разбиении. Если для существующего интерфейса тесты часто требуют mock'ить методы которые тест не использует — интерфейс слишком широк. Client-specific интерфейсы (Role Interfaces) — идиома чётко разделяющая нужды разных клиентов.

### Dependency Inversion Principle: не путать с DI

Формулировка: «High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions».

Часто путают с **Dependency Injection**. DI — механизм внедрения (Spring `@Autowired`), DIP — принцип дизайна о направлении зависимостей.

Нарушение DIP:

```java
// High-level: business logic
class OrderService {
    private final PostgresOrderRepository repo = new PostgresOrderRepository();
    private final SmtpEmailSender email = new SmtpEmailSender();
    
    public void placeOrder(Order order) {
        repo.save(order);
        email.send(order.getUserEmail(), "Order placed");
    }
}
```

`OrderService` (high-level, бизнес-логика) напрямую зависит от `PostgresOrderRepository` (low-level, детали БД) и `SmtpEmailSender` (low-level, детали email). Изменение БД (Postgres → MongoDB) требует переписывания `OrderService`. Тесты OrderService невозможны без реального Postgres и SMTP.

Следуя DIP:

```java
// Абстракции определены в domain (high-level слое)
interface OrderRepository { void save(Order order); }
interface EmailSender { void send(String to, String body); }

class OrderService {
    private final OrderRepository repo;
    private final EmailSender email;
    
    // DI: конкретные реализации внедряются извне
    public OrderService(OrderRepository repo, EmailSender email) { ... }
    
    public void placeOrder(Order order) {
        repo.save(order);
        email.send(order.getUserEmail(), "Order placed");
    }
}

// Реализации в infrastructure (low-level слое) зависят от абстракций
class PostgresOrderRepository implements OrderRepository { ... }
class SmtpEmailSender implements EmailSender { ... }
```

Направление зависимостей: `PostgresOrderRepository → OrderRepository ← OrderService`. Обе стороны зависят от абстракции `OrderRepository`. Абстракция «в середине», принадлежит domain слою (там где `OrderService`).

Ключевое: **abstraction принадлежит high-level слою**, не low-level. Интерфейс `OrderRepository` находится в пакете `com.example.domain`, не в `com.example.infrastructure.postgres`. Реализация в infrastructure импортирует domain — не наоборот.

**Реальные последствия DIP в prod**. Тесты OrderService — mock репозиторий и email, никакой БД в тестах. Смена БД на MongoDB — новый класс `MongoOrderRepository implements OrderRepository`, OrderService не тронут. Переход на event-based email (вместо SMTP) — новый `EventBasedEmailSender`, OrderService не тронут.

**DIP — фундамент**:
- **Dependency Injection** (Spring, Guice) — инструмент реализации DIP на уровне класса.
- **Hexagonal Architecture** — DIP на уровне системы (домен определяет ports, adapters их реализуют).
- **Testability** — mock'и работают потому что зависимость на абстракцию, не на реализацию.
- **Plugin architecture** — новые plugins добавляются реализацией известного интерфейса.

Where DIP гнётся. Для действительно простых utility функций (математика, форматирование строк) абстракция избыточна. `Math.sqrt(x)` — прямая зависимость от JDK, не через `SquareRootCalculator` interface. Правило: DIP применять к границам между **важными концепциями** системы (домен ↔ инфраструктура, бизнес-логика ↔ внешние API), не ко всему подряд.

## Композиция vs наследование: старая битва

Одна из фундаментальных ошибок начинающих — использовать наследование там где надо композицию. GoF book в 1994 сформулировала: **«Favor composition over inheritance»**. Сорок лет спустя правило актуально.

**Проблемы наследования**:

**Хрупкая иерархия**. Изменения в базовом классе распространяются на всех subclasses. Класс с 20 subclasses — изменение в базовом методе может сломать любой из них непредсказуемо.

**Fragile base class problem**. Изменение implementation деталей базового класса (даже без изменения API) может сломать subclasses. Пример: базовый класс `HashSet` использовал `add(...)` внутренне для `addAll(...)`. Subclass переопределил `add(...)` для счётчика. `addAll(coll)` вдруг стал считать каждый элемент дважды (один раз в `add`, один раз внутри `addAll`). Изменение внутренней реализации в `HashSet` меняющее внутреннюю связь — сломает subclass.

**Иерархия не соответствует реальности**. Real-world отношения редко чисто иерархичны. `Manager extends Employee` — а если Manager может стать Employee (демоушен)? Наследование не поддерживает смену типа во время жизни объекта.

**Ограничения LSP** (обсуждено выше). Многие «естественные» иерархии нарушают LSP (Square/Rectangle, Ostrich/Bird — страус не может fly()).

**Единственное наследование в Java**. Класс может extends только один класс. Если нужно поведение из двух мест — придётся дублировать или переструктурировать.

**Композиция решает всё это**:

```java
// Наследование:
class LoggingHashMap<K,V> extends HashMap<K,V> {
    @Override
    public V put(K key, V value) {
        log.debug("put: " + key);
        return super.put(key, value);
    }
    // Проблема: HashMap может внутренне использовать put в других методах,
    // логирование сработает неожиданно. Плюс — все методы HashMap автоматически 
    // унаследованы, включая те что не хочется exposить.
}

// Композиция:
class LoggingMap<K,V> implements Map<K,V> {
    private final Map<K,V> delegate;
    
    public LoggingMap(Map<K,V> delegate) { this.delegate = delegate; }
    
    public V put(K key, V value) {
        log.debug("put: " + key);
        return delegate.put(key, value);
    }
    // ... остальные методы Map делегируем без магии
}
```

Композиционная версия работает предсказуемо: логгируется только явный `put(...)`, не внутренние вызовы `HashMap`. И работает с **любой** реализацией `Map` (HashMap, ConcurrentHashMap, LinkedHashMap), не только с HashMap.

**Когда наследование всё-таки правильно**:

- **Настоящий is-a**, где subclass действительно всё что супертип и что-то ещё. `IOException extends Exception` — IOException это exception + дополнительная информация.
- **Template Method** — общий скелет алгоритма плюс варьируемые шаги. Хотя часто лучше выразить через Strategy (композиция).
- **Sealed hierarchies** для ADT. `Result` = `Success<T> | Failure<E>` — closed set, известные варианты.
- **Framework hooks** — переопределение методов для интеграции с фреймворком (например `extends AbstractController` в старом Spring MVC).

**Правило**: по умолчанию **композиция**. Наследование только когда есть чёткий is-a **и** контракт супертипа честно выполняется. Никогда не наследовать «чтобы переиспользовать код» — это code reuse via inheritance, известный anti-pattern.

## Паттерны абстракции: механика и trade-offs

Разберём паттерны глубже — не «что это», а «когда работает / когда нет / чем платишь».

### Template Method: скелет + переменные шаги

Базовый класс определяет общий алгоритм с абстрактными «дырами» для переменных шагов. Subclasses реализуют шаги.

```java
abstract class DataProcessor {
    // Template method — final чтобы subclass не мог сломать скелет
    public final void process() {
        Data data = fetch();
        Data validated = validate(data);
        Data transformed = transform(validated);
        save(transformed);
        notify();
    }
    
    protected abstract Data fetch();
    protected abstract Data transform(Data data);
    protected abstract void save(Data data);
    
    // Hook с default реализацией — subclass может override
    protected Data validate(Data data) { return data; }
    protected void notify() { /* по умолчанию ничего */ }
}

class DailyReportProcessor extends DataProcessor {
    protected Data fetch() { return db.query("..."); }
    protected Data transform(Data d) { return aggregate(d); }
    protected void save(Data d) { fileSystem.write(d); }
    protected void notify() { email.send("Report ready"); }
}
```

**Плюсы**. Скелет фиксирован — все процессоры проходят те же шаги в том же порядке. Общий код (`process()`) не дублируется. Обеспечивается Hollywood Principle («don't call us, we'll call you») — subclass не контролирует flow, только заполняет hooks.

**Минусы**. Жёсткое наследование — вся LSP-проблематика применима. Инверсия зависимости невозможна (subclass **зависит от** базового класса, не наоборот). Тестировать сложнее — нужно создавать fake subclass, не может mock'аться. Runtime поведение зашито в тип — нельзя менять на лету.

**Когда работает**. Класс операций с настоящим общим скелетом (`Spring JdbcTemplate.query(...)` — открыть connection → PreparedStatement → выполнить → mapping → закрыть, разные mappers как subclass шаги). Alternative — **Strategy через композицию** обычно лучше, кроме случаев где скелет действительно фиксирован и общий.

### Strategy: композиция взаимозаменяемых алгоритмов

Семейство алгоритмов вынесено в отдельные классы, реализующие общий интерфейс. Клиент выбирает нужный через DI или явную передачу.

```java
interface CompressionStrategy {
    byte[] compress(byte[] data);
    byte[] decompress(byte[] data);
}

class GzipStrategy implements CompressionStrategy { ... }
class LzoStrategy implements CompressionStrategy { ... }
class ZstdStrategy implements CompressionStrategy { ... }
class NoCompressionStrategy implements CompressionStrategy {
    public byte[] compress(byte[] data) { return data; }
    public byte[] decompress(byte[] data) { return data; }
}

class FileArchiver {
    private final CompressionStrategy compression;
    
    public FileArchiver(CompressionStrategy compression) { ... }
    
    public void archive(File file) {
        byte[] compressed = compression.compress(read(file));
        save(compressed);
    }
}
```

**Плюсы**. Composable — можно комбинировать strategies runtime (`if (fileSize > 1GB) use zstdStrategy else use gzipStrategy`). Тестируется через mock. Добавление нового алгоритма — новый класс, старый код не меняется (OCP). Стратегия — first-class object, можно передавать, сериализовать, конфигурировать.

**Минусы**. Больше классов чем при простом `switch` — overhead cognitive load для очевидных случаев. Каждый strategy — отдельный объект (аллокация, GC). Если стратегий очень много (сотни) — управление ими становится проблемой.

**Practical Java 8+ упрощение**. Многие strategy interfaces с одним методом можно заменить на functional interfaces + lambdas:

```java
// Вместо интерфейса + классов:
Function<byte[], byte[]> compression = GZIPStrategy::compress;

// Или прямо в конструкторе:
new FileArchiver(data -> gzipCompressor.compress(data));
```

Для простых случаев (одноstrofный алгоритм) lambda короче и не создаёт лишних классов. Для сложных (несколько методов, состояние, конфигурация) — отдельный класс лучше.

### Adapter: переходник между несовместимыми интерфейсами

Есть класс с интерфейсом `X`, клиент ждёт интерфейс `Y`. Adapter — обёртка, конвертирующая `X → Y`.

Два вида: **Object Adapter** (композиция — adapter содержит адаптируемый объект как поле) и **Class Adapter** (multiple inheritance — не работает в Java напрямую, только через interface + delegation).

**Реальный пример из enterprise**. Legacy система с proprietary API, нужно интегрировать через стандартный interface своей системы:

```java
// Наш стандартный interface:
interface PaymentGateway {
    PaymentResult pay(BigDecimal amount, String currency, String cardToken);
}

// Legacy library с другим API (не можем менять):
class LegacyPaymentService {
    public int processPayment(double amount, String currencyCode, 
                              String encryptedCard, Map<String,Object> options) {
        // ... возвращает integer status code
    }
}

// Adapter:
class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPaymentService legacy;
    
    public PaymentResult pay(BigDecimal amount, String currency, String cardToken) {
        Map<String,Object> options = defaultOptions();
        int statusCode = legacy.processPayment(
            amount.doubleValue(), currency, cardToken, options
        );
        return convertStatus(statusCode);
    }
    
    private PaymentResult convertStatus(int code) {
        return switch (code) {
            case 0 -> PaymentResult.success();
            case 1, 2, 3 -> PaymentResult.declined();
            default -> PaymentResult.error("Unknown code: " + code);
        };
    }
}
```

Клиент использует `PaymentGateway`, ничего не знает про legacy. При замене legacy на другое (или на новый API) — новый adapter, клиент не меняется.

**Не путать Adapter с Facade**. Adapter меняет **форму** интерфейса (1-to-1 конвертация). Facade **объединяет** несколько интерфейсов за одним упрощённым (N-to-1 упрощение). Оба паттерна wrap, но с разной целью.

### Bridge: две иерархии, независимая эволюция

Bridge разделяет **абстракцию** и её **реализацию** так, чтобы каждая могла эволюционировать независимо. Классический пример: Shape / Renderer.

```java
// Abstraction hierarchy
abstract class Shape {
    protected final Renderer renderer;   // bridge to implementation
    
    protected Shape(Renderer renderer) { this.renderer = renderer; }
    
    abstract void draw();
}

class Circle extends Shape {
    private final double radius;
    
    public Circle(Renderer renderer, double radius) {
        super(renderer);
        this.radius = radius;
    }
    
    public void draw() { renderer.renderCircle(radius); }
}

class Square extends Shape {
    private final double side;
    // ...
    public void draw() { renderer.renderSquare(side); }
}

// Implementation hierarchy
interface Renderer {
    void renderCircle(double radius);
    void renderSquare(double side);
}

class OpenGLRenderer implements Renderer { ... }
class DirectXRenderer implements Renderer { ... }
class SvgRenderer implements Renderer { ... }
```

Две иерархии: Shape (Circle, Square, Triangle...) и Renderer (OpenGL, DirectX, SVG...). Комбинация даёт **M × N** возможных сочетаний без создания класса на каждую комбинацию.

**Разница с Strategy**. Strategy — один интерфейс алгоритма + композиция в клиенте. Bridge — две **параллельные иерархии**, обе развиваются. Отношения между ними — many-to-many (любой Shape с любым Renderer).

**Реальные примеры Bridge**:
- **JDBC** — Application ← DriverManager → Driver (Postgres, MySQL, Oracle). Application не привязана к конкретной БД, driver подключается через unified interface.
- **SLF4J** — Application code → SLF4J API → Logging implementation (Logback, Log4j2, java.util.logging). Разработка application и logging backend независимы.

### Repository: абстракция persistence

Домен работает с `OrderRepository.findById(id)`, не с SQL/ORM/файлами. Реализация может быть какой угодно.

```java
// В domain layer
interface OrderRepository {
    Optional<Order> findById(OrderId id);
    List<Order> findByStatus(OrderStatus status);
    void save(Order order);
    void delete(OrderId id);
}

// В infrastructure layer — Hibernate implementation
@Repository
class HibernateOrderRepository implements OrderRepository {
    @PersistenceContext EntityManager em;
    
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(em.find(OrderEntity.class, id.value()))
                       .map(this::toDomain);
    }
    // ...
    
    private Order toDomain(OrderEntity entity) { ... }
    private OrderEntity toEntity(Order domain) { ... }
}
```

**Ключевые характеристики**:

- Repository — **интерфейс в domain**, реализация в infrastructure. Пример правильного применения DIP.
- Возвращает и принимает **domain entities**, не persistence entities. Между ними — mapping.
- **Не exposes JPA/SQL abstractions**. Клиент не видит EntityManager, Session, Query. Только domain-специфичные методы (`findByCustomerAndDateRange(...)`).
- Даёт **интерфейс коллекции** — findAll(), findByX(), save(), delete(). Как in-memory коллекция, только персистентная.

**Отличие от DAO**. DAO (Data Access Object) — тонкий wrapper над JDBC/ORM, ориентирован на persistence операции. Repository — абстракция коллекции domain объектов, ориентирован на domain-язык. `UserDao.executeQuery("...")` vs `UserRepository.findActiveUsersInRegion(...)`.

**Spring Data JPA** — фреймворк частично автоматизирующий Repository. Определяешь interface `extends JpaRepository<T, ID>`, Spring генерирует implementation via proxy. Плюс — меньше boilerplate. Минус — интерфейс не в domain (он расширяет infrastructure JpaRepository), и часто оказывается слишком широким (см. ISP выше).

Правильный enterprise подход: свой узкий interface в domain, отдельная имплементация которая **делегирует** в Spring Data JPA. Domain не зависит от Spring Data.

### Layers: горизонтальное разделение с dependency inversion

Классические слои — Presentation → Application → Domain → Infrastructure. Верхние зависят от нижних, не наоборот. Каждый слой — уровень абстракции.

Проблема традиционных layers: **infrastructure на дне**, а domain от него зависит через repository. Направление зависимостей: Domain → Infrastructure. Плохо: изменение infrastructure требует изменения domain.

Решение через **Dependency Inversion**: repository interface в **domain**, implementation в infrastructure. Направление: Infrastructure → Domain (реализация зависит от абстракции). Domain ни о чём не знает.

```
Presentation (Controllers)
     ↓ depends on
Application (Use Cases, Orchestration)
     ↓ depends on
Domain (Entities, Value Objects, Repository interfaces, Domain Services)
     ↑ depends on (dependency inversion!)
Infrastructure (Repository implementations, DB, external APIs)
```

Стрелки зависимости с infrastructure идут **вверх** — infrastructure импортирует domain, не наоборот. Domain (сердце системы) свободен от технических деталей.

**Правило layer**: каждый слой видит только слой прямо ниже (или через инверсию). Presentation не знает про infrastructure напрямую. Application ничего не знает про HTTP или SQL. Domain — чистая бизнес-логика.

Нарушения на практике часто в форме «Anemic domain + fat service». Entity без поведения (только getters/setters), вся логика в service. Domain теряет смысл — становится DTO вместо носителя знания.

### Hexagonal / Ports & Adapters: layers на стероидах

Alistair Cockburn (2005) переформулировал layers в **hexagonal architecture**. Домен в центре, вокруг **ports** (интерфейсы которые домен определяет), снаружи **adapters** (реализации портов для конкретных технологий).

```
                ┌─── HTTP Adapter ────┐
                │  Kafka Consumer     │ ← inbound adapters
                │  CLI                │
         ┌──────┴──────┐          ┌───┴──────────┐
         │             │          │              │
   Inbound         Inbound       Domain       Outbound     Outbound
   Adapters        Ports         Core         Ports        Adapters
                                                            │
                                                    ┌───────┴──────┐
                                                    │ PostgreSQL   │
                                                    │ Redis Cache  │ ← outbound adapters
                                                    │ Payment API  │
                                                    └──────────────┘
```

**Inbound ports** — интерфейсы через которые внешний мир зовёт домен (`CreateOrderUseCase.execute(...)`, `SearchProducts.search(...)`). **Outbound ports** — интерфейсы через которые домен зовёт внешний мир (`OrderRepository`, `PaymentGateway`, `EventPublisher`).

**Inbound adapters** — HTTP controllers, Kafka consumers, CLI. Каждый переводит внешний запрос в вызов inbound port. **Outbound adapters** — Hibernate repository, HTTP клиент к payment provider, Kafka publisher. Каждый реализует outbound port технологией.

Ключевое: **домен определяет форму портов**. Не HTTP layer говорит «мне нужен вот такой метод», а домен говорит «вот use case» — HTTP layer подстраивается через adapter.

**Плюсы hexagonal**:

- **Тестируемость**. Тесты бизнес-логики — только с mock adapters. Никакой Spring, никакой БД. Быстрые unit tests с реальным domain кодом.
- **Заменяемость технологий**. REST → GraphQL — новый inbound adapter, домен не меняется. Postgres → MongoDB — новый outbound adapter. Kafka → RabbitMQ — тот же паттерн.
- **Чёткие границы**. Что бизнес-логика, что инфраструктура — видно сразу. Меньше «boundary erosion» (просачивание технических деталей в домен).
- **Плагиновость**. Новый канал (например Slack bot вместо HTTP) — inbound adapter. Домен работает как раньше.

**Минусы**:

- **Overhead для CRUD-приложений**. Если приложение — тонкий CRUD-shell поверх БД, hexagonal — overkill. Много кода на mapping domain ↔ persistence entities, много interfaces которые никогда не будут replaced.
- **Cognitive load**. Разработчику надо помнить где что живёт (domain vs infrastructure), какие интерфейсы куда, mapping между слоями.
- **Учебная кривая**. Команда должна понимать модель. Junior часто ломает границы («быстренько импортну JpaRepository в service, что такого»).

**Where hexagonal shines**: сложный домен с реальной бизнес-логикой (страхование, банкинг, налоги, ERP), долгоживущее приложение (5+ лет), несколько каналов доступа (HTTP + Kafka + CLI + batch), возможность смены технологий (миграция с Oracle на Postgres, замена enterprise message bus).

**Where hexagonal вреден**: simple CRUD (blog, todo app), prototypes, short-lived scripts, приложения где domain логика тривиальная.

Модель лежит в основе современного clean architecture (Uncle Bob), DDD tactical patterns.

## Domain-Driven Design: тактические паттерны

DDD (Eric Evans, 2003) — набор принципов и паттернов для абстракции сложной бизнес-логики. Не про технологии, а про то **как моделировать домен**. Особенно ценно в сложных предметных областях.

### Bounded Context: разные модели для разных задач

Основная идея DDD: **не пытайся сделать одну «правильную» модель**. Разные части системы работают с разными аспектами одной реальности. Каждая часть должна иметь свою модель, оптимизированную для её задач.

Пример: система e-commerce, сущность «Customer». В разных контекстах она разная:

- **Sales context**: Customer с payment history, credit limit, discount tier, purchase frequency. Оптимизировано для расчёта цен и скидок.
- **Shipping context**: Customer с delivery addresses, delivery preferences (утро/вечер), signature required flag. Оптимизировано для логистики.
- **Support context**: Customer с contact info, support ticket history, communication preferences. Оптимизировано для work'а с обращениями.
- **Marketing context**: Customer с demographic data, campaign responses, segment membership. Оптимизировано для таргетинга.

Одна реальная сущность (человек-покупатель) — четыре модели. Пытаться сделать одну God-модель с всеми полями — путь в никуда: сложная валидация, огромный класс, изменение под одну потребность ломает другие.

**Bounded Context** — граница, внутри которой модель имеет одно чёткое значение. За границей — другой контекст, другая модель. Интеграция между контекстами — **explicit boundaries**, обычно через events или API contracts.

**Context Map** — визуализация связей между контекстами:

- **Shared Kernel** — общий кусок кода, разделяемый двумя контекстами (редко, обычно anti-pattern из-за coupling).
- **Customer-Supplier** — один контекст (supplier) предоставляет API, другой (customer) использует. Supplier должен учитывать нужды customer'а.
- **Conformist** — customer вынужден адаптироваться к API supplier'а без переговоров (external systems).
- **Anti-Corruption Layer** — слой перевода между контекстами (обычно между legacy и новым). Внешние структуры не «протекают» в новый домен.
- **Published Language** — общий формат обмена (обычно JSON schema, Protobuf) между независимыми контекстами.

Правильная микросервисная архитектура следует Bounded Context: каждый микросервис — отдельный context, свой домен, свой storage, интеграция через events/API.

### Aggregate: транзакционная граница

**Aggregate** — кластер связанных объектов, обрабатываемых как единое целое. Имеет **aggregate root** — единственную точку входа. Инварианты (правила консистентности) обеспечиваются внутри aggregate transactionally.

Пример: Order aggregate.

```java
class Order {   // Aggregate root
    private final OrderId id;
    private OrderStatus status;
    private final List<OrderItem> items;
    private BigDecimal total;
    
    public void addItem(Product product, int quantity) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Can only add items to draft orders");
        }
        items.add(new OrderItem(product, quantity));
        recalculateTotal();   // инвариант: total = sum(items)
    }
    
    public void removeItem(OrderItemId itemId) {
        if (status != OrderStatus.DRAFT) throw new IllegalStateException(...);
        items.removeIf(i -> i.getId().equals(itemId));
        recalculateTotal();
    }
    
    public void submit() {
        if (items.isEmpty()) throw new IllegalStateException("Cannot submit empty order");
        this.status = OrderStatus.SUBMITTED;
    }
    
    private void recalculateTotal() {
        this.total = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

class OrderItem {   // Внутренний объект aggregate
    private final OrderItemId id;
    private final Product product;
    private final int quantity;
    // Не имеет ссылок наружу aggregate
}
```

**Правила aggregate**:

- **Единственная точка входа — root**. Внешний код работает только с `Order`, не с `OrderItem` напрямую. `orderItem.setQuantity(5)` — запрещено, только через `order.updateItemQuantity(itemId, 5)`.
- **Инварианты внутри одной транзакции**. `total = sum(items)` — при любом изменении items, total пересчитывается атомарно. Транзакция БД охватывает весь aggregate.
- **Ссылки на другие aggregates — только по ID**, не по object reference. `Order` содержит `CustomerId`, не `Customer`. Иначе aggregates «спутываются», консистентность становится глобальной проблемой.
- **Один aggregate за транзакцию**. Если операция требует изменения двух aggregates — это не транзакция, а eventually consistent flow (через события, saga pattern — файл 50).

**Как выбрать границы aggregate**. Правило: инварианты должны быть внутри. Если правило «total = sum(items)» — items и total в одном aggregate. Если правило «total per customer < credit limit» — Customer credit limit должен быть в Order aggregate или проверка через event.

**Anti-pattern: слишком большие aggregates**. `Customer` содержит все Orders, все Addresses, всю history — гигантский aggregate. Загрузка всего Customer'а на каждое изменение — dead slow. Concurrent updates бьются за один lock. Правильно — разделить на несколько aggregates, объединять через IDs.

### Entity vs Value Object: identity vs равенство

**Entity** — объект с identity. Определяется идентификатором, не значением полей. Два `Order`'а с одинаковыми полями — разные, если разные ID. `Order` меняется во времени (статус, items), но остаётся тем же Order.

**Value Object** — объект без identity. Определяется значением полей. Два `Money(100, USD)` — эквивалентны и взаимозаменяемы. Обычно immutable — новое значение = новый объект.

Классические value objects:

- **Money** — сумма + валюта. Immutable. Операции возвращают новый Money: `money.add(other)` = новый Money.
- **DateRange** — начало + конец. Immutable. Операции: `range.overlaps(other)`, `range.contains(date)`.
- **Address** — улица, город, индекс. Immutable. Изменение адреса = новый Address, не мутация старого.
- **PhoneNumber** — code + number. Immutable. Валидация в конструкторе.
- **Coordinates** — lat, lng. Immutable.

**Java records** (Java 16+) — идеальная реализация value objects:

```java
record Money(BigDecimal amount, Currency currency) {
    public Money {   // compact constructor
        if (amount == null || currency == null) 
            throw new IllegalArgumentException();
        if (amount.signum() < 0)
            throw new IllegalArgumentException("Money cannot be negative");
    }
    
    public Money add(Money other) {
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
        return new Money(amount.add(other.amount), currency);
    }
    
    public Money multiply(BigDecimal factor) {
        return new Money(amount.multiply(factor), currency);
    }
}
```

Автоматические equals/hashCode/toString, автоматическая immutability, compact constructor для валидации.

**Плюсы value objects**:

- **Иммутабельность → thread-safety** без синхронизации.
- **Value equality** — можно использовать как ключи Map, элементы Set.
- **Явные операции с семантикой** — `money.add(...)`, `range.overlaps(...)` вместо `if (money1.amount + money2.amount < ...)`.
- **Валидация в конструкторе** — невозможно создать invalid state.
- **Rich domain expression** — `if (order.getShippingAddress().isInSameCityAs(billingAddress))` вместо grocery-level кода.

**Anemic model vs Rich model**. Anemic — Entity как data structure с getters/setters, вся логика в services. Обычно проще на старте, но становится cluster of transactions procedures. Rich — Entity с поведением (`order.submit()`, `order.cancel()`), Value Objects с операциями, Domain Services только для операций не принадлежащих одной сущности. Rich лучше выражает домен, но требует дисциплины.

### Domain Service: операции между сущностями

Не все операции принадлежат одной entity. Money transfer между двумя accounts — не принадлежит ни отправителю, ни получателю, а обеим. Placing order — включает Customer, Product inventory, Payment.

**Domain Service** — класс без state, содержащий такие операции:

```java
class MoneyTransferService {
    public void transfer(Account from, Account to, Money amount) {
        from.withdraw(amount);   // операция на entity
        to.deposit(amount);      // операция на entity
        // Note: транзакционная граница обычно вокруг всего вызова
    }
}
```

**Отличать от application service**. Application service — orchestrator use case (открывает транзакцию, зовёт repository, зовёт domain services, коммитит, публикует events). Domain service — чистая бизнес-логика без infrastructure concerns.

**Anti-pattern**: превращение domain service в свалку операций («UserService» с 40 методами — user management, permissions, notifications, reporting). Правильно — узкие focused services (`UserRegistrationService`, `UserPermissionService`, `UserProfileService`), каждый одна operation или очень тесно связанные.

### Domain Event: важные факты домена

**Domain Event** — факт что что-то важное произошло в домене. `OrderPlaced`, `PaymentReceived`, `ShipmentDelivered`, `CustomerRegistered`. Обычно immutable, содержит достаточно данных для обработки consumers.

```java
record OrderPlaced(
    OrderId orderId,
    CustomerId customerId,
    List<OrderLine> lines,
    Money total,
    Instant occurredAt
) {}
```

События генерируются aggregates (обычно в моменте важного изменения состояния):

```java
class Order {
    private final List<DomainEvent> events = new ArrayList<>();
    
    public void submit() {
        if (items.isEmpty()) throw new IllegalStateException();
        this.status = OrderStatus.SUBMITTED;
        events.add(new OrderPlaced(id, customerId, snapshotLines(), total, now()));
    }
    
    public List<DomainEvent> pullEvents() {
        List<DomainEvent> snapshot = new ArrayList<>(events);
        events.clear();
        return snapshot;
    }
}
```

Публикация — через **outbox pattern** (файл 51). Сначала сохранить события в БД в той же транзакции что и aggregate, потом отдельный publisher шлёт в брокер. Гарантия: если БД коммит, событие обязательно опубликуется.

**Плюсы событийного подхода**:

- **Decoupling** — publisher не знает про subscribers. Новый consumer (аналитика, notifications) — просто подписывается.
- **Audit trail** — события сами по себе — историческая запись. Что произошло, когда, с какими данными.
- **Event Sourcing** — крайний случай, где события **и есть** state. Текущее состояние aggregate вычисляется replay'ом всех его событий. Даёт time-travel debugging, но сложнее в реализации.

### Anti-Corruption Layer: защита от внешнего хаоса

Часто система интегрируется с legacy или external system со своей моделью, отличающейся от нашей. **Anti-Corruption Layer (ACL)** — слой перевода, который **не пропускает** внешние концепции в наш домен.

```java
// Внешний legacy API имеет свою модель:
class LegacyOrderApi {
    // Order c 40 полями, включая технические, дублирующиеся, устаревшие
    LegacyOrder getOrder(long id);
    Map<String,Object> processOrderMetadata(...); // возвращает untyped Map
}

// Наш домен:
class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final Money total;
    private final OrderStatus status;
}

// ACL — переводчик:
class LegacyOrderAdapter {
    private final LegacyOrderApi legacyApi;
    
    public Order fetchOrder(OrderId id) {
        LegacyOrder legacy = legacyApi.getOrder(id.value());
        return new Order(
            new OrderId(legacy.getOrderId()),
            new CustomerId(legacy.getCustomerRef()),   // legacy calls it customerRef
            new Money(new BigDecimal(legacy.getAmt()), Currency.getInstance(legacy.getCurr())),
            mapLegacyStatus(legacy.getSt())
        );
    }
    
    private OrderStatus mapLegacyStatus(String legacyStatus) {
        // Legacy: "N" for new, "P" for paid, ...
        return switch (legacyStatus) {
            case "N" -> OrderStatus.NEW;
            case "P" -> OrderStatus.PAID;
            case "S" -> OrderStatus.SHIPPED;
            default -> throw new IllegalArgumentException("Unknown status: " + legacyStatus);
        };
    }
}
```

Клиенты внутри нашего домена видят только `Order` (наш). Всё «уродство» legacy — в ACL. Замена legacy на новую систему — переписать ACL, домен не меняется.

**Классический use case ACL**: legacy migration. Постепенное переписывание из monolith в microservices — новые сервисы работают со своей моделью, legacy читается через ACL пока не будет удалён.

## Практические правила абстракции: расширенно

Разберём глубже правила которые действительно работают в prod.

**Правило трёх** (Sandi Metz). Не абстрагируй до третьего повторения. Первый раз — просто пишешь код. Второй раз — copy-paste с изменениями, терпи дублирование. Третий раз — теперь видно паттерн, можно выделить общее.

Обоснование: с двух примеров нельзя понять что реально общее, а что случайно совпадающее. Три примера дают достаточно data points для видения оси изменений. Абстракция сделанная на двух примерах почти всегда неправильно проведена — на третьем случае приходится либо ломать абстракцию, либо натягивать через параметры и специальные случаи.

Есть исключения. Trivial дублирование (три `if (x == null) throw new NPE()`) — можно выделить сразу в утилиту (`Objects.requireNonNull` уже есть). Domain концепции с одной ясной семантикой (Money) — value object с самого начала, не ждём троекратного повторения. Правило работает для **бизнес-логики**, не для утилит.

**Правило симметрии**. Абстракция должна быть симметричной. `add()` и `remove()` должны быть одинаково эффективны и одинаково реализованы. Асимметрия — знак что либо абстракция плохо продумана, либо реальность плохо ложится в предложенную форму.

Пример нарушения: `List` имеет O(1) `add(element)`, но `remove(element)` — O(n) (линейный поиск + сдвиг). Асимметрия. Пользователю List кажется что операции равноправны, а фактически они разного порядка сложности. Осторожность нужна, документация должна отражать реальную стоимость.

Иногда асимметрия неизбежна (природа задачи), но должна быть явной — не «скрытой» за одинаковыми названиями.

**Правило одного уровня абстракции**. Метод должен работать на одном уровне. Смешивание уровней — anti-pattern:

```java
// Плохо: смешаны уровни
void processOrder(OrderRequest req) {
    Order order = new Order(req);
    orderRepo.save(order);
    
    try (Connection conn = dataSource.getConnection()) {   // низкий уровень
        try (PreparedStatement ps = conn.prepareStatement("UPDATE ...")) {
            ps.setLong(1, order.getId());
            ps.executeUpdate();
        }
    }
    
    email.send(order.getCustomerEmail(), "Order confirmed");   // высокий уровень
}

// Хорошо: один уровень
void processOrder(OrderRequest req) {
    Order order = createOrder(req);
    saveOrder(order);
    updateInventory(order);
    sendConfirmation(order);
}
```

Второй метод читается как оглавление. Каждый шаг — на том же уровне бизнес-логики. JDBC-детали спрятаны в `updateInventory`.

**Правило имени**. Имя абстракции должно описывать **что** она предоставляет, не **как** реализована. `OrderRepository` (что) лучше `PostgresOrderDAO` (как). Клиент не должен знать что там Postgres — иначе замена БД сломает клиентский код (или хотя бы не соответствие смысла и имени).

Тот же принцип для методов. `getUsers()` (что) лучше `queryUsersFromDatabase()` (как).

**Правило «одна реализация → нет интерфейса»**. Если единственная имплементация — обычно интерфейс не нужен. Java-энтерпрайз культура часто перегибает: `UserService interface + UserServiceImpl class` с единственной реализацией. Интерфейс тут не даёт никакой ценности — просто boilerplate.

Исключения. **Мockability для unit tests** сама по себе — недостаточная причина (Mockito умеет mock конкретных классов, если не final). **Реальная possibility замены** — например Repository имеет одну JPA реализацию сейчас, но планируется добавить кэширующую версию — тогда интерфейс оправдан.

**Правило времени жизни**. Не абстрагируйся преждевременно. Абстракция сделанная до понимания требований почти всегда неправильно проведена. Терпи первую версию с прямым кодом. Когда появятся 2-3 варианта — реальная ось изменений станет видна, тогда абстрагируй.

## Эволюция абстракций во времени

Абстракции не статичны. Хорошая абстракция сегодня может стать плохой через 2 года, потому что домен изменился. Понимать это — часть зрелости инженера.

**Fenwick's principle**: абстракции стареют. Они создаются под конкретный набор известных на момент создания требований. Новые требования могут либо ложиться на существующую абстракцию, либо конфликтовать с ней.

Признаки старения абстракции:

- **Растущее число параметров**. Метод раньше был `sendEmail(to, subject, body)`, стал `sendEmail(to, from, cc, bcc, subject, body, attachments, priority, replyTo, headers)`. Клиенты вынуждены передавать много null'ов. Абстракция не масштабируется на новые требования.
- **Специальные случаи через флаги**. `processOrder(order, isRefund, isB2B, isSubscription, ...)`. Boolean-параметры — красный флаг. Часто нужно разделить на несколько операций.
- **Условная логика вокруг типа объекта**. `if (payment instanceof CryptoPayment)` в клиентском коде — иерархия не покрывает новую potребность. Либо добавить в абстракцию, либо признать что варианты стали слишком разными.
- **Documentation-driven usage**. Клиенты не могут использовать абстракцию без документации что делать в каком случае. Хороший API self-documenting.

**Рефакторинг абстракции**:

- **Split abstraction** — одна абстракция превращается в две. `Notification` разделён на `EmailNotification` и `PushNotification` с разными интерфейсами.
- **Merge abstractions** — две абстракции сливаются когда outshine become the same. Редко, обычно противоположный тренд.
- **Replace abstraction** — старая заменяется новой. Обычно с миграцией всех клиентов (dual-write, gradual rollout).
- **Extend abstraction** — добавить методы для новых кейсов, старые клиенты не ломаются. Java 8 default methods помогают.

**Тактика миграции**. Не менять существующую абстракцию сразу. Стратегия:

1. Ввести **новую** абстракцию рядом со старой.
2. Постепенно мигрировать клиентов на новую (по одному, с тестами).
3. Когда все мигрированы — старая помечается @Deprecated.
4. Через версию — удаление.

Быстрая замена «одним PR» на большой codebase — почти всегда провал: пропущенные клиенты, регрессии, откаты.

## Специфика Java для абстракций

Разберём инструменты Java глубже.

**extends vs implements**. `extends` — наследование класса от класса или интерфейса от интерфейса. `implements` — реализация интерфейса классом. Множественное наследование классов запрещено (diamond problem), множественная реализация интерфейсов разрешена.

С Java 8 default methods в интерфейсах формально дали «multiple inheritance of behavior» через интерфейсы. Diamond problem решается компилятором — если два интерфейса имеют конфликтующие default методы, класс обязан явно переопределить (`Foo.super.method()` или `Bar.super.method()`).

**Default methods** (Java 8+) — можно добавлять методы с реализацией к интерфейсу без ломания клиентов. Идиома для evolvable interfaces:

```java
interface Collection<E> {
    // Старые методы...
    
    default Stream<E> stream() {   // Добавлено в Java 8
        return StreamSupport.stream(spliterator(), false);
    }
}
```

Все существующие реализации (ArrayList, HashSet, etc.) получили `stream()` без изменений. Backward compatible evolution — правильное применение default methods.

Anti-pattern: default methods как замена abstract методам. Interface должен определять **обязательный** контракт. Default methods — для optional / evolutionary добавлений.

**Sealed classes / interfaces** (Java 17+) — controlled abstraction. Явный список permitted subclasses, компилятор гарантирует что других нет:

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
final class Circle implements Shape { ... }
final class Rectangle implements Shape { ... }
final class Triangle implements Shape { ... }

// Компилятор проверяет exhaustive
String describe(Shape s) {
    return switch (s) {
        case Circle c -> "circle with radius " + c.radius();
        case Rectangle r -> "rectangle " + r.width() + "x" + r.height();
        case Triangle t -> "triangle";
        // Компилятор enforces: если добавится новый Shape, здесь ошибка
    };
}
```

Sealed hierarchies — правильный инструмент для **ADT** (Algebraic Data Types). `Result<T>` = `Success<T> | Failure`, `Option<T>` = `Some<T> | None`. Компилятор помогает не пропустить случаи.

Разница с обычной hierarchy: обычная — **open set** (кто угодно может extends), sealed — **closed set** (только permitted). Открытость даёт extensibility (плюс полиморфизма), закрытость даёт exhaustiveness (плюс pattern matching).

**Records** (Java 16+) — immutable value objects с автоматическими equals, hashCode, toString, accessors:

```java
record Money(BigDecimal amount, Currency currency) {
    // Compact constructor — валидация
    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.signum() < 0)
            throw new IllegalArgumentException("Money cannot be negative");
    }
    
    // Кастомные методы
    public Money add(Money other) {
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
        return new Money(amount.add(other.amount), currency);
    }
    
    // Static factory
    public static Money of(String amount, String currencyCode) {
        return new Money(new BigDecimal(amount), Currency.getInstance(currencyCode));
    }
}
```

Ограничения records: implicit final (нельзя extends), все поля final (нельзя иметь mutable state), нельзя extends other class (только implement interfaces). Идеально для value objects, DTO, event payloads. Плохо для entities с identity + mutable state.

**Immutable collections** (`List.of()`, `Set.of()`, `Map.of()`, Java 9+) — правильная абстракция read-only коллекции. Клиент не может мутировать. Возврат `List.of(...)` из метода — чёткий сигнал «эту коллекцию менять нельзя», в отличие от `Collections.unmodifiableList(new ArrayList<>(items))` который создаёт view, но исходный ArrayList можно менять.

**Functional interfaces** (`Function<T, R>`, `Consumer<T>`, `Supplier<T>`, `Predicate<T>`, `BiFunction<T, U, R>` и т.д.) — абстракция «функция как first-class citizen». Passable в методы, composable через `.andThen`, `.compose`, `.and`, `.or`.

```java
Function<String, Integer> length = String::length;
Function<Integer, Integer> square = x -> x * x;
Function<String, Integer> lengthSquared = length.andThen(square);

lengthSquared.apply("hello");   // 25 (length=5, squared=25)
```

Абстрактный уровень — не сам класс, а поведение. Позволяет высокоуровневые операции (`stream().map(fn).filter(pred).collect(...)`) без создания отдельных классов для каждого маленького преобразования.

**Generic types и bounded wildcards**. Абстракция «работает с типом X, но X может быть чем-то более specific»:

```java
// Метод принимает любую коллекцию Number или его подтипов
double sum(Collection<? extends Number> nums) {
    return nums.stream().mapToDouble(Number::doubleValue).sum();
}

// Метод принимает коллекцию куда можно класть Integer или его супертипы
void addIntegers(Collection<? super Integer> coll) {
    for (int i = 1; i <= 10; i++) coll.add(i);
}
```

PECS (Producer Extends Consumer Super) — правило когда какой wildcard использовать. Producer (мы берём из коллекции) — `? extends X`. Consumer (мы кладём в коллекцию) — `? super X`.

Type erasure в Java (в отличие от reified generics в C#) — компромисс совместимости. `List<String>` и `List<Integer>` в runtime — одинаковые `List`. Ловушки: нельзя `instanceof List<String>`, нельзя `new List<String>[10]`, нельзя `catch (MyException<Foo> e)`.

## Заключение

Абстракция — процесс выделения существенного и сокрытия несущественного. Не бесплатна — платишь runtime overhead (indirection), cognitive load (больше уровней для понимания), rigidity при wrong abstraction. Хорошая абстракция окупается изоляцией изменений, тестируемостью, ясностью намерения.

**Уровни абстракции** — от машинных инструкций до бизнес-домена. Хороший код работает на одном уровне за раз. Смешивание — признак утечки.

**Инструменты в языках**: функция, класс, interface, abstract class, module, generic type, sealed class (Java 17+), record (Java 16+). Каждый со своей семантикой, выбирать осознанно.

**ADT (Abstract Data Type)** — теоретическая база. Операции + аксиомы, независимо от реализации.

**Cost of abstraction**: runtime overhead (indirection, allocations, virtual calls — JIT многое устраняет через inlining); cognitive load (больше уровней сложнее держать в голове); rigidity при wrong abstraction.

**Leaky abstractions** (Spolsky): все нетривиальные абстракции в какой-то степени leaky. Детали реализации просачиваются — TCP latency, ORM N+1, HTTP timeouts. Абстракции экономят время, но не время на обучение.

**Wrong abstraction** (Metz): duplication is far cheaper than the wrong abstraction. Правило трёх — терпи дублирование до третьего повторения.

**Anti-patterns**: over-abstraction («на всякий случай»), premature abstraction (до понимания требований), God interface, speculative generality, wrong hierarchy (наследование вместо композиции).

**SOLID** глубоко:
- **SRP** — одна причина меняться, привязка к stakeholder'у. Классические нарушения — utility classes, fat controllers, anemic domain.
- **OCP** — открыто для расширения через полиморфизм, закрыто для изменения. Гнётся при малых стабильных иерархиях (sealed + switch лучше).
- **LSP** — subtype выполняет контракт supertype (предусловия, постусловия, инварианты, история). Классика — Square/Rectangle. Fix — immutability или разделение иерархии.
- **ISP** — маленькие focused интерфейсы. Клиент не должен зависеть от неиспользуемых методов. В Java >5-7 методов = задуматься о разбиении.
- **DIP** — direction of dependencies. Абстракция принадлежит high-level слою, реализация зависит от абстракции. Фундамент DI, hexagonal architecture, тестируемости.

**Композиция vs наследование**: по умолчанию композиция. Наследование только при true is-a + честном выполнении контракта. Fragile base class problem, LSP-ловушки, единственное наследование в Java — всё против неаккуратного использования наследования.

**Паттерны абстракции** с механикой и trade-offs: Template Method (жёсткий скелет + hooks), Strategy (композиция взаимозаменяемых алгоритмов, Java 8+ lambda-friendly), Adapter (переходник форм), Bridge (две независимо развивающиеся иерархии), Repository (абстракция persistence в domain), Layers с DIP, Hexagonal / Ports & Adapters (домен в центре, порты + adapters).

**DDD tactical patterns**:
- **Bounded Context** — разные модели для разных задач, не одна God-модель. Микросервисы должны следовать.
- **Aggregate** — транзакционная граница + инварианты. Aggregate root как единственная точка входа. Ссылки между aggregates только по ID.
- **Entity vs Value Object** — identity vs value equality. Records в Java 16+ идеальны для VO.
- **Domain Service** — операции между сущностями. Отличать от application service (orchestrator).
- **Domain Event** — важные факты домена. Публикация через outbox pattern.
- **Anti-Corruption Layer** — защита домена от внешнего/legacy хаоса.

**Практические правила расширенно**: правило трёх (Sandi Metz), симметрия, один уровень абстракции, имя (что не как), один интерфейс = одна реализация обычно = интерфейс не нужен, время жизни (не преждевременно).

**Эволюция абстракций во времени** — они стареют. Признаки: растущее число параметров, boolean flags, instanceof в клиентском коде. Тактика миграции — новая рядом со старой, постепенный переход, deprecation, удаление.

**Java specifics**: extends vs implements + multiple inheritance of behavior через default methods; sealed для controlled abstractions (ADT); records для value objects; immutable collections; functional interfaces как first-class functions; generics + PECS.

Правильная абстракция — не «сделать красиво», а сделать так, чтобы **изменения ложились по границам абстракции**. Если каждое изменение требует трогать и абстракцию, и реализацию, и клиента — граница неверна. Если изменения ложатся на одну сторону границы за раз — абстракция работает.

Facade и Proxy детально в 85. Design patterns Facade/Proxy там же. Здесь была концепция абстракции: определение, стоимость, хорошая vs плохая, SOLID глубоко, паттерны с trade-offs, DDD tactical, композиция vs наследование, эволюция во времени, Java specifics.

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
