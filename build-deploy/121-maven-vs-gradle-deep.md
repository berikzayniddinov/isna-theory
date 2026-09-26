# 121. Maven vs Gradle: полное сравнение глубоко

## Зачем это знать

Ты приходишь на новый Java-проект. Первое что видишь — либо `pom.xml`, либо `build.gradle`. Каждый разработчик имеет мнение — «Maven стабильный», «Gradle быстрее», «XML устарел», «Kotlin DSL лучше Groovy». Мнения полярные. На вопрос «какой лучше» — 50 разных ответов.

Реальность сложнее. Оба инструмента mature, оба используются в production крупными компаниями. Каждый решает те же задачи (build, test, dependencies, package) но с фундаментально разными философиями. Выбор влияет на: скорость сборки CI (реальные проекты — разница 2-10x), время onboarding новых разработчиков, сложность настройки custom logic, производительность IDE, размер команды которая может поддерживать build scripts.

Разница между «работаю с Maven/Gradle» и «понимаю оба» — способность за минуту ответить: почему Gradle incremental build работает а Maven каждый раз всё пересобирает. Что делает `gradle daemon` и почему первый запуск медленный а следующие быстрые. Как разрешаются конфликты версий транзитивных зависимостей в Maven (nearest wins) vs Gradle (highest wins) — и почему это может ломать сборку при миграции. Что такое multi-module проекты и в каком инструменте они удобнее. Почему Maven — declarative а Gradle — imperative и что это реально значит. Когда стоит мигрировать с Maven на Gradle и когда — не стоит.

Разберём: что такое build tool фундаментально — какую проблему решает. История и философия Maven и Gradle. Ключевая разница: конфигурация (XML vs Groovy/Kotlin DSL). Структура проекта. Lifecycle Maven vs Task Graph Gradle — фундаментальная архитектурная разница. Зависимости и conflict resolution (nearest vs highest). Repositories и Nexus/Artifactory. Плагины — как работают в каждом. Multi-module проекты. Performance: daemon, incremental build, cache (build cache vs Maven approach). Реальные времена сборки на примерах. Extensibility — как добавить custom logic. Reproducibility (одинаковые сборки везде). Publishing artifacts. IDE integration (IntelliJ, Eclipse). Migration Maven → Gradle пошагово. Когда что выбирать — реальные критерии. Мифы про Maven и Gradle. Real prod pitfalls в обеих системах. Практические рецепты для типовых задач.

Файл 03 (build-deploy/) — Gradle detailed. Здесь — сравнение и понимание trade-offs.

## Что такое build tool: одна и та же задача

Прежде чем сравнивать — понять что оба решают одно и то же.

**Build tool** — программа автоматизирующая процесс превращения исходного кода в исполняемый артефакт. Основные задачи:

**1. Compile** — превратить `.java` файлы в `.class`. Используется javac. Простой шаг для маленького проекта, сложный для большого (много модулей, зависимости, разные версии Java).

**2. Manage dependencies** — скачать нужные библиотеки. Твой проект использует Spring Boot, Hibernate, Jackson. Каждая библиотека — JAR-файл. Каждая тянет свои transitive dependencies (Spring Boot 3 тянет ~40 JARs). Build tool находит все нужные, скачивает, ставит в classpath.

**3. Run tests** — запустить unit tests, integration tests. Собрать отчёты.

**4. Package** — упаковать всё в артефакт. JAR (обычный или Spring Boot fat JAR), WAR, native image. С правильной структурой директорий, манифестом.

**5. Publish** — залить артефакт в repository (Maven Central, Nexus, Artifactory) чтобы другие проекты могли использовать.

Без build tool — всё это делать вручную:
- Скачать 100+ JARs с правильными версиями (учитывая совместимость).
- Правильно построить classpath (для compile, test, runtime — разные).
- Вызвать `javac` с правильными опциями.
- Запустить `junit` для каждого теста.
- Собрать `jar` с правильной структурой.
- Управлять зависимостями между модулями.
- Всё это воспроизводимо на любой машине.

Невозможно. Отсюда все build tools — от Ant в 2000-х до Maven и Gradle сейчас.

**Maven** и **Gradle** — самые популярные в Java-мире. Оба решают все эти задачи, но по-разному.

## История и философия

### Maven (2004+)

Появился как альтернатива Ant. Ant был imperative — ты вручную писал «скомпилируй файлы X → положи в директорию Y → упакуй в JAR». XML-based, но очень многословно, дублирование в каждом проекте.

