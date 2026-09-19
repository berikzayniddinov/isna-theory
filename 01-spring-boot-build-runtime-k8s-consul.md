# 01. Полный стек Spring Boot микросервиса: от кода до продакшна

## С чего начинается понимание

Работая senior-инженером над микросервисной архитектурой вроде КНП, ты каждый день имеешь дело с длинной цепочкой технологий. Пишешь Java-код в IDE. Собираешь через Gradle. Получаешь fat JAR, содержащий приложение и все его зависимости. Заворачиваешь в Docker image. Деплоишь в Kubernetes. Приложение регистрируется в Consul для service discovery, начинает принимать HTTP-запросы через nginx-ingress, общается с PostgreSQL, RabbitMQ, MinIO, Keycloak. Каждый элемент этой цепочки — своя большая тема, но вместе они формируют один цельный стек.

Большинство разработчиков знают эту цепочку фрагментарно. Могут поправить Spring-код и Gradle-конфиг, но плохо понимают что происходит между "Ассертиум push origin master" и "приложение отвечает пользователю". Слои сливаются в неразличимую магию: gitlab-ci что-то делает, Docker что-то делает, K8s что-то делает, работает. Пока работает — всё хорошо. Когда что-то ломается — начинается угадывание.

Senior отличается от middle тем, что понимает каждый слой этого стека отдельно, знает как они соединены, и может отладить проблему на любом уровне. В этом файле мы пройдём этот путь от начала до конца, не углубляясь в детали каждой темы (для этого есть последующие файлы 02-100), но связывая всё в цельную картину. Что реально происходит внутри `.java` файлов при компиляции. Как Gradle организует сборку. Что такое fat JAR и почему Spring Boot делает JAR-в-JAR. Как приложение поднимается при `java -jar`. Как Docker его контейнеризирует. Как K8s его оркестрирует. Как Consul обеспечивает service discovery. И как всё это сложено вместе на реальном примере обработки запроса пользователя.

## Java как виртуальная машина: базовый принцип

Прежде чем говорить про Spring Boot или JAR, нужно понимать один фундаментальный принцип, отличающий Java от C++ и большинства native языков. В C++ ты пишешь код, компилятор превращает его в машинный код конкретной архитектуры процессора (x86-64, ARM), получаешь executable, который работает только на этой архитектуре и этой операционной системе. Хочешь запустить на другой платформе — пересобирай с нуля.

Java изначально была спроектирована для web-эры, когда программа могла работать на разных типах устройств. Решение: не компилировать в машинный код, а в промежуточное представление, называемое **байткодом**. Байткод — это последовательность инструкций для абстрактного виртуального процессора, у которого нет физической реализации. Он существует только как спецификация — Java Virtual Machine (JVM) её реализует программно.

Цепочка получается такая. Ты пишешь `.java` файлы. Компилятор `javac` превращает их в `.class` файлы, содержащие байткод. Один `.java` = один `.class` (обычно, кроме внутренних классов). JVM запускается на конкретной ОС (Windows, Linux, macOS) и конкретной архитектуре (x86, ARM), но она умеет читать любой байткод одинаково — потому что JVM работает с абстрактной моделью выполнения, а не с реальным процессором. Один `.class` можно скопировать с Windows на Linux, с x86 на ARM, и он будет работать (если есть подходящая JVM).

Это принцип "write once, run anywhere". За него платят производительностью: между байткодом и реальным процессором есть слой интерпретации. Но JVM решает эту проблему через **JIT (Just-In-Time compilation)** — на лету компилирует часто выполняемый байткод в реальный машинный код процессора, кэширует его, использует для последующих вызовов. После нескольких минут прогрева Java-приложение работает почти как native code.

Из этого разделения возникает несколько понятий, которые часто путают. **JVM** — виртуальная машина, движок исполнения байткода. **JRE (Java Runtime Environment)** — JVM плюс стандартная библиотека классов (`java.lang`, `java.util`, `java.io`). Достаточно чтобы запустить готовое приложение. **JDK (Java Development Kit)** — JRE плюс инструменты разработчика: компилятор `javac`, упаковщик `jar`, отладчик, `jshell`, диагностические утилиты. С JDK ты и разрабатываешь, и запускаешь. Начиная с Java 11 отдельный JRE больше не поставляется — только JDK.

В КНП везде используется Amazon Corretto — открытая реализация OpenJDK от Amazon с прод-качеством и long-term support. Corretto 11 для сервисов на master ветке, Corretto 21 для новых модулей на master-21. Одна нода K8s может иметь любое количество версий JDK, потому что каждый контейнер приносит свою через Docker image (`FROM amazoncorretto:21-alpine`). Приложения не конфликтуют.

## От Java-кода к JAR-архиву

Когда у тебя есть коллекция `.java` файлов (тысячи в реальном проекте), их нужно организовать в что-то запускаемое. Промежуточное представление — `.class` файлы, но напрямую их деплоить неудобно: множество файлов в определённой структуре директорий, легко потерять один, сложно распространять.

Стандартный способ упаковки Java-приложений — **JAR (Java ARchive)**. Технически это обычный ZIP-архив с определённой структурой. Внутри — иерархия папок, отражающая packages Java-кода, `.class` файлы в них, ресурсы (`.properties`, `.yml`, `.xml`) в правильных местах, и специальный файл `META-INF/MANIFEST.MF` с метаданными.

