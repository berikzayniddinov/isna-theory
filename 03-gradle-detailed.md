# 03. Gradle: система сборки Java-проектов

## Что делает система сборки

Простой Java-проект — один файл `Main.java` с методом `main`. Собрать вручную легко: `javac Main.java`, `java Main`. Работает. Но реальный enterprise проект — это сотни или тысячи `.java` файлов, разбитых на пакеты. Каждый зависит от других. Есть внешние библиотеки (Spring, Hibernate, Jackson), которые нужно скачивать. Есть unit-тесты, интеграционные тесты, ресурсы, конфигурация. Есть многомодульная структура — приложение из нескольких подпроектов. Есть разные окружения (dev, prod), разные версии Java (11, 21). Есть публикация в артефакт-репозиторий.

Собрать всё это руками невозможно. Нужен инструмент, автоматизирующий процесс. Такой инструмент называется **build system** или **build tool**. Он знает где искать зависимости, в каком порядке компилировать, как упаковать результат, как прогнать тесты, как опубликовать. Разработчик пишет декларативное описание проекта, инструмент выполняет.

В Java-мире исторически два основных build tool: **Maven** и **Gradle**. Maven появился в 2004, использует XML-конфигурацию (`pom.xml`), строгие конвенции, стабильный lifecycle. Gradle появился в 2007, использует Groovy или Kotlin DSL для скриптов сборки, гибче, быстрее (кэширование, инкрементальная сборка, демон). В КНП везде Gradle, поэтому в этом файле мы разберём его детально.

Знание build tool — не первоочередной навык для написания бизнес-кода, но обязательный для senior. Ты будешь настраивать сборку многомодульного проекта, разбираться с конфликтами версий зависимостей, оптимизировать время сборки на CI, публиковать библиотеки в внутренний Nexus. Все эти задачи требуют глубокого понимания того, что происходит когда ты запускаешь `./gradlew build`.

## Структура Gradle-проекта

Стандартный Gradle-проект имеет фиксированную структуру, основанную на **conventions** (соглашениях). Convention over configuration — если следовать соглашениям, конфигурация минимальна. Если хочешь по-своему, всё настраивается, но обычно нет нужды.

Корень проекта содержит несколько ключевых файлов. `settings.gradle` — определяет структуру проекта: имя корня, список подпроектов (модулей). Обычно небольшой. `build.gradle` — главный конфигурационный файл: плагины, зависимости, задачи. `gradle.properties` — property-переменные, обычно JVM args для Gradle-демона, версии зависимостей. `gradlew` и `gradlew.bat` — Gradle wrapper скрипты. `gradle/wrapper/gradle-wrapper.jar` и `gradle-wrapper.properties` — сам wrapper, фиксирующий версию Gradle.

Исходники Java лежат в `src/main/java`. Ресурсы — в `src/main/resources`. Тесты соответственно — `src/test/java` и `src/test/resources`. Эта структура — конвенция от Maven, унаследованная Gradle. Меняется через `sourceSets`, но обычно не имеет смысла.

Результат сборки идёт в `build/` (например, `build/libs/myapp-1.0.jar`). Эта директория в `.gitignore`.

Многомодульный проект (типичный в КНП) организован иерархически. Корневой `settings.gradle` перечисляет подмодули:

```gradle
rootProject.name = 'isna-knp'
include 'isna-knp-integration'
include 'isna-knp-fno'
include 'isna-knp-gateway'
```

Каждый подмодуль — это директория с своим `build.gradle` и своим `src/`. Корневой `build.gradle` часто содержит общие настройки для всех подмодулей через блоки `allprojects { ... }` (применяется к корню и всем подмодулям) и `subprojects { ... }` (только к подмодулям).

Такая структура позволяет естественно моделировать enterprise-приложение с несколькими логическими компонентами. `isna-knp-shared` — общие модели и утилиты. `isna-knp-integration` — сервис интеграции. `isna-knp-fno` — сервис форм налоговой отчётности. Каждый может зависеть от других (`implementation project(':isna-knp-shared')`).

## Gradle Wrapper: воспроизводимость сборки

Одна из типовых проблем разработки — "у меня работает, у тебя нет". Часто причина — разные версии инструментов. Пётр использует Gradle 7.4, Василий — 8.5, CI имеет 6.9. Сборка может вести себя по-разному, ломаться на одних, работать на других.

**Gradle Wrapper** решает эту проблему. Вместо системного `gradle` команды используется `./gradlew` — скрипт, лежащий в проекте. При запуске он:

Читает `gradle/wrapper/gradle-wrapper.properties`, где указана версия Gradle:
```
distributionUrl=https://services.gradle.org/distributions/gradle-8.5-bin.zip
```

Проверяет, скачана ли эта версия локально (в `~/.gradle/wrapper/dists/`). Если нет — скачивает.

Запускает эту конкретную версию, передавая ей все аргументы.

Результат — независимо от того, что установлено в системе, wrapper использует ту версию Gradle, которая указана в проекте. Разработчик клонирует репозиторий, запускает `./gradlew build`, все зависимости и правильная версия Gradle подтягиваются автоматически.

**Правило проекта: всегда использовать `./gradlew`, никогда системный `gradle`**. Это гарантирует воспроизводимость. Первый запуск на новой машине долгий (скачивание Gradle), последующие — из локального кэша.

Обновить версию Gradle: `./gradlew wrapper --gradle-version=8.5`. Это перезапишет `gradle-wrapper.properties`, следующий запуск использует новую версию.

## Плагины

Gradle сам по себе не знает как компилировать Java или собирать JAR. Всю функциональность добавляют **плагины**. Плагин регистрирует задачи, конфигурации, конвенции.

Ключевые плагины Gradle для Java.

**`java`** — базовый Java plugin. Добавляет задачи `compileJava`, `test`, `jar`. Определяет `sourceSets` (main и test). Регистрирует стандартные конфигурации зависимостей (`implementation`, `runtimeOnly`, `testImplementation` и другие).

**`java-library`** — расширение `java`. Добавляет конфигурацию `api` для транзитивно доступных зависимостей. Используется для проектов, публикующих библиотеки для использования другими.

**`org.springframework.boot`** — Spring Boot плагин. Добавляет `bootJar` (сборка fat JAR), `bootRun` (запуск приложения), `bootBuildImage` (сборка Docker image через Cloud Native Buildpacks). Переопределяет `MANIFEST.MF` на нужную для Spring Boot структуру (Main-Class = JarLauncher, Start-Class = твой класс).

**`io.spring.dependency-management`** — управление версиями через Maven BOM. Позволяет писать зависимости без явных версий: `implementation 'org.springframework.boot:spring-boot-starter-web'` — версия придёт из импортированного BOM.

**`maven-publish`** — публикация артефактов в Maven-совместимый репозиторий (Nexus, Artifactory).

**`jacoco`** — измерение покрытия тестами.

**`checkstyle`, `spotbugs`, `pmd`** — static analysis.