Maven принёс **convention over configuration**. Идея: большинство Java-проектов делают то же самое (компилируют src/main/java в target/classes, тестируют src/test/java, упаковывают в JAR). Не надо описывать это каждый раз — есть стандартные соглашения. Пишешь только что **отличается** от defaults.

**Философия Maven**: 
- Declarative — говоришь **что** хочешь, не **как** это сделать.
- Standard project structure — `src/main/java`, `src/main/resources`, `src/test/java` фиксированы.
- Standard lifecycle — `validate → compile → test → package → install → deploy` в фиксированном порядке.
- Dependencies через центральный repository (Maven Central).
- Все ключевые операции — через plugins (compile plugin, surefire для tests, jar plugin, etc).

XML для конфигурации. `pom.xml` (Project Object Model).

### Gradle (2007+)

Появился как ответ на ограничения Maven. Основные проблемы Maven которые Gradle решает:

- **XML многословный** — простые вещи требуют десятков строк.
- **Extensibility сложная** — custom logic требует написания plugin'а на Java (месяц работы для простой задачи).
- **Performance не идеальная** — каждая сборка с нуля, никакого incremental.
- **Multi-module projects painful** — конфигурация повторяется.

Gradle взял идеи Maven (dependency management через central repo, convention over configuration) и добавил:

- **Groovy DSL** (позже Kotlin DSL) вместо XML. Настоящий язык — можно писать логику, циклы, условия.
- **Task graph** вместо жёсткого lifecycle. Задачи (tasks) и зависимости между ними. Гибче.
- **Incremental builds** — не пересобирать то, что не менялось.
- **Build cache** — переиспользование результатов между сборками (даже между разными машинами).
- **Daemon** — long-running процесс, не платим за старт JVM каждый раз.

**Философия Gradle**:
- Более imperative (можно писать логику), но с поддержкой declarative через DSL.
- Convention over configuration тоже есть, но более гибкая.
- Extensibility через плагины (пишутся легко) или прямо в build script.
- Performance — приоритет №1.

Groovy или Kotlin для конфигурации. `build.gradle` (Groovy) или `build.gradle.kts` (Kotlin).

### Кто использует что

**Maven** доминирует в enterprise Java. Причины:
- Стабильность (не меняется 20 лет).
- Простота onboarding — стандартная структура, легко разобраться.
- Хорошая интеграция со всеми инструментами.
- Большая экосистема plugins.

Использует: подавляющее большинство enterprise проектов, Apache Software Foundation, Red Hat, Oracle.

**Gradle** доминирует в Android и modern Java projects. Причины:
- Speed — критично для больших проектов.
- Flexibility — легко добавить custom build logic.
- Multi-module лучше.
- Kotlin support (Android активно использует).

Использует: Android (обязательно), Netflix, LinkedIn, Spring Framework (сам!), многие microservices стеки.

**У КНП**: Gradle (`build.gradle.kts` во многих модулях).

## Ключевая разница: конфигурация

Разберём на реальном примере — простое Spring Boot приложение с одной зависимостью.

### Maven: pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    
    <properties>
        <java.version>21</java.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

**~40 строк** для минимального Spring Boot проекта. XML синтаксис — много boilerplate (`<dependency><groupId>...</groupId><artifactId>...</artifactId></dependency>`).

### Gradle: build.gradle (Groovy)

```groovy
plugins {
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'java'
}

group = 'com.example'
version = '1.0.0-SNAPSHOT'
sourceCompatibility = '21'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

test {
    useJUnitPlatform()
}
```

**~20 строк**. Groovy DSL — компактнее, читаемее.

### Gradle: build.gradle.kts (Kotlin DSL)