MANIFEST — это текстовый файл в формате key-value, описывающий что за архив. Минимальный манифест содержит только версию формата. Executable JAR (тот, что можно запустить через `java -jar`) содержит ключ `Main-Class`, указывающий класс со статическим методом `main`. Когда ты выполняешь `java -jar myapp.jar`, JVM открывает архив как ZIP, читает манифест, находит имя главного класса, загружает его, вызывает `main` — и с этого начинается выполнение твоего кода.

Это работает до тех пор, пока твоё приложение не зависит от внешних библиотек. Реальное приложение зависит от сотен: Spring Framework, Jackson для JSON, Hibernate для ORM, PostgreSQL JDBC driver, Apache Commons, и десятки других. Каждая — свой JAR. Классический подход — держать зависимости отдельно, в директории `lib/`, указывать при запуске:

```
java -cp "myapp.jar:lib/spring-core.jar:lib/spring-web.jar:..." com.example.Main
```

Неудобно и хрупко. Забыл добавить `lib/`, забыл обновить classpath при новой библиотеке, конфликты версий разных зависимостей — источники бесконечных проблем. Spring Boot изменил ситуацию, введя концепцию **fat JAR** или **uber JAR**.

## Fat JAR и Spring Boot's подход

Идея fat JAR простая: положить в один архив и твой код, и все его зависимости, целиком. Один файл, никаких external classpath — просто `java -jar myapp.jar` и всё работает. Разные инструменты реализуют это по-разному.

Наивный подход, реализованный Gradle Shadow plugin и Maven Shade plugin, — распаковать все JAR-зависимости, слить их классы в один архив. Такой uber JAR содержит все classes подряд: твой код в `com.example`, Spring в `org.springframework`, Jackson в `com.fasterxml.jackson`. Работает, но создаёт проблемы. Разные библиотеки могут иметь конфликтующие ресурсы в `META-INF/services` (Java Service Loader mechanism) — при слиянии одни затираются другими. Разные JAR могут иметь `META-INF/spring.factories` — тоже конфликты. Нужны специальные merge-стратегии, они бывают ошибочны.

Spring Boot придумал более элегантное решение: **JAR-in-JAR**. В fat JAR зависимости остаются целыми JAR-файлами, положенными в специальную директорию `BOOT-INF/lib/`. Твой код лежит отдельно в `BOOT-INF/classes/`. Никаких коллизий имён — каждый JAR живёт в своём пространстве. Никаких проблем с META-INF файлами — они остаются внутри своих JAR-ов.

Проблема этой архитектуры — стандартный JVM classloader не умеет читать классы из JAR внутри JAR. Стандартный `URLClassLoader` понимает `jar:file:/home/app/lib/spring.jar!/` (класс внутри одного JAR), но не понимает `jar:file:/home/app/myapp.jar!/BOOT-INF/lib/spring.jar!/` (класс во вложенном JAR, двойной `!`). Обычный JVM просто не находит классы.

Spring Boot решает это через custom classloader — **LaunchedURLClassLoader**. Он расширяет стандартный, регистрирует свой `URLStreamHandler` для протокола `jar:`, умеет парсить цепочки `!/...!/` и открывать вложенные ZIP-стримы. Это специализированный код, часть Spring Boot loader.

Отсюда специфическая структура MANIFEST.MF Spring Boot JAR-а. Главный класс — не твой:

```
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: kz.gov.kgd.isna.knp.KnpApplication
Spring-Boot-Version: 2.2.4.RELEASE
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
```

`Main-Class` указывает на `JarLauncher` из Spring Boot loader. `Start-Class` — это уже твой класс с `main` методом. Когда ты запускаешь JAR через `java -jar`, JVM видит `Main-Class`, вызывает JarLauncher. Тот регистрирует LaunchedURLClassLoader как system classloader. Через него загружает `Start-Class`, вызывает его `main`. С этого момента твой код исполняется в среде, где classloader знает про JAR-в-JAR, и Spring работает нормально.

Начиная с Spring Boot 2.3 появилась ещё одна оптимизация — **layered JAR**. Fat JAR разбивается на логические слои: dependencies (редко меняются), snapshot-dependencies (SNAPSHOT версии, тоже редко), spring-boot-loader (стабильный), application (твой код, часто меняется). В manifest добавляется `layers.idx` описывающий эти слои. Утилита `layertools` может извлечь их в отдельные директории. В Dockerfile это позволяет положить каждый слой в отдельный Docker layer — при пересборке меняется только application layer, dependencies остаются кэшированными. Существенно ускоряет пересборку и деплой Docker images.

## Gradle как система сборки

Между `.java` файлами и готовым fat JAR — большая работа. Скачать все зависимости из внешних репозиториев. Скомпилировать `.java` в `.class`. Прогнать unit-тесты. Собрать ресурсы. Упаковать в JAR правильной структуры. Опубликовать в артефакт-репозиторий. Ручной труд невозможен, нужен инструмент.

В Java-мире два основных инструмента: **Maven** и **Gradle**. Maven старше, использует XML-конфигурацию (pom.xml), строже, конвенциональнее. Gradle моложе, использует Groovy или Kotlin DSL для конфигурации, гибче, быстрее (кэширование, инкрементальная сборка, демон). В КНП везде Gradle.

Gradle-проект имеет стандартную структуру, унаследованную от Maven. `settings.gradle` в корне определяет какие есть подпроекты (модули). `build.gradle` описывает как собирать конкретный проект: какие плагины подключить, откуда брать зависимости, какие есть зависимости, кастомные задачи. `gradlew` (и `gradlew.bat` для Windows) — обёртка (wrapper), которая автоматически скачивает нужную версию Gradle из указанной в `gradle-wrapper.properties`. Использование wrapper обязательно: гарантирует, что все разработчики и CI используют одну и ту же версию Gradle.