Подключение плагинов делается в блоке `plugins`:

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.2.4.RELEASE'
    id 'io.spring.dependency-management' version '1.0.9.RELEASE'
    id 'jacoco'
}
```

Порядок важен только для некоторых плагинов (Spring Boot нужно применять после Java).

Есть два способа подключения: `plugins { }` блок (современный, декларативный) и `apply plugin:` (старый, императивный). Первый предпочтителен для читаемости и производительности (Gradle оптимизирует resolution).

## Конфигурации зависимостей

Одна из главных функций Gradle — управление зависимостями. Проект зависит от библиотек, библиотеки — от других библиотек (транзитивные зависимости). Gradle резолвит весь этот граф, скачивает нужные JAR из репозиториев, кэширует локально в `~/.gradle/caches/`.

Зависимости объявляются в блоке `dependencies` с указанием **конфигурации**, определяющей где нужна эта зависимость и как обрабатывается.

**`implementation`** — самая частая. Зависимость доступна и при компиляции, и в runtime. НЕ пробрасывается транзитивно потребителям этого проекта. Если модуль A имеет `implementation 'lib'`, модули зависящие от A не видят lib. Смысл — encapsulation: пользователь не знает про internal implementation детали, изменение внутренней зависимости не заставляет пересобирать пользователей.

**`api`** — как `implementation`, но пробрасывается транзитивно. Используется только когда типы из зависимости появляются в публичном API этого модуля (в параметрах публичных методов, возвращаемых значениях). Если нет — используй `implementation`. Меньше `api` — быстрее пересборка, меньше coupling.

**`compileOnly`** — только при компиляции. Не идёт в runtime classpath, не упаковывается в JAR. Классические примеры: **Lombok** (обрабатывает аннотации при компиляции, в runtime сгенерированный код не требует Lombok), Servlet API для WAR-приложений (сервер сам предоставит в runtime).

**`runtimeOnly`** — только в runtime, не при компиляции. Классический пример: **JDBC-драйверы**. Код работает через интерфейсы `java.sql.DriverManager`, `java.sql.Connection`. Реальная реализация (PostgreSQL, MySQL) подключается через service loader при runtime. Кодовая база на них не ссылается напрямую.

**`testImplementation`, `testRuntimeOnly`** — аналогично, но только для тестов.

**`annotationProcessor`** — annotation processor подключается компилятору для обработки аннотаций во время компиляции. Lombok — генерирует boilerplate (getters, setters). MapStruct — генерирует мапперы объектов. Spring configuration processor — генерирует metadata для `@ConfigurationProperties`.

Пример типичного `dependencies` блока Spring Boot микросервиса:

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.cloud:spring-cloud-starter-consul-discovery'
    
    runtimeOnly 'org.postgresql:postgresql'
    
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

## Транзитивные зависимости и разрешение конфликтов

Одна прямая зависимость (`spring-boot-starter-web`) тянет десятки транзитивных: `spring-web`, `spring-webmvc`, `jackson`, `tomcat-embed-core`, `logback` и другие. Ты не пишешь их явно — Gradle подтягивает автоматически, разбираясь в `pom.xml` каждой библиотеки.

Проблема возникает когда разные зависимости требуют разные версии одной библиотеки. Библиотека X требует `guava:30`, библиотека Y требует `guava:20`. В runtime classpath может быть только одна версия одного класса.

Gradle по умолчанию использует стратегию **highest wins** — выбирает самую свежую из запрошенных версий. `guava:30` победит `guava:20`. Это работает большинство времени, потому что major версии обычно обратно совместимы. Но иногда ломает: если Y использовал метод, удалённый в 30, при runtime — `NoSuchMethodError`.

Отладка через `./gradlew :module:dependencies --configuration runtimeClasspath`. Выводит дерево зависимостей: что от чего пришло, какие версии были запрошены, какие выбраны. Строки с `(*)` — уже отображено выше. `-> 30.0-jre` — версия была изменена (например, с 20.0 на 30.0-jre из-за конфликта).

Управление версиями. **Force** — принудительная фиксация:

```gradle
configurations.all {
    resolutionStrategy {
        force 'com.google.guava:guava:29.0-jre'
    }
}
```

**Exclude** — исключение транзитивной зависимости:

```gradle
implementation('org.springframework.boot:spring-boot-starter-web') {
    exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
}
```

Часто используется для замены дефолтного Tomcat на Undertow или Jetty.

**BOM (Bill of Materials)** — импорт согласованного набора версий. BOM — специальный `pom.xml` без реальных артефактов, только версии зависимостей. Spring Boot публикует свой BOM `spring-boot-dependencies`, где перечислены совместимые версии всей экосистемы.

```gradle
implementation platform('org.springframework.boot:spring-boot-dependencies:2.2.4.RELEASE')
implementation 'org.springframework.boot:spring-boot-starter-web'  // версия из BOM
```

В КНП есть собственный BOM `isna-global-21` для Java 21 версии, где фиксированы версии внутренних библиотек. Модули указывают зависимости без версий — они приходят из BOM. Изменение версии в одном месте обновляет все модули.

**Реальный кавет из КНП**: `commons-21` пинит `isna-global-21` master `1.2.18`, а номерные тэги ветки могут указывать на `1.3.5`. Смешивание source разных tags может привести к collision классов и NoSuchMethodError в runtime.

## Репозитории

Gradle ищет JAR-файлы зависимостей в **repositories**, указанных в build.gradle.

```gradle
repositories {
    mavenLocal()                                    // ~/.m2/repository
    maven { url 'https://nexus.isna.internal/…' }  // внутренний
    mavenCentral()                                  // публичный Maven Central
    gradlePluginPortal()                            // плагины
}
```

Порядок важен — Gradle пробует репозитории сверху вниз. `mavenLocal()` полезен только для локальной разработки (когда сам пишешь библиотеку, публикуешь в локальный `.m2`, тут же используешь из другого проекта). На CI использовать `mavenLocal()` — плохая идея (нет гарантии что артефакт есть).

**Enterprise практика — внутренний прокси-репозиторий**. В КНП это `nexus.isna.internal`. Nexus проксирует Maven Central (кэширует запрошенные артефакты локально) и хранит собственные артефакты команды (внутренние библиотеки, releases микросервисов). Плюсы: контроль над артефактами (запрещать использование определённых), кэш (скачивание из локальной сети быстрее интернета), возможность работать без внешнего интернета (все зависимости уже кэшированы).

Реальный кавет из КНП: `knp-eaes-master-port-mechanics` — Nexus недоступен, isna-global локально не собрать. Значит внутренние библиотеки живут только в Nexus, снаружи их нет. Это стандартный setup для enterprise.

Настройка credentials для приватных репозиториев обычно через `~/.gradle/gradle.properties`:

```
nexusUsername=myuser
nexusPassword=mypass
```

И в build.gradle:

```gradle
maven {
    url 'https://nexus.isna.internal/repository/maven-releases/'
    credentials {
        username = project.findProperty('nexusUsername') ?: ''
        password = project.findProperty('nexusPassword') ?: ''
    }
}
```

## Task graph и жизненный цикл

Каждое действие в Gradle — **task**. Компиляция — task `compileJava`. Тесты — task `test`. Упаковка JAR — task `jar`. У каждой task есть **inputs** (входные файлы) и **outputs** (выходные файлы), а также **dependencies** (другие tasks, которые должны выполниться раньше).

Все tasks образуют **DAG** (directed acyclic graph). Когда пользователь запускает `./gradlew build`, Gradle находит все tasks от которых `build` зависит транзитивно, топологически сортирует, выполняет по порядку.

Стандартные задачи Java plugin:

- `clean` — удаляет `build/` (сбрасывает результаты предыдущих сборок).
- `compileJava` — компилирует `src/main/java` в `build/classes/java/main/`.
- `processResources` — копирует `src/main/resources` в `build/resources/main/` с обработкой placeholders.
- `classes` — синтетическая task, "все inputs для main classpath готовы" (composite `compileJava` + `processResources`).
- `compileTestJava` — компилирует `src/test/java`.
- `test` — прогоняет JUnit тесты. Depends on `classes` и `testClasses`.
- `jar` — упаковывает classes в JAR.
- `assemble` — все `jar`/`war`/etc. tasks (без тестов).
- `check` — все verification tasks (тесты, static analysis).
- `build` — `assemble` + `check`.

С Spring Boot plugin добавляются:

- `bootJar` — собирает fat JAR со всеми зависимостями.
- `bootRun` — запускает приложение через Gradle.
- `bootBuildImage` — собирает Docker image.

При Spring Boot plugin активен, `jar` task обычно disabled (`jar { enabled = false }`), чтобы не было confusion — используем только `bootJar`.

Собственные задачи создаются через `tasks.register`:

```gradle
tasks.register('copyDocs', Copy) {
    from 'docs'
    into 'build/docs'
}