```kotlin
plugins {
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
    java
}

group = "com.example"
version = "1.0.0-SNAPSHOT"
java.sourceCompatibility = JavaVersion.VERSION_21

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

Kotlin DSL — тот же размер, но со статической типизацией. IDE автокомплит работает лучше.

### Реальная разница в размере

Простой проект: Maven ~40 lines, Gradle ~20 lines.
Средний проект: Maven ~200 lines, Gradle ~80-120 lines.
Большой multi-module проект: Maven ~2000+ lines в pom.xml файлах, Gradle 500-800 lines.

Gradle обычно 40-70% размера Maven конфига для того же функционала.

## Структура проекта

Оба используют **одинаковую стандартную структуру** — Maven convention которую Gradle принял.

```
my-app/
├── src/
│   ├── main/
│   │   ├── java/              ← Java исходники
│   │   ├── resources/         ← config, static resources
│   │   └── webapp/            ← только для WAR
│   └── test/
│       ├── java/              ← unit tests
│       └── resources/         ← test resources
├── target/ (Maven) или build/ (Gradle)   ← output директория
│   ├── classes/               ← compiled .class файлы
│   ├── test-classes/
│   └── *.jar                  ← final artifact
├── pom.xml (Maven) или build.gradle(.kts) (Gradle)
└── .gitignore (обычно исключает target/ или build/)
```

Единственная реальная разница — имя output директории (`target` vs `build`) и имя build file.

## Lifecycle vs Task Graph: фундаментальная разница

Это ключевое архитектурное различие. Понимать надо для принятия решений.

### Maven Lifecycle: фиксированные фазы

Maven имеет **фиксированный жизненный цикл** — 3 основных lifecycles (default, clean, site), каждый состоит из phases. Default lifecycle:

```
validate → compile → test → package → verify → install → deploy
```

Phases выполняются **строго в этом порядке**. `mvn package` автоматически выполнит validate → compile → test → package. Ты не можешь пропустить phase (только через флаги типа `-DskipTests`).

Каждая phase может иметь **привязанные plugins**:
- `compile` phase → `maven-compiler-plugin:compile` goal.
- `test` phase → `maven-surefire-plugin:test` goal.
- `package` phase → `maven-jar-plugin:jar` goal.

Кастомные goals prilagается к нужной phase.

**Плюсы**:
- Просто понять. `mvn install` — всегда одно и то же.
- Стандартизация — любой Maven проект работает одинаково.
- Onboarding нового разработчика — знаешь Maven, знаешь любой Maven проект.

**Минусы**:
- Жёсткость. Хочешь пропустить compile? Нельзя без workaround'ов.
- Custom flows сложно. Хочешь запустить только integration tests без unit tests? Требует конфигурации.
- Всё привязано к phases — нельзя описать «эту task после этой, независимо от phase».

### Gradle Task Graph

Gradle работает с **task graph** — направленный граф задач с зависимостями между ними.

Каждая **task** — единица работы:
- `compileJava` — компилирует main Java.
- `compileTestJava` — компилирует test Java.
- `test` — запускает тесты.
- `jar` — упаковывает JAR.
- `bootJar` — упаковывает Spring Boot fat JAR.

Между tasks — **зависимости**:
- `test` зависит от `compileTestJava` (тесты нужно скомпилировать).
- `compileTestJava` зависит от `compileJava` (test код зависит от main).
- `jar` зависит от `compileJava`.

Когда ты запускаешь `gradle build`, Gradle:
1. Определяет какие task'и нужны (transitively через dependencies).
2. Строит topological order (граф).
3. Выполняет в нужном порядке (может параллельно если task'и независимы).

Можно вызвать любую task напрямую: `gradle test` — только компилирует и тестирует, JAR не упаковывает. `gradle jar` — компилирует и упаковывает, тесты не запускает.

**Custom task**:

```groovy
task hello {
    doLast {
        println 'Hello from Gradle!'
    }
}

task greetAndTest {
    dependsOn 'hello', 'test'
}
```

`gradle greetAndTest` — выполнит `hello` и `test`.

**Плюсы**:
- Гибкость. Custom task в 3 строки. Своя логика легко.
- Только нужные task'и — не всё что «должно быть до».
- Параллельное выполнение независимых task'ов.

**Минусы**:
- Сложнее понять для новичка. `gradle` — что реально произойдёт?
- Больше свободы = больше возможностей ошибок. Плохо написанный build script — сложнее отлаживать.

### Practical difference

Maven — «оnormal Java project». Gradle — «дай мне гибкость собирать что угодно как угодно».

Для типового микросервиса — оба работают одинаково хорошо. Для сложного multi-module проекта с custom logic — Gradle обычно легче.

## Dependency Management

Ключевая функция обоих инструментов.

### Как объявляются зависимости

**Maven**:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.2.0</version>
    <scope>compile</scope>  <!-- default -->
</dependency>
```