Исходники Java лежат в `src/main/java`, ресурсы в `src/main/resources`. Тесты соответственно в `src/test/java` и `src/test/resources`. Это convention, встроенная в Gradle Java plugin. Меняется, но обычно не имеет смысла.

Ключевая абстракция Gradle — **task** (задача). Компиляция — задача. Тесты — задача. Упаковка JAR — задача. Публикация — задача. Задачи имеют inputs (что нужно чтобы выполнить) и outputs (что производят). Зависимости между задачами формируют направленный ациклический граф (DAG). Когда пользователь запускает конкретную задачу (например `./gradlew build`), Gradle находит все задачи от которых `build` зависит транзитивно, топологически сортирует, выполняет по порядку.

Огромная часть эффективности Gradle — **incremental build**. Gradle помнит inputs/outputs каждой задачи в кэше. Если inputs не изменились с прошлого запуска, задача помечается как UP-TO-DATE и не выполняется — outputs берутся из кэша. Это делает повторные сборки очень быстрыми: только затронутые изменениями задачи реально выполняются. `./gradlew build --info` показывает какие задачи cached, какие executed, какие skipped.

Gradle работает через **daemon** — фоновый JVM-процесс, живущий между вызовами `./gradlew`. Первый запуск в сессии долгий (JVM стартует, конфигурация читается), последующие — быстрые (daemon уже прогрет). Это нормально; убить если что-то пошло не так: `./gradlew --stop`.

## Управление зависимостями

Одна из главных функций Gradle — управление зависимостями. Приложение зависит от библиотек, библиотеки — от других библиотек (транзитивные зависимости). Gradle должен разрешить весь этот граф, скачать нужные JAR из репозиториев, кэшировать локально.

Зависимости объявляются в блоке `dependencies` с разными **конфигурациями** (scopes), которые определяют где именно эта зависимость нужна. `implementation` — на compile и runtime, но скрыта от потребителей этого проекта (используется в 90% случаев). `api` — то же самое, но пробрасывается транзитивно наружу — только для случаев когда тип из зависимости фигурирует в публичном API. `compileOnly` — только при компиляции, не в runtime (Lombok — компилятор его нужен для обработки аннотаций, а в runtime сгенерированный код не требует Lombok). `runtimeOnly` — только в runtime (PostgreSQL JDBC driver — код общается через JDBC интерфейсы, реальная реализация подключается runtime). `testImplementation` — только для тестов.

Разница `implementation` и `api` тонкая, но важная для многомодульных проектов. Если модуль A имеет зависимость `implementation 'org.example:library'`, эта зависимость доступна коду A, но не пробрасывается модулям, зависящим от A. Пользователь A не знает про library, изменение library не требует пересобирать пользователей A. Это ускоряет пересборку и уменьшает "coupling". Если бы было `api`, library была бы транзитивно видна пользователям — если бы им это нужно (например, они используют классы из library) — то это правильно; если нет — implementation экономит.

Транзитивные зависимости неизбежно приводят к конфликтам версий. Библиотека X требует `guava:30`, библиотека Y требует `guava:20`. В classpath может быть только одна версия. Gradle по умолчанию использует стратегию "highest wins" — берёт самую свежую из запрошенных. Иногда это ломает Y, если она использовала методы, изменившиеся в новой guava.

Для стабильности используется **BOM (Bill of Materials)** — набор совместимых версий, поддерживаемых как единое целое. Spring Boot публикует свой BOM `spring-boot-dependencies`, где перечислены все совместимые версии всех библиотек экосистемы. Импортируешь его — все версии подтянутся согласованными. В КНП есть собственный BOM `isna-global` (и `isna-global-21` для Java 21 версии), где фиксированы версии внутренних библиотек. Модули пишут зависимости без явных версий, версии приходят из BOM.

Репозитории — откуда Gradle скачивает JAR-ы. `mavenCentral()` — публичный Maven Central. `maven { url '...' }` — произвольный. В enterprise обычно всё идёт через внутренний Nexus/Artifactory, который проксирует Maven Central и хранит собственные артефакты. Плюс: контроль (что можно, что нельзя), кэш (быстрее), возможность работать без внешнего интернета. В КНП внутренний Nexus — `nexus.isna.internal`.

## Spring Framework: Inversion of Control

Итак, у нас есть Java-код, скомпилированный в байткод, упакованный в fat JAR, готовый к запуску. Что этот код делает? В типичном enterprise приложении — это Spring Boot приложение, использующее Spring Framework как фундамент.

Spring Framework — библиотека для управления объектами и их зависимостями. Ключевая идея — **Inversion of Control (IoC)**. В обычном коде ты сам создаёшь объекты, знаешь как их конструировать, помнишь порядок инициализации. С IoC контейнер создаёт объекты за тебя, ты только описываешь: какие мне нужны компоненты и что от чего зависит. Контейнер сам разбирается.

Практическая реализация IoC в Spring — **Dependency Injection (DI)**. Твой класс объявляет что ему нужно (через параметры конструктора или поля), Spring подсовывает нужные зависимости. Пример без Spring:

```java
class KnpService {
    private final FnoRepository repo;
    public KnpService() {
        DataSource ds = new HikariDataSource(readConfig());
        Database db = new Database(ds);
        this.repo = new FnoRepository(db);
    }
}
```

Класс знает про Hikari, конфиг файл, Database. Поменять DataSource на другую реализацию — правки везде. Тестировать невозможно — конструктор подтягивает реальную БД. С Spring:

```java
@Service
class KnpService {
    private final FnoRepository repo;
    public KnpService(FnoRepository repo) {
        this.repo = repo;
    }
}
```

Класс не знает откуда репозиторий — просто принимает через конструктор. Spring, глядя на аннотацию `@Service`, знает "это компонент, надо создать при старте". Смотрит конструктор, видит зависимость на `FnoRepository`, создаёт его (тоже помеченный `@Component`/`@Repository`), передаёт в конструктор `KnpService`. Ты не пишешь `new`, только описываешь зависимости.

Плюсы очевидны. Слабая связанность — сервис не знает про реализацию репозитория, можно поменять без правок. Тестирование — легко создать `new KnpService(mockRepo)`. Единая точка настройки — Spring один раз собирает граф зависимостей. Переиспользуемость — один репозиторий используется многими сервисами, каждый получает тот же экземпляр.

Spring различает три способа инъекции. **Constructor injection** — через параметры конструктора, как выше. **Setter injection** — через отдельные setter-методы, аннотированные `@Autowired`. **Field injection** — прямо в поля с `@Autowired`. Constructor injection — правильный современный выбор: поля могут быть `final` (immutable), зависимости обязательны (класс не создастся без них), явно видно что нужно (в сигнатуре конструктора), легко тестировать. Setter — только для опциональных зависимостей или разруливания циклов. Field — плохо, потому что скрывает зависимости, требует Spring для работы, поля не могут быть final.

## Bean lifecycle и ApplicationContext

Объект, находящийся под управлением Spring, называется **bean** (компонент, боб). Это не просто объект — Spring создаёт его, внедряет зависимости, вызывает lifecycle-хуки, следит за scope. Контейнер, содержащий все beans приложения, — **ApplicationContext**.

При старте Spring делает несколько шагов. Читает `@Configuration` классы и `@ComponentScan` пути. Сканирует classpath, находит все классы с аннотациями стереотипов (`@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`, `@Configuration`) — все они кандидаты в beans. Строит граф зависимостей: кто от кого зависит. Топологически сортирует. Создаёт beans в порядке (сначала независимые, потом зависящие от них).

Для каждого bean lifecycle такой. Инстанцирование — вызов конструктора (или `@Bean`-метода в `@Configuration`). Populate properties — если есть `@Autowired` поля или сеттеры, они заполняются. Aware интерфейсы — если bean реализует `ApplicationContextAware` или подобные, Spring подсовывает нужное. `@PostConstruct` метод (если есть) — для инициализации после того как все зависимости внедрены. BeanPostProcessor — Spring может обернуть bean в proxy (для `@Transactional`, `@Async` и других AOP-аспектов). После этого bean готов, живёт в контексте, используется. При shutdown — `@PreDestroy` метод для cleanup.

`@PostConstruct` полезен для прогрева кэшей, валидации конфигурации, регистрации во внешних системах. `@PreDestroy` — для graceful shutdown: закрыть connection pool, деregister из discovery, отправить последнюю партию сообщений.

Beans имеют **scope** — сколько времени живут. По умолчанию **singleton**: один экземпляр на весь ApplicationContext, все получают тот же. Это правильно для сервисов, репозиториев, конфигураций — не нужно много копий одного и того же. **Prototype** — новый экземпляр на каждый запрос (`ctx.getBean()` или `@Autowired`). Полезно для stateful объектов, которые не должны шариться. **Request** и **session** — для web-приложений, живут в контексте HTTP запроса/сессии. Редко используются в микросервисах.

Важный gotcha: инъекция prototype bean'а в singleton даёт всегда тот же экземпляр (первый запрошенный). Если действительно нужен новый экземпляр на каждое использование, надо использовать `ObjectProvider` или method-level `@Lookup`.

## Аннотации-стереотипы

Все аннотации, помечающие класс как Spring bean, называются **стереотипы**. Функционально почти одинаковы, но с семантическими нюансами.

`@Component` — базовая. Просто "я bean". Используется когда никакая другая семантика не подходит.

`@Service` — бизнес-логика, application services. Функционально тот же `@Component`, но семантика говорит "это сервисный слой".

`@Repository` — доступ к данным. Дополнительно: Spring подключает `PersistenceExceptionTranslationPostProcessor`, оборачивающий низкоуровневые исключения (SQLException, HibernateException) в Spring-специфичные `DataAccessException`. Это позволяет писать catch-блоки универсально, независимо от нижней технологии.

`@Controller` — MVC-контроллер. Возвращает view names (для рендеринга через template engine). В микросервисах редко.

`@RestController` — `@Controller` + `@ResponseBody` на всех методах. Возвращает JSON/XML, не view. Стандарт для REST-сервисов.

`@Configuration` — класс с `@Bean`-методами. Особая обработка: Spring оборачивает такой класс в CGLIB proxy, чтобы гарантировать singleton при вызовах `@Bean`-методов внутри класса.

Для `@Configuration` proxying важно понимать. В таком классе `@Bean` метод вроде `dataSource()` может вызываться из другого `@Bean`-метода, например `repository()`. Без proxy каждый такой вызов создавал бы новый экземпляр — два разных DataSource. Через proxy Spring перехватывает вызовы `@Bean`-методов и возвращает singleton, зарегистрированный в контексте.

## Spring Boot: конвенции и auto-configuration

Spring Framework сам по себе — гибкий конструктор. Настроить всё вручную (XML или явные `@Bean` методы) — недели работы. **Spring Boot** — набор надстроек, делающих настройку близкой к нулевой через "convention over configuration".