build.dependsOn 'copyDocs'
```

**Жизненный цикл сборки** — три фазы, через которые Gradle всегда проходит.

**Initialization** — Gradle читает `settings.gradle`, определяет структуру проекта, создаёт `Project` объекты для корня и всех подмодулей.

**Configuration** — Gradle выполняет `build.gradle` всех проектов. Ключевое: это КОД, реально выполняется. Регистрирует задачи, применяет плагины, настраивает зависимости. Все `println` в build.gradle сработают. Тяжёлая работа в этой фазе замедляет сборку.

**Execution** — Gradle получает от пользователя список задач для запуска (например `build`), строит DAG, топологически сортирует, запускает задачи по порядку. В этой фазе выполняются реальные `doLast { }` и `doFirst { }` блоки.

Важное правило: **тяжёлую работу помещать в execution phase**, не в configuration. Пример:

```gradle
task heavyTask {
    // Configuration phase - выполнится всегда, даже если task не запускается
    def data = fetchDataFromNetwork()
    
    doLast {
        // Execution phase - только при запуске task
        processData(data)
    }
}
```

Здесь `fetchDataFromNetwork()` выполнится при каждой сборке, независимо от того запущена ли `heavyTask`. Правильно:

```gradle
task heavyTask {
    doLast {
        def data = fetchDataFromNetwork()
        processData(data)
    }
}
```

Разница может составлять секунды vs минуты при configuration большого проекта.

## Incremental build и кэширование

Ключевая часть производительности Gradle — **incremental build**. Gradle отслеживает inputs и outputs каждой task. Если inputs с предыдущего запуска не изменились и outputs существуют, task помечается как UP-TO-DATE и не выполняется — outputs берутся из предыдущего запуска.

Наблюдаемо в выводе `./gradlew build`:

```
> Task :compileJava UP-TO-DATE
> Task :processResources UP-TO-DATE
> Task :classes UP-TO-DATE
> Task :bootJar
> Task :assemble
> Task :test
> Task :check
> Task :build
```

`UP-TO-DATE` — task пропущена, результаты из предыдущего запуска.

Для этого механизма task должна правильно объявить inputs и outputs. Стандартные Java tasks делают это автоматически. Кастомные — нужно явно:

```gradle
task processData {
    inputs.file 'src/data.json'
    outputs.file 'build/processed.txt'
    doLast {
        // process
    }
}
```

Более мощная фича — **build cache**. Gradle хешует inputs task и складывает результат в кэш (локальный `~/.gradle/caches/build-cache-1/` или удалённый). Если ты сделаешь `clean` и снова `build`, task не только пропустятся (потому что UP-TO-DATE не работает — outputs удалены `clean`'ом), но результаты возьмутся из cache без повторного выполнения.

Локальный build cache экономит время при переключении между ветками git. Меняешь ветку — компилируешь. Возвращаешься на предыдущую — те же исходники, cache hit, compile мгновенный.

Удалённый build cache (Gradle Enterprise, DevelocityStudio, self-hosted) — расшаривает cache между команды. Один разработчик собрал — все получают из кэша. Особенно ценно на CI, где build cache позволяет не пересобирать модули, которые не меняются.

Включение в `gradle.properties`:
```
org.gradle.caching=true
```

## Gradle Daemon

Gradle запускается как **daemon** — фоновый JVM-процесс, живущий между вызовами `./gradlew`. Первый вызов долгий (JVM стартует, Gradle инициализируется, конфигурация читается). Второй — быстрый (daemon уже прогрет).

Daemon экономит секунды-минуты на каждый сборочный вызов. Проверить статус: `./gradlew --status`. Остановить: `./gradlew --stop`.

Daemon использует память — типично сотни MB до пары гигабайт. Настраивается через `gradle.properties`:

```
org.gradle.jvmargs=-Xmx2g -XX:MaxMetaspaceSize=512m
```

На CI обычно daemon не даёт выгоды (каждый build запускается на свежей машине). Отключается: `--no-daemon`.

## Параллельное выполнение

Многомодульный проект — модуль A и модуль B независимы, но пока Gradle собирает A, B ждёт. Активация параллельности:

```
org.gradle.parallel=true
org.gradle.workers.max=8
```

Или флаг: `./gradlew build --parallel`.

Параллельность работает на уровне независимых подпроектов. Если между модулями нет зависимости, они собираются параллельно. Если A зависит от B, порядок сохраняется.

Ускорение существенное для больших многомодульных проектов. Для однокомпонентных приложений — ноль эффекта.

## Тесты в Gradle

`test` task запускает JUnit тесты. Настройка:

```gradle
test {
    useJUnitPlatform()                    // JUnit 5
    testLogging {
        events "passed", "skipped", "failed"
    }
    maxParallelForks = 4                  // параллельно
    jvmArgs '-Xmx1g', '-Duser.timezone=Asia/Almaty'
}
```

`useJUnitPlatform()` — использовать JUnit 5 (Jupiter). Для старого JUnit 4 — `useJUnit()`.

Для разных типов тестов (unit, integration) обычно создают отдельные source sets и tasks. Пример для integration:

```gradle
sourceSets {
    integrationTest {
        java.srcDir 'src/integrationTest/java'
        resources.srcDir 'src/integrationTest/resources'
        compileClasspath += sourceSets.main.output
        runtimeClasspath += sourceSets.main.output
    }
}

configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
}

task integrationTest(type: Test) {
    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath
    useJUnitPlatform()
}

check.dependsOn integrationTest
```

Теперь `./gradlew integrationTest` запускает интеграционные тесты, `./gradlew build` включает их через `check`.

## Публикация артефактов

Для внутренних библиотек, используемых другими проектами, нужна публикация в артефакт-репозиторий.

Плагин `maven-publish`:

```gradle
plugins {
    id 'maven-publish'
}

publishing {
    publications {
        maven(MavenPublication) {
            from components.java
            groupId = 'kz.gov.kgd.isna'
            artifactId = 'isna-common-lib'
            version = '1.0.42'
        }
    }
    repositories {
        maven {
            name = 'nexus'
            url = 'https://nexus.isna.internal/repository/maven-releases/'
            credentials {
                username = project.findProperty('nexusUser') ?: ''
                password = project.findProperty('nexusPass') ?: ''
            }
        }
    }
}
```

`./gradlew publish` собирает и заливает.

Для SNAPSHOT-версий (в разработке) — отдельный репозиторий `maven-snapshots`. SNAPSHOT версии могут перезаписываться, releases — нет (immutable).

## Практический пример: build.gradle в КНП стиле

Корневой `build.gradle` многомодульного проекта:

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.2.4.RELEASE' apply false
    id 'io.spring.dependency-management' version '1.0.9.RELEASE' apply false
}

allprojects {
    group = 'kz.gov.kgd.isna'
    version = '1.0.42'

    repositories {
        maven { url 'https://nexus.isna.internal/repository/maven-public/' }
    }
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'io.spring.dependency-management'

    sourceCompatibility = JavaVersion.VERSION_21

    dependencyManagement {
        imports {
            mavenBom 'org.springframework.boot:spring-boot-dependencies:2.2.4.RELEASE'
            mavenBom 'org.springframework.cloud:spring-cloud-dependencies:Hoxton.SR3'
            mavenBom 'kz.gov.kgd.isna:isna-global-21:1.2.18'
        }
    }

    test {
        useJUnitPlatform()
    }
}
```