**Gradle**:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web:3.2.0'
}
```

### Scopes / Configurations

Обе системы имеют **scope** зависимости — где она нужна.

**Maven scopes**:
- **`compile`** (default) — доступна везде: compile, test, runtime.
- **`provided`** — нужна для compile, но не включена в JAR (например Servlet API — предоставляется контейнером).
- **`runtime`** — не нужна для compile, но нужна для runtime (например JDBC driver).
- **`test`** — только для тестов (JUnit, Mockito).
- **`system`** — как provided, но из локального jar (deprecated, не использовать).
- **`import`** — специальный для BOMs.

**Gradle configurations** (более гибкие):
- **`implementation`** — использую внутри модуля, зависимость не exposed для потребителей.
- **`api`** — экспортирую в API моего модуля (потребители тоже её видят транзитивно).
- **`compileOnly`** — только для compile (аналог provided).
- **`runtimeOnly`** — только для runtime.
- **`testImplementation`** — для тестов.
- **`testRuntimeOnly`** — для test runtime.
- **`annotationProcessor`** — для annotation processing (Lombok, MapStruct).

**Ключевая разница — `implementation` vs `api`**.

В Maven все compile-scoped зависимости — **транзитивно видимы**. Если модуль A зависит от библиотеки B, а модуль C зависит от A → C автоматически видит B в своём classpath.

Это плохо для encapsulation:
- Case: A использует Guava как деталь реализации. C не должен использовать Guava (это внутренняя деталь A). Но может — она транзитивно доступна.
- Случайные зависимости. Хочется поменять Guava на что-то другое → потенциально ломает C.

Gradle разделяет:
- **`implementation`** — B доступна для A, но НЕ транзитивно. C её не видит.
- **`api`** — B доступна и для A, и для транзитивно потребителей (C).

Пример:

```groovy
// Модуль my-lib
dependencies {
    implementation 'com.google.guava:guava:32.1.0'  // внутренняя
    api 'com.fasterxml.jackson.core:jackson-core:2.15.0'  // часть API
}
```

Модули использующие `my-lib` увидят Jackson (могут работать с ObjectMapper), но НЕ увидят Guava (это внутренняя деталь my-lib).

Плюс — лучшая инкапсуляция, изменение implementation не ломает потребителей. Плюс — быстрее компиляция (меньше classpath).

### Conflict Resolution: критическая разница

Что делать когда две зависимости требуют разные версии третьей?

Пример:
- Мой проект зависит от Spring Boot 3.2 (тянет Jackson 2.15.3).
- Мой проект также зависит от какой-то другой библиотеки (тянет Jackson 2.14.0).
- Какая Jackson будет в classpath?

**Maven: nearest wins**. Побеждает версия объявленная **ближе к корню графа зависимостей**. Если я явно объявил Jackson 2.14.0 — эта победит (я ближе всех). Если обе — транзитивные, определяется путём в графе.

**Gradle: highest wins**. Побеждает **наибольшая** версия. Из 2.14.0 и 2.15.3 → 2.15.3.

**Практические последствия**:

Migration с Maven на Gradle может **сломать сборку** — версия зависимости изменилась (потому что теперь highest wins а не nearest). Могут быть API-несовместимые изменения.

Обход: явно задавать версии критичных зависимостей (`implementation 'com.fasterxml.jackson:jackson-core:2.14.0'`) — тогда конфликтов нет.

**Gradle подход обычно правильнее** — highest wins защищает от использования устаревших версий с багами/уязвимостями. Но требует внимания при миграции.

**Просмотр дерева зависимостей**:

Maven:
```bash
mvn dependency:tree
```

Gradle:
```bash
gradle dependencies
```

Показывает всё дерево + помечает конфликты и какая версия победила.

## Performance: реальные числа

Самая заметная разница между инструментами.

### Cold build (первый запуск, кэши пустые)

Одинаковый Spring Boot проект (~50 файлов, 20 зависимостей):

- **Maven**: 30-45 секунд.
- **Gradle**: 25-40 секунд.

Практически одинаково для маленьких проектов. Разница появляется на больших.

### Incremental build (после изменения одного файла)

Изменил один Java файл, снова build:

- **Maven**: 15-25 секунд. Каждый раз почти всё пересобирает. Некоторые plugins поддерживают incremental, но по умолчанию — полная пересборка.
- **Gradle**: 3-8 секунд. Пересобирает только затронутое (task inputs/outputs tracking).

**Разница 3-5x**. На большом проекте — ещё драматичнее.

### Build cache (второй запуск того же build)

**Maven**: нет built-in build cache. Каждый CI build с нуля. Есть `takari` plugin который добавляет — но не стандарт.

**Gradle**: **build cache** встроен. Task outputs cached по hash inputs. Второй запуск с теми же inputs → outputs достаются из cache мгновенно.

Более того — **remote build cache**. Cache доступен всей команде через HTTP. Коллега собрал task на своей машине → cache в remote → ты делаешь тот же commit → cache hit локально.

Ускорение колоссальное. У Netflix reported 50-80% cache hit rate на CI, время сборки уменьшилось в разы.

### Gradle Daemon

Gradle запускает **long-running daemon process** — Groovy compiler, JVM initialized, plugins loaded один раз. Следующие сборки используют existing daemon.

Первый `gradle build` — 15 секунд включая cold JVM start.
Второй `gradle build` — 2 секунды на маленьком проекте.

Maven **не имеет daemon по умолчанию** (есть `mvnd` — Maven Daemon как community proj, но не mainstream). Каждый `mvn` — fresh JVM start (~2-3 секунды overhead).

### Real-world numbers

Реальный проект КНП (~30 модулей, ~200 зависимостей суммарно):

- **Maven full build** (`mvn clean install`): 8-12 минут.
- **Gradle full build** (`gradle build`): 5-8 минут (без cache).
- **Gradle incremental** (изменил один файл): 30 секунд.
- **Gradle с cache hit**: 15 секунд.

Разница на CI (где обычно clean build) — не такая большая (2x-3x). Но local development — Gradle значительно приятнее.

## Multi-module projects

Реальные проекты редко имеют один pom.xml/build.gradle. Обычно — parent + N child modules.

### Maven multi-module

**Parent pom.xml**:
```xml
<groupId>com.example</groupId>
<artifactId>my-app-parent</artifactId>
<packaging>pom</packaging>  <!-- pom packaging для parent -->