Ключевые компоненты Spring Boot.

**Starters** — метапакеты зависимостей. `spring-boot-starter-web` — не один artifact, а BOM, тянущий за собой Spring MVC, Jackson (JSON), встроенный Tomcat, valid, logging. Одна dependency в build.gradle — работающий web-сервер. `spring-boot-starter-data-jpa` — Hibernate, Spring Data JPA, connection pool, JDBC. Стартеры — стандартный способ добавить функциональность.

**Auto-configuration** — умный механизм настройки. Spring Boot содержит сотни классов `*AutoConfiguration`, каждый настраивает определённую технологию. Они помечены conditional-аннотациями: `@ConditionalOnClass(DataSource.class)` — включиться если DataSource есть в classpath (значит нужен JDBC), `@ConditionalOnMissingBean(DataSource.class)` — но только если пользователь сам не определил свой. Комбинация этих условий делает так, что auto-configuration работает "из коробки", но не мешает если ты хочешь свою конфигурацию.

Список auto-configuration классов раньше хранился в `META-INF/spring.factories` (Boot 2), сейчас в `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 3). Spring Boot читает эти файлы из всех JAR-ов в classpath, собирает список авто-конфигураций, применяет их с учётом условий.

Отключить конкретную auto-configuration — через `spring.autoconfigure.exclude` в конфиге или `@SpringBootApplication(exclude = ...)` в коде. Пример из КНП: `knp-form-hz5-actuator-cache-nosuchmethod` — при обновлении на Hazelcast 5 CacheMetricsAutoConfiguration и HazelcastHealthContributorAutoConfiguration ссылались на несуществующий метод, пришлось явно отключить.

**Embedded server** — HTTP-сервер (Tomcat, Jetty, Undertow) встроен внутрь fat JAR. Приложение — само себе сервер. При старте JVM Spring Boot запускает embedded Tomcat на настроенном порту (по умолчанию 8080). Никаких external application server, никакого WAR деплоя. Это радикально упростило микросервисную архитектуру.

По умолчанию Tomcat. Можно исключить и подключить Jetty или Undertow через exclude в build.gradle. Undertow часто предпочтительнее в high-load сценариях: легче по памяти, лучше по пропускной способности. Reactive-стек (WebFlux) использует Netty.

**application.yml/properties** — единый файл конфигурации. Все настройки Spring Boot и приложения — в одном месте, читаются автоматически. Секция `server.*` — про Tomcat. `spring.datasource.*` — про БД. `logging.*` — про логи. `management.*` — про Actuator. Плюс твои специфичные ключи для бизнес-логики.

**Profiles** — набор конфигураций для разных окружений. `application-dev.yml`, `application-prod.yml` — подхватываются автоматически при `--spring.profiles.active=prod`. В классах — `@Profile("prod")` для условной регистрации beans. В КНП: dev, test, preprod, prod, local — стандартный набор.

**Actuator** — endpoints для мониторинга: `/actuator/health` — жив ли, `/actuator/metrics` — метрики, `/actuator/prometheus` — Prometheus-formatted, `/actuator/env` — конфигурация. Основа для health-check в Kubernetes и Consul, для сбора метрик Prometheus.

## Запуск приложения: полная цепочка

Теперь соберём всё в цепочку — что происходит от `java -jar knp-app.jar` до момента "приложение отвечает на HTTP-запросы".

Первое — JVM стартует. Кернел загружает Java executable, JVM инициализируется, читает системные properties, настраивает heap, готовит classloading.

JVM открывает JAR как ZIP, находит `META-INF/MANIFEST.MF`, читает `Main-Class`. У Spring Boot fat JAR это `JarLauncher`. JVM загружает класс через свой system classloader, вызывает `JarLauncher.main()`.

JarLauncher выполняет специальный setup. Регистрирует `LaunchedURLClassLoader` как system classloader — теперь JVM умеет читать классы из JAR-в-JAR. Читает `Start-Class` из манифеста — это `KnpApplication`. Загружает через новый classloader, вызывает через reflection его `main` метод.

Управление переходит в `KnpApplication.main(args)`. Стандартный код:

```java
public static void main(String[] args) {
    SpringApplication.run(KnpApplication.class, args);
}
```

`SpringApplication.run` — точка входа Spring Boot. Внутри: создаёт объект `SpringApplication`, конфигурирует его (тип приложения, listeners, environment), запускает `run` метод.

`run` делает многое. Печатает banner (легендарный Spring логотип). Создаёт и настраивает `Environment` — читает `application.yml`, применяет active profiles, разрешает placeholder-ы (`${...}` в конфиге), включает переменные окружения. Создаёт `ApplicationContext` подходящего типа. Для web-приложения это `ServletWebServerApplicationContext` (Tomcat/Jetty/Undertow) или `ReactiveWebServerApplicationContext` (Netty для WebFlux).

Затем **refresh контекста**. Component scan — рекурсивный обход пакета `KnpApplication` и вложенных, поиск классов со стереотипами. Разбор `@Configuration` классов, регистрация `@Bean` методов. Применение auto-configuration классов, каждый со своими conditional проверками. Строится граф зависимостей всех beans. Топологическая сортировка.

Создание beans по порядку. Каждый: инстанцирование, инъекция зависимостей, PostProcessor'ы (включая CGLIB/JDK proxy для AOP), `@PostConstruct`. Здесь могут быть ошибки: не может резолвить зависимость, конфликт версий классов (NoSuchMethodError), исключение в PostConstruct, циклическая зависимость.

Когда все beans созданы, Spring Boot стартует embedded server. Tomcat инициализируется, привязывается к настроенному порту (обычно 8080). Регистрирует все `@RestController` в MVC infrastructure. Проверяет доступность порта — если занят, приложение падает.

Опционально — регистрация в service registry. Если в classpath есть `spring-cloud-starter-consul-discovery`, при инициализации создаётся ConsulDiscoveryClient, при старте регистрирует приложение в Consul: имя сервиса, IP пода, порт, health check URL.

Публикуется `ApplicationReadyEvent`. Все `@EventListener(ApplicationReadyEvent.class)` срабатывают — часто здесь warmup, регистрация периодических задач, запуск фоновых worker'ов.

С этого момента приложение работает: слушает порт, обрабатывает HTTP запросы, отвечает клиентам. Kubernetes периодически дёргает `/actuator/health` — узнать что живо. Consul делает то же самое для service discovery. Prometheus скрейпит `/actuator/prometheus` для метрик.

Весь этот процесс занимает 20-60 секунд для типичного Spring Boot приложения — startup — не мгновенный. Это одна из причин почему в K8s настраивается `startupProbe` с большим `failureThreshold`. При падении на любом шаге (не резолвится зависимость, порт занят, БД не отвечает при init) — приложение падает, K8s пересоздаёт под, цикл повторяется. Если проблема системная — CrashLoopBackOff.

## Docker: контейнеризация приложения

Fat JAR запускается на любом хосте с JDK. Но "любой хост" — теоретическое понятие. На практике на разных серверах — разные версии Java, разные системные библиотеки, разные конфиги, разные timezones. Классическая проблема "у меня работает, на проде — нет". Решение — **контейнеризация** через Docker.

Docker container — это process в OS, запущенный в изолированной среде с собственной файловой системой, сетью, лимитами ресурсов. Изоляция реализуется через ядерные механизмы Linux: namespaces (изолируют PID, network, mount и другие пространства) и cgroups (лимитируют CPU, память, I/O). Из-за использования ядерных механизмов контейнер работает почти без overhead — не как виртуалка, где эмулируется весь compute.

**Image** — immutable шаблон, из которого запускаются контейнеры. Слоеный формат: base image (например, `amazoncorretto:21-alpine` — Alpine Linux с JDK 21), поверх — дополнительные слои от каждой инструкции Dockerfile. Каждый слой identifiable по content-hash, что даёт кэширование: если base image не изменилось, не нужно скачивать заново.

**Dockerfile** — инструкции для сборки image. Типичный для Spring Boot микросервиса:

```dockerfile
FROM amazoncorretto:21-alpine
WORKDIR /app
COPY build/libs/isna-knp-integration.jar app.jar
ENV JAVA_OPTS="-Xms512m -Xmx1024m -XX:MaxRAMPercentage=75"
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