`apply false` — плагин объявлен, но не применен к root проекту (root — только оркестратор, не собирает свой JAR). Подмодули сами применяют где нужно.

Подмодуль `isna-knp-integration/build.gradle`:

```gradle
apply plugin: 'org.springframework.boot'

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.cloud:spring-cloud-starter-consul-discovery'
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'kz.gov.kgd.isna:isna-commons-21'
    implementation project(':isna-knp-shared')

    runtimeOnly 'org.postgresql:postgresql'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

bootJar {
    archiveFileName = 'isna-knp-integration.jar'
}
```

Здесь `implementation project(':isna-knp-shared')` — зависимость на локальный модуль. Gradle резолвит его в build output другого подмодуля.

## Полезные команды для повседневного использования

```bash
./gradlew tasks                              # список задач
./gradlew tasks --all                        # включая скрытые
./gradlew dependencies                       # дерево зависимостей корня
./gradlew :module:dependencies --configuration runtimeClasspath
./gradlew build --info                       # verbose логи
./gradlew build --debug                      # очень verbose
./gradlew build -x test                      # собрать без тестов
./gradlew clean build                        # с нуля
./gradlew build --refresh-dependencies       # игнорировать кэш dependency resolution
./gradlew build --no-daemon                  # без демона
./gradlew build --scan                       # публикация build scan
./gradlew build --parallel                   # параллельная сборка
./gradlew wrapper --gradle-version=8.5       # обновить wrapper
```

`--info` показывает какие таски выполняются, какие пропускаются, время каждой. Полезно для оптимизации медленной сборки.

`--scan` публикует детальный анализ сборки на scans.gradle.com — время каждой task, dependencies, environment. Отличный инструмент для performance analysis.

## Заключение

Gradle — мощная система сборки, освоение которой окупается многократно. Понимание конфигураций зависимостей (implementation vs api vs runtimeOnly), различий между BOM и явными версиями, task graph и жизненного цикла (initialization → configuration → execution), incremental build и cache mechanics — база, без которой сложно работать с многомодульным enterprise проектом.

Ключевые практики. Всегда `./gradlew`, никогда системный `gradle` — воспроизводимость через wrapper. `implementation` по умолчанию, `api` только когда типы уходят в публичный API — снижение coupling. BOM для согласованных версий — Spring Boot BOM плюс собственный (например, `isna-global-21`). Custom tasks через `tasks.register`, тяжёлая работа в `doLast { }`, не в configuration phase.

Оптимизация производительности сборки. Incremental build работает автоматически если inputs/outputs правильно объявлены. Build cache (локальный и удалённый) экономит на повторных сборках. Daemon держит JVM прогретой между вызовами. Parallel execution для многомодульных проектов. `--scan` для профилирования.

Для КНП контекста ключевое: понимание BOM `isna-global-21`, работа с Nexus как внутренним репозиторием, паттерны многомодульного проекта. Каждый сервис — свой подмодуль, общие библиотеки в отдельных модулях (`isna-commons`, `isna-common-lib`).

Дальше — практика. Возьми существующий проект в КНП, разберись как он собран. Прогони `./gradlew build --scan` — увидишь детальный анализ. Прогони `./gradlew :module:dependencies --configuration runtimeClasspath` — увидишь дерево. Попробуй добавить новую библиотеку, разрешить конфликт версий через exclude или force. Каждое такое упражнение даёт больше понимания чем часы чтения.