<modules>
    <module>my-app-common</module>
    <module>my-app-web</module>
    <module>my-app-service</module>
</modules>

<dependencyManagement>
    <!-- Централизованные версии, не сами зависимости -->
    <dependencies>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>2.0.9</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**Child pom.xml**:
```xml
<parent>
    <groupId>com.example</groupId>
    <artifactId>my-app-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</parent>

<artifactId>my-app-web</artifactId>

<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>my-app-common</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <!-- version из parent dependencyManagement -->
    </dependency>
</dependencies>
```

**Плюсы Maven multi-module**:
- Стандартная структура.
- `dependencyManagement` централизует версии.
- Родительский pom координирует.

**Минусы**:
- XML многословный, дублируется в каждом child.
- Каждый child — свой pom.xml (даже если почти пустой).

### Gradle multi-module

**settings.gradle**:
```groovy
rootProject.name = 'my-app'

include 'my-app-common'
include 'my-app-web'
include 'my-app-service'
```

**Root build.gradle**:
```groovy
plugins {
    id 'org.springframework.boot' version '3.2.0' apply false
    id 'io.spring.dependency-management' version '1.1.4' apply false
    id 'java'
}

// Общие настройки для всех subprojects
subprojects {
    apply plugin: 'java'
    apply plugin: 'io.spring.dependency-management'
    
    group = 'com.example'
    version = '1.0.0-SNAPSHOT'
    sourceCompatibility = '21'
    
    repositories {
        mavenCentral()
    }
    
    dependencyManagement {
        imports {
            mavenBom 'org.springframework.boot:spring-boot-dependencies:3.2.0'
        }
    }
    
    test {
        useJUnitPlatform()
    }
}
```

**Child build.gradle** (`my-app-web/build.gradle`):
```groovy
apply plugin: 'org.springframework.boot'

dependencies {
    implementation project(':my-app-common')
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // версии из dependencyManagement в root
}
```

**Плюсы Gradle multi-module**:
- Один root build.gradle с общими настройками, child'ы наследуют.
- Меньше boilerplate.
- Легко использовать программную логику для многих модулей.
- Ссылка на другой модуль — `project(':name')`, чистый синтаксис.

**Минусы**:
- Немного сложнее для новичка (Groovy DSL нужно понимать).

Для больших multi-module проектов Gradle обычно значительно удобнее.

## Plugins: как работают

Оба инструмента расширяются через plugins.

### Maven plugins

Plugin = jar file + `plugin.xml` описывающий goals. Goals — как tasks в Gradle.

**Стандартные plugins**:
- `maven-compiler-plugin` — compile Java.
- `maven-surefire-plugin` — unit tests.
- `maven-failsafe-plugin` — integration tests.
- `maven-jar-plugin` — упаковка в JAR.
- `maven-install-plugin` — install в local repo.

**Использование**:
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <version>3.2.0</version>
            <executions>
                <execution>
                    <goals>
                        <goal>repackage</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

Многословно, но декларативно.

**Написать свой Maven plugin** — отдельный Java проект с mojo classes, packaging = `maven-plugin`. Сложно (~день работы даже для простого).

### Gradle plugins

**Три вида plugins**:

**1. Core plugins** (встроены):
```groovy
plugins {
    id 'java'
    id 'application'
    id 'maven-publish'
}
```