FROM указывает base image. WORKDIR устанавливает рабочую директорию внутри контейнера. COPY копирует файлы с хоста в контейнер. ENV устанавливает переменные окружения. EXPOSE — документация о том, какой порт слушает (не reallocation, чисто аннотация). ENTRYPOINT — команда, выполняемая при запуске контейнера.

Каждая инструкция создаёт слой. Docker кэширует их: если инструкция и её inputs не изменились, слой берётся из кэша. Это делает пересборку быстрой. Layered JAR Spring Boot оптимизирует это ещё сильнее — dependencies (редко меняются) и application (часто меняется) идут в разные слои, при пересборке качается только application layer.

Image собирается через `docker build`, публикуется в **registry** (Docker Hub, внутренний Nexus, GitLab Container Registry). Kubernetes достаёт из registry по тегу, запускает контейнер.

## Kubernetes: оркестрация

Один Docker container — хорошо. Но микросервисная архитектура — это десятки-сотни сервисов, каждый по несколько replicas. Запускать вручную, следить за здоровьем, перезапускать при падении, балансировать нагрузку, прокидывать конфиги — невозможно. Нужен orchestrator. Стандарт — **Kubernetes (K8s)**.

Kubernetes работает по декларативному принципу. Ты описываешь **желаемое состояние**: "хочу 3 реплики этого сервиса, доступного на порту 8080, с такими environment variables". Kubernetes держит это состояние: если под падает — запускает новый, если нода недоступна — переносит на другую, если ты меняешь количество replicas — плавно масштабирует.

Архитектурно K8s состоит из **control plane** и **worker nodes**. Control plane — "мозг": API server (единственная точка входа для kubectl и всего остального), etcd (распределённая база хранящая всё состояние кластера), scheduler (решает где запускать новые pods), controller manager (собирает нужное поведение через контроллеры).

Worker nodes — физические или виртуальные машины, где реально работают контейнеры. На каждой ноде: kubelet (агент K8s, слушает API server, запускает/останавливает контейнеры через container runtime), kube-proxy (реализует Service — сетевую абстракцию), container runtime (containerd или другое, реально запускает контейнеры).

Основные ресурсы K8s.

**Pod** — минимальная единица деплоя. Обычно один под содержит один контейнер (одно приложение), иногда несколько (sidecar pattern: приложение + прокси, приложение + логгер). Контейнеры в поде делят сеть (один IP, доступ друг к другу через localhost) и volumes. Под — эфемерный: умер — появился новый с другим IP. Никогда не обращаешься к поду напрямую по IP.

**Deployment** — декларация "хочу N replicas такого-то пода". Deployment создаёт ReplicaSet (описывает набор одинаковых подов), тот создаёт Pods. Если под умер, ReplicaSet его пересоздаёт. Rolling update — плавная замена подов новой версии на новую (несколько новых поднимаются, старые убиваются).

**Service** — стабильная точка входа к группе подов. Виртуальный IP + порт, за которым скрыты N реплик. kube-proxy настраивает iptables/IPVS правила: трафик на service ClusterIP распределяется по подам, соответствующим selector. Внутренние сервисы имеют ClusterIP (доступны только внутри кластера). Внешние — LoadBalancer или Ingress.

**Namespace** — логическое разделение кластера. В КНП: `knp`, `fno`, `fo`, `tax-report`, `arm`. Ресурсы разных namespace изолированы (одни имена возможны в разных namespaces).

**ConfigMap** и **Secret** — конфигурация. ConfigMap для обычной, Secret для чувствительной (пароли, токены). Пробрасываются в поды как environment variables или файлы volumes.

**Ingress** — внешняя точка входа (HTTP). Обычно nginx-ingress controller: слушает входящий трафик, роутит по host/path на нужный Service. В КНП: `knp.kgd.gov.kz` → nginx-ingress → сервисы.

Каждый ресурс описан YAML-манифестом. Deployment пример:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: isnaknpintegration
  namespace: knp
spec:
  replicas: 3
  selector:
    matchLabels: {app: isnaknpintegration}
  template:
    metadata:
      labels: {app: isnaknpintegration}
    spec:
      containers:
      - name: app
        image: nexus.isna/isna-knp-integration:1.0.42
        ports: [{containerPort: 8080}]
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: prod
        livenessProbe:
          httpGet: {path: /actuator/health/liveness, port: 8080}
        readinessProbe:
          httpGet: {path: /actuator/health/readiness, port: 8080}