**2. Community plugins** (из Plugin Portal):
```groovy
plugins {
    id 'org.springframework.boot' version '3.2.0'
    id 'com.diffplug.spotless' version '6.22.0'
}
```

**3. Inline plugin в build.gradle**:
```groovy
task deployToStaging {
    doLast {
        exec {
            commandLine 'kubectl', 'apply', '-f', 'k8s/staging.yaml'
        }
    }
}
```

Простую логику можно писать **прямо в build script**. Никакого отдельного проекта.

**Написать свой Gradle plugin** — тоже отдельный проект, но проще. Или **precompiled script plugin** в директории `buildSrc/` — Groovy/Kotlin файл, автоматически доступен из build.gradle.

## Reproducibility

**Reproducible build** — та же самая исходная кодовая база собирается идентично на любой машине, в любое время.

### Проблема

Разработчик собирает проект локально — работает. CI собирает — падает. Reasons:
- Разные версии Java (11 у dev, 17 у CI).
- Разные версии Maven/Gradle.
- Транзитивные зависимости обновились (SNAPSHOT версия, LATEST версия — что даст сегодня и завтра разные).
- Разные plugins.

Классический «works on my machine».

### Maven Wrapper

Скрипт `mvnw` в проекте. Автоматически скачивает нужную версию Maven если не установлена:

```bash
./mvnw clean install
```

`mvn-wrapper.properties`:
```
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.5/apache-maven-3.9.5-bin.zip
```

Все члены команды и CI используют одну и ту же версию Maven.

### Gradle Wrapper

Скрипт `gradlew` — то же самое.

```bash
./gradlew build
```

`gradle/wrapper/gradle-wrapper.properties`:
```
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
```

**Wrapper — обязателен в prod projects**. Никогда не использовать глобальную установку.

### Lock files (dependency version locking)

Транзитивные зависимости могут обновиться (новый minor release библиотеки) → билд теперь получает другую версию. Опасно для reproducibility.

**Gradle**: `gradle dependencies --write-locks` создаёт `gradle.lockfile` с точными версиями. Следующие сборки — только эти версии.

**Maven**: нет встроенного механизма. Есть `maven-enforcer-plugin` для policies и unofficial lock решения.

Для критичных проектов — использовать locks.

## Publishing artifacts

### Maven publishing

```xml
<distributionManagement>
    <repository>
        <id>nexus-releases</id>
        <url>https://nexus.example.com/repository/releases</url>
    </repository>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>https://nexus.example.com/repository/snapshots</url>
    </snapshotRepository>
</distributionManagement>
```

Deploy:
```bash
mvn deploy
```

Credentials в `~/.m2/settings.xml` (не в pom.xml — секреты!).

### Gradle publishing

```groovy
plugins {
    id 'maven-publish'
}

publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java
        }
    }
    repositories {
        maven {
            name = 'nexus'
            url = version.endsWith('SNAPSHOT') 
                ? 'https://nexus.example.com/repository/snapshots'
                : 'https://nexus.example.com/repository/releases'
            credentials {
                username = project.findProperty('nexusUsername') ?: System.getenv('NEXUS_USERNAME')
                password = project.findProperty('nexusPassword') ?: System.getenv('NEXUS_PASSWORD')
            }
        }
    }
}
```

Deploy:
```bash
gradle publish
```

Credentials через environment variables или `~/.gradle/gradle.properties`.

## IDE Integration

Обе системы отлично поддерживаются в **IntelliJ IDEA** — best-in-class инструмент для Java.

**Maven**:
- IntelliJ реально понимает pom.xml. Импорт быстрый.
- Автоматически detected при open проекта.
- Reload dependencies через Maven панель.

**Gradle**:
- Импорт медленнее (запуск Gradle daemon).
- IntelliJ понимает build.gradle(.kts).
- **Kotlin DSL значительно лучше в IDE** — статическая типизация, автокомплит, quick fixes. Groovy DSL — dynamic, автокомплит ограниченный.
- Reload через Gradle панель.

**Eclipse**:
- Maven support отличный (m2e plugin).
- Gradle support через Buildship — работает, но не так гладко как в IntelliJ.

**VS Code**:
- Maven и Gradle оба работают через Java Extension Pack.
- Не для heavy Java development, но приемлемо.

Для Gradle проектов — **IntelliJ + Kotlin DSL** — золотой стандарт.

## Migration Maven → Gradle

Стандартный сценарий — команда решила перейти на Gradle для performance / flexibility.

### Автоматическая конвертация

```bash
gradle init --type pom
```

Читает pom.xml, генерирует базовый build.gradle. Работает для 80% случаев — простые проекты конвертируются автоматически. Сложные (много custom plugins, non-standard structures) — требуют ручной работы.

### Ручные шаги

1. **Установить Gradle Wrapper**:
   ```bash
   gradle wrapper --gradle-version=8.5
   ```

2. **Convertировать pom.xml в build.gradle**. Основное:
   - `<dependencies>` → `dependencies { implementation '...' }`
   - `<properties>` → `ext.propertyName = 'value'` (или напрямую)
   - `<build><plugins>` → `plugins { id '...' }`

3. **Настроить multi-module**:
   - `settings.gradle` с `include 'module-name'`
   - Root `build.gradle` с общими настройками
   - Child `build.gradle` каждого модуля

4. **Пересобрать, проверить classpath**:
   - Разные conflict resolution может привести к разным версиям.
   - `gradle dependencies` — сравнить с `mvn dependency:tree`.
   - Фиксировать критичные версии явно если нужно.

5. **Convert CI scripts**:
   - `mvn clean install` → `gradle build`
   - Настроить build cache для CI (если Gradle Enterprise / self-hosted).

6. **Update team**:
   - Тренинг Groovy/Kotlin DSL.
   - Documentation.

### Ловушки миграции

**Разные version resolution** — уже упоминали. Может сломать сборку.

**Custom Maven plugins** — нет прямого аналога в Gradle. Требуют переписывания. Оценить количество custom plugins перед миграцией.

**Инфраструктура CI/CD** — если много Maven-specific в pipelines. Требует обновления.

**Учебная кривая** — команда должна learn Gradle. Timeline: неделя-две для базового уровня, месяц для комфорта, 3-6 месяцев для expert-level.

## Когда что выбирать

### Выбирай Maven если

- **Небольшая команда** без dedicated build engineer'а. Maven проще поддерживать.
- **Проект простой** — обычный Spring Boot микросервис без сложной build logic.
- **Standard Java project** без специфичных нужд. Maven "just works".
- **Legacy проекты** — уже на Maven, миграция не стоит выгоды.
- **Команда более комфортна с XML** и declarative подходом.
- **Максимальная стабильность** важнее performance.

### Выбирай Gradle если

- **Большой проект** — 20+ модулей, много custom build logic.
- **Multi-language** — Java + Kotlin + Groovy + что-то ещё. Gradle гибче.
- **Performance критична** — большие сборки где важна каждая минута.
- **Много custom logic** в build (генерация кода, custom packaging, integration с внешними инструментами).
- **Android** — обязательно Gradle.
- **Спускать больше свободы** команде.

**У КНП**: Gradle (много модулей, custom Java 11→21 миграция была бы намного сложнее на Maven).

### Не выбирай сам build tool

Если приходишь на существующий проект — используй то что уже есть. Миграция очень редко окупается для приличных Maven проектов.

## Мифы про производительность

**Миф 1: "Gradle всегда быстрее"**.

Правда: **incremental и cached** builds быстрее в Gradle. Cold clean build — сопоставим. Для simple projects — разница минимальная.

**Миф 2: "Maven надёжнее"**.

Правда: оба mature. Maven не меняется 20 лет — да, стабильный. Но Gradle тоже production-ready 10+ лет, используется FAANG.

**Миф 3: "XML легче Groovy"**.

Правда: XML легче **читать без знаний**. Groovy легче **писать и модифицировать**. Kotlin DSL — легче обеих для IDE integration.

**Миф 4: "Maven плагинов больше"**.

Правда: оба имеют огромные экосистемы. Многое Maven plugin работает и в Gradle через adapter. Gradle Plugin Portal имеет 10K+ plugins.

## Real prod pitfalls

### Maven pitfalls

**Слишком много phases** — `mvn install` пересобирает всё, даже когда нужен только один модуль. Скорость деградирует на больших проектах.

**Snapshot dependencies** — `1.0.0-SNAPSHOT` версии могут обновиться незаметно. Non-reproducible builds.

**Настройки в `~/.m2/settings.xml`** — не в проекте, зависят от машины. Работает у dev, не работает у CI.

**Профили (profiles)** — легко забыть какой активен. Разные результаты сборки в разных environments.

**Multi-module `mvn install`** — часто нужно установить все модули в local repo прежде чем работать с одним. Медленно на больших проектах.

### Gradle pitfalls

**Groovy dynamic typing** — опечатки в build script не видны до runtime. `implementaion '...'` вместо `implementation '...'` — молчит.

**Fix**: Kotlin DSL — статическая типизация, opечатки видны в IDE.