```

**Probes** — механизм проверки здоровья пода. Liveness — жив ли контейнер, при отказе K8s его убивает и запускает новый. Readiness — готов ли к трафику, при отказе pod остаётся, но исключается из Service (перестаёт получать трафик). Startup — есть ли время на startup, пока probe не проходит, liveness не проверяется. Все они настраиваются на `/actuator/health/*` endpoints Spring Boot.

## Consul: service discovery

В K8s есть Service — по-хорошему это уже service discovery. Но в КНП исторически используется дополнительно **Consul** от HashiCorp. Причины комплексные. Spring Cloud микросервисы часто хотят client-side load balancing (клиент сам выбирает под, не через Service). Consul умеет тонкое: веса подов, тегирование, свои health checks с бизнес-логикой (не просто TCP порт открыт, а `/actuator/health` возвращает UP). Плюс Consul даёт KV store для распределённых конфигов и feature flags.

Consul — три роли в одном продукте: service registry (каталог сервисов и их инстансов), health checking (активно опрашивает сервисы, помечает нездоровые), KV store (распределённое хранилище ключ-значение).

Как Spring Boot приложение регистрируется в Consul. В build.gradle: `implementation 'org.springframework.cloud:spring-cloud-starter-consul-discovery'`. В bootstrap.yml настройки: адрес Consul, имя сервиса, health check path, интервал.

При старте Spring Boot вызывает Consul API и регистрирует: имя сервиса (`isnaKnpUser`), IP пода, порт, health check URL. Consul сохраняет это в свою базу. Раз в 15 секунд Consul сам делает HTTP запрос на health check URL — если 200 OK, отмечает как passing, иначе failing. При shutdown приложения — deregister.

Другие сервисы используют Consul для discovery. Когда `isnaKnpIntegration` хочет позвать `isnaKnpUser`, он спрашивает у Consul: "дай мне список всех passing экземпляров isnaKnpUser". Consul возвращает список IP:port. Клиентская библиотека (Spring Cloud LoadBalancer, ранее Ribbon) выбирает один по алгоритму (round-robin по умолчанию), делает HTTP запрос напрямую туда.

Это называется **client-side load balancing**. В отличие от K8s Service, где балансировка на уровне iptables на сетевой стек, здесь клиент явно выбирает целевой pod. Плюс: тонкое управление, ретраи, circuit breaker, специфичная логика (по тегам, по регионам). Минус: клиент должен уметь.

**Feign** — декларативный HTTP клиент, использующий Consul discovery. Ты пишешь интерфейс:

```java
@FeignClient(name = "isnaKnpUser")
public interface UserClient {
    @GetMapping("/api/users/{id}")
    UserDto getById(@PathVariable Long id);
}
```

Spring Cloud генерирует proxy: при вызове `userClient.getById(5)` — proxy идёт в Consul за списком экземпляров isnaKnpUser, выбирает один, формирует HTTP GET, парсит JSON ответ в UserDto. Никакого явного HTTP кода — только интерфейс.

В КНП все internal коммуникации между сервисами — через Feign + Consul.

## Полный путь запроса

Соберём все компоненты в цельную картину на примере реального сценария. Пользователь в кабинете налогоплательщика подписывает ФНО (форма налоговой отчётности), нажимает "Отправить".

Браузер отправляет HTTPS POST на `knp.kgd.gov.kz/api/fno/submit`. Запрос идёт через интернет, DNS резолвит на IP nginx-ingress, TLS handshake.

nginx-ingress принимает HTTPS запрос, терминирует TLS, смотрит host и path. Правила Ingress говорят: `/api/*` → сервис `isna-knp-gateway`. Nginx forwards запрос туда.

`isna-knp-gateway` — Spring Cloud Gateway (или в legacy — Zuul). Роутит по префиксам: `/api/fno/*` → сервис `isnaKnpIntegration`. Спрашивает у Consul список экземпляров, выбирает один, forwards.

Запрос попадает в pod `isnaknpintegration-abc-xyz`. Внутри — Spring Boot приложение. Tomcat принимает запрос, передаёт в Spring MVC. Тот находит `@RestController` метод `@PostMapping("/api/fno/submit")`, вызывает.

Метод контроллера получает `Fno` объект (десериализованный Jackson-ом из JSON тела запроса). Валидация. Вызов `fnoService.submit(fno)`.

FnoService помечен `@Transactional`. Spring proxy перехватывает вызов, открывает транзакцию через `PlatformTransactionManager`. Внутри метода:

Валидация бизнес-правил. Вызов `userClient.getUserInfo(fno.userId)` через Feign. Feign идёт в Consul за списком `isnaKnpUser` экземпляров, выбирает один, HTTP GET, получает UserDto.

Сохранение в PostgreSQL через `fnoRepository.save(fno)`. Repository использует Hibernate/JPA, тот генерирует SQL INSERT, отправляет в PostgreSQL через JDBC connection из HikariCP пула. Connection пул подключается не напрямую к PostgreSQL, а через PgBouncer — connection pooler, снижающий нагрузку на реальный PostgreSQL backend.

Публикация события в RabbitMQ через `rabbitTemplate.convertAndSend()`. `FnoSubmittedEvent` сериализуется в JSON, отправляется в exchange `fno.events`. RabbitMQ маршрутизирует в очередь `tax-rep.fno.submitted`, где ждёт consumer сервиса `tax-rep`.

Метод завершается, proxy делает COMMIT транзакции. Ответ формируется, возвращается через все слои обратно: контроллер → Tomcat → gateway → nginx → пользователь.

Асинхронно, отдельно от исходного запроса, сервис `tax-rep` подхватывает сообщение из очереди. Обрабатывает — сохраняет в свою БД, отправляет уведомление, инициирует workflow в АРМ налогового органа. Всё независимо от исходного HTTP-запроса, который уже давно вернул 200 OK.

Плюс инфраструктура вокруг. Actuator endpoints собирают метрики. Prometheus периодически скрейпит их. Grafana визуализирует. Logs пишутся в stdout, kubelet перекладывает в файлы, Fluentbit собирает и отправляет в Elasticsearch/Loki. Hazelcast используется как distributed cache между инстансами. ShedLock координирует scheduled jobs между репликами (только один выполняет). Kalkan ECP для проверки электронной подписи. MinIO для хранения файлов (сканы, PDF). Keycloak для OAuth2/OIDC аутентификации.

## Что бывает когда всё это ломается

Каждый слой этой цепочки может отказать. Знание типовых проблем — часть работы senior.

Сборка. Gradle не может резолвить зависимость — нет доступа к Nexus, версия не найдена, конфликт версий. Compile error — сigntax problem, missing symbol. Test failure — юнит-тесты не прошли, сборка falls.

Docker. Base image не скачивается (registry unreachable). Build падает на COPY (файл отсутствует). Image слишком большой (плохой Dockerfile, forgot multi-stage build).

Deployment. Image pull failed (registry auth, wrong tag). Pod stuck Pending (нет ресурсов на кластере, taints/tolerations misconfig). CrashLoopBackOff (приложение падает при старте).

Startup Spring Boot. Не резолвится DataSource (БД недоступна). Bean creation failed (missing dependency, циклическая зависимость). Порт занят (обычно другой процесс, или проблемы с K8s Service). NoSuchMethodError (конфликт версий JAR-ов в classpath — реальный кейс из КНП с Hazelcast).

Runtime. Slow queries (плохие планы БД). OOMKilled (heap слишком мал или utечка). Deadlock (concurrent модификации). Consul deregister (health check failed, но pod живой — нужен rollout restart). Feign timeout (upstream service медленный или недоступен).

Диагностика идёт по слоям сверху вниз. Начинаешь с высокоуровнего: пользователь жалуется, приложение медленное. Смотришь метрики: latency растёт, throughput стабильный — значит проблема не в нагрузке. Логи: есть ли ошибки? Активные запросы: что делают? Trace конкретного запроса через Sleuth/Zipkin: где время теряется? Постепенно спускаешься до конкретного места: медленный SQL в PostgreSQL, timeout на внешний сервис, лок в Hibernate, что угодно.

Понимание всей цепочки от `.java` до `HTTP response` даёт возможность строить эту диагностику осмысленно, не гадая на кофейной гуще.

## Что дальше

Этот файл — обзорный. Каждая упомянутая тема имеет свой подробный файл в этой серии. `02-jvm-jdk-jre-bytecode.md` углубляется в JVM: как работает JIT, GC, память, classloading. `03-gradle-detailed.md` — детально о Gradle: конфигурации, задачи, incremental build, публикация. `04-jar-fatjar-detailed.md` — устройство JAR и fat JAR, layered jar, custom classloader. `05-spring-framework-ioc-di.md` — глубокое погружение в Spring: bean lifecycle, scopes, AOP, transaction management. Файлы 06 и дальше — про Spring Boot, HTTP серверы, JPA, микросервисы, K8s внутренности.

Полное понимание стека приходит не после прочтения, а после практики. Возьми свой сервис в КНП, разберись как он собирается, что в его fat JAR, как выглядят его манифесты K8s, как он регистрируется в Consul. Проведи запрос от браузера до ответа, посмотри trace, замерь время на каждом слое. Каждый такой пасс через реальную систему даёт больше понимания, чем часы чтения статей.