**Плагины могут делать что угодно** — build script имеет full access к system. Плагин от stranger может делать вредоносное. Использовать только trusted plugins.

**Дифференцированный API между версиями** — plugin для Gradle 5 может не работать в Gradle 8. Требует обновления.

**Daemon может «застрять»** — memory leaks, OOM в daemon. `gradle --stop` перезапускает.

**Configuration cache** — новая фича, ещё не всегда работает с plugins. Может ломать builds когда включаешь.

**Groovy performance** — Groovy сам по себе не быстрый. Compilation build script'а занимает время. Kotlin DSL быстрее.

## Практические рецепты

### Убедиться в reproducibility

Обе системы:
1. Использовать Wrapper (`mvnw` / `gradlew`).
2. Фиксировать версии всех зависимостей (без LATEST, без range).
3. Избегать SNAPSHOT в production builds.
4. Fixed version Java (`sourceCompatibility = '21'`, `.mvn/jvm.config`).

Gradle: dependency locking (`gradle.lockfile`).

### Ускорить CI build

Maven:
- `-T 4C` — parallel build с 4 threads.
- `-o` (offline mode) если dependencies кэшированы.
- Кэшировать `~/.m2/repository` между CI runs.

Gradle:
- Build cache (local + remote).
- `--parallel` — параллельно между modules.
- Daemon (обычно уже включён).
- `--configure-on-demand` — configure только нужные modules.

### Найти конфликт версий

Maven:
```bash
mvn dependency:tree -Dverbose
mvn dependency:analyze
```

Gradle:
```bash
gradle dependencies
gradle dependencyInsight --dependency guava
```

### Обновить все зависимости

Maven:
```bash
mvn versions:display-dependency-updates
mvn versions:use-latest-releases
```

Gradle: **не встроено**. Использовать plugin `com.github.ben-manes.versions`:
```bash
gradle dependencyUpdates
```

## Заключение

**Maven** и **Gradle** — оба mature production-ready build tools для Java. Решают одни задачи: compile, dependencies, test, package, publish. Отличаются философией и подходом.

**Maven**: declarative, XML, convention over configuration, fixed lifecycle, стабильность, простота onboarding. Стандарт enterprise Java.

**Gradle**: imperative + declarative, Groovy/Kotlin DSL, task graph, гибкость, performance (incremental, cache, daemon). Стандарт Android, modern Java projects.

**Ключевые различия**:

- **Configuration**: XML vs Groovy/Kotlin DSL. Gradle на 40-70% компактнее.
- **Lifecycle vs Task Graph**: Maven — фиксированные phases; Gradle — гибкий граф task'ов.
- **Dependency scopes**: Maven — все transitive; Gradle различает `implementation` (не transitive) vs `api` (transitive).
- **Conflict resolution**: Maven — nearest wins; Gradle — highest wins.
- **Performance**: Gradle значительно быстрее для incremental и cached builds; сопоставимы для cold clean.
- **Multi-module**: Gradle обычно значительно удобнее (меньше boilerplate).
- **Extensibility**: Custom logic в Gradle намного проще (inline в build script vs отдельный plugin проект в Maven).

**Реальные числа performance** (medium проект):
- Cold build: Maven 30-45s, Gradle 25-40s.
- Incremental: Maven 15-25s (полная пересборка), Gradle 3-8s (только затронутое).
- Cache hit: Gradle секунды.

**Выбор**:
- **Maven** — простые проекты, небольшие команды, максимальная стабильность, legacy.
- **Gradle** — большие проекты, много модулей, custom build logic, performance критична, Android.

**Ловушки Maven** — многословный XML, snapshot dependencies, `~/.m2/settings.xml` machine-specific.

**Ловушки Gradle** — Groovy dynamic typing (использовать Kotlin DSL), plugin trust, daemon memory issues.

**Обязательное для обоих**:
- Wrapper (`mvnw` / `gradlew`) — не глобальная установка.
- Фиксированные версии — не LATEST, не version ranges.
- Не SNAPSHOTs в prod.

**Migration Maven → Gradle** — 80% автоматизируется через `gradle init --type pom`. Остальное — ручная работа. Timeline 1-3 месяца для приличного проекта. Стоит только если реально нужна производительность или гибкость.

Для новых проектов — обычно **Gradle с Kotlin DSL** — лучший choice для большинства случаев. Но Maven остаётся отличным выбором для типовых enterprise проектов где стабильность и простота важнее гибкости.

Gradle подробно — файл 03 (build-deploy/). JAR / Fat JAR — 04. Здесь было полное сравнение с trade-offs, реальными числами, миграцией и практическими рецептами.
