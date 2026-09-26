# 04. JAR и Fat JAR: как Java-приложения упаковываются

## Зачем разбираться в устройстве JAR

JAR-файл — контейнер, в котором Java-приложение живёт с момента компиляции до запуска. Ты видишь `myapp.jar` в `build/libs/`, загружаешь его в Docker image, запускаешь через `java -jar`. Работает. Пока работает — про внутреннее устройство можно не думать.

Но бывают ситуации, когда думать приходится. Приложение не стартует с `ClassNotFoundException` — где-то в classpath не хватает библиотеки, но какой JAR должен был её принести? Стартует, но падает с `NoSuchMethodError` — конфликт версий, какие JAR-ы содержат конфликтующие классы? Docker image слишком большой — что раздуло? Startup медленный — сколько там реально классов, зависимостей?

Ответы на эти вопросы требуют понимания что такое JAR изнутри, чем отличается от WAR, как Spring Boot делает fat JAR, почему нужен layered JAR для Docker, как работает custom classloader для чтения JAR внутри JAR. Эти детали кажутся низкоуровневыми, но регулярно всплывают в реальных production проблемах.

В этом файле разберём JAR подробно. Формат и структура. Манифест и что в нём. Различия JAR и WAR. Fat JAR как способ распространения приложения. Spring Boot подход с JAR-in-JAR. Layered JAR для Docker. Custom classloader от Spring Boot. Типичные проблемы и диагностика.

## Что такое JAR

**JAR (Java ARchive)** — стандартный формат упаковки Java-классов и ресурсов. Технически это обычный **ZIP-архив** с определённой конвенцией внутренней структуры. Если переименовать `.jar` в `.zip`, любой архиватор откроет его — там ZIP-формат внутри.

Отличие от произвольного ZIP — обязательное содержимое `META-INF/MANIFEST.MF` и стандартные соглашения о раскладке classes и resources.

Типичная структура простого JAR:

```
myapp.jar
├── META-INF/
│   └── MANIFEST.MF
├── com/
│   └── example/
│       ├── Main.class
│       ├── Service.class
│       └── util/
│           └── Helper.class
└── application.properties
```

Пакеты Java (`com.example.util`) отображаются на директории. Ресурсы (`.properties`, `.yml`, `.xml`, статические файлы) кладутся в правильные места так, чтобы classloader мог их найти.

Работа с JAR через встроенные утилиты JDK или обычные архиваторы:

```bash
jar tf myapp.jar             # список содержимого
unzip -l myapp.jar           # то же через unzip
jar xf myapp.jar META-INF/MANIFEST.MF   # извлечь один файл
unzip -p myapp.jar META-INF/MANIFEST.MF # напечатать в stdout
```

Размер JAR обычно небольшой для чисто твоего кода — сотни килобайт. Основной размер приходит с зависимостями (см. Fat JAR ниже).

## MANIFEST.MF: метаданные архива

`META-INF/MANIFEST.MF` — обязательный файл в JAR. Текстовый, в формате key-value, каждая пара на своей строке.

Минимальный манифест содержит только версию формата:

```
Manifest-Version: 1.0
Created-By: 21.0.11 (Amazon.com Inc.)
```

Такой JAR — просто библиотека, добавляется в classpath других приложений, не запускается напрямую.

**Executable JAR** — тот, который можно запустить через `java -jar`. Требует ключ `Main-Class`:

```
Manifest-Version: 1.0
Main-Class: com.example.Main
```

При `java -jar myapp.jar` JVM читает манифест, находит `Main-Class`, загружает этот класс, вызывает его метод `main(String[])`.

Другие ключи манифеста, которые могут встречаться.

**`Class-Path`** — список внешних JAR-файлов (относительные пути), которые должны быть в classpath при запуске. Позволяет распространять приложение как основной JAR плюс папка с зависимостями. Устаревший подход — сейчас используется Fat JAR.

**`Implementation-Version`, `Implementation-Vendor`, `Implementation-Title`** — метаданные о версии и авторе. Иногда полезны для runtime интроспекции через `Class.getPackage().getImplementationVersion()`.

**`Sealed`** — маркер что все классы пакета должны идти только из этого JAR. Редко используется.

**`Start-Class`** — используется Spring Boot (см. ниже).

**Особенности формата**. Каждая строка не длиннее 72 байт (плюс CRLF). Длинные значения переносятся с ведущим пробелом на следующей строке. Файл должен заканчиваться пустой строкой (иначе `Invalid manifest`).

Пропуск пустой строки в конце — классическая ошибка при ручной генерации манифеста. Утилиты сборки (Gradle, Maven) делают это правильно автоматически.

## Что происходит при `java -jar`

Разберём цепочку событий. Команда `java -jar myapp.jar`:

JVM стартует, парсит аргументы, включая флаг `-jar`.

Открывает `myapp.jar` как ZIP-архив.

Читает `META-INF/MANIFEST.MF`. Парсит key-value pairs.

Ищет ключ `Main-Class`. Если отсутствует — ошибка "no main manifest attribute".

Устанавливает `java.class.path` в путь к JAR (не файловая система, а внутренний classpath).

Через **System ClassLoader** загружает класс, указанный в `Main-Class`.

Вызывает его метод `main(String[] args)`, передавая аргументы командной строки после имени JAR.

Дальше — обычное выполнение Java-программы. Если Main-Class нужны другие классы из этого же JAR, они загружаются лениво при первом использовании. Загрузка через тот же System ClassLoader, который умеет читать содержимое JAR.

**Проблема одиночного JAR** становится очевидной когда приложение зависит от библиотек. Твой код использует Spring, Jackson, Hibernate, PostgreSQL JDBC driver. Каждая — отдельный JAR со своими классами. Просто положить твой код в JAR — при первом же `new ObjectMapper()` будет `ClassNotFoundException`, потому что Jackson не в classpath.

Раньше это решали через `Class-Path` в манифесте или явный classpath при запуске:

```bash
java -cp "myapp.jar:lib/spring-web.jar:lib/jackson-databind.jar:..." com.example.Main
```

Неудобно: два артефакта (основной JAR плюс папка с зависимостями), нужно синхронизировать, легко потерять. Классический подход был WAR + application server, но и он неудобен для микросервисов.

Современное решение — **Fat JAR**.

## WAR: исторический контекст

Прежде чем к Fat JAR, коротко про WAR для полноты картины.

**WAR (Web ARchive)** — специальный JAR для web-приложений. Появился с началом Java servlets в конце 90-х. Структура фиксированная:

```
myapp.war
├── META-INF/MANIFEST.MF
├── WEB-INF/
│   ├── web.xml               ← дескриптор servlet-приложения
│   ├── classes/              ← твои классы (не в корне!)
│   │   └── com/example/
│   └── lib/                  ← библиотеки
│       ├── spring-web.jar
│       └── …
└── index.html                ← статика в корне
```

Классы приложения — не в корне, а в `WEB-INF/classes/`. Библиотеки — в `WEB-INF/lib/`. Статические ресурсы (HTML, CSS, JS, картинки) — прямо в корне архива.

Как работает. Есть отдельно установленный **application server** — Tomcat, WildFly, WebSphere, GlassFish. Кидаешь `.war` в его `webapps/` директорию. Сервер разворачивает архив, читает `web.xml`, регистрирует servlets, начинает обслуживать HTTP-запросы.

Плюсы модели application server. Один сервер обслуживает несколько приложений. Общие ресурсы (connection pools, JMS) настраиваются в сервере. Единая точка администрирования.

Минусы, из-за которых WAR практически исчез в микросервисной эре. Два артефакта (сервер + war) требуют синхронизации версий. Один сервер = single point of failure для нескольких приложений. Общая JVM = один плохой приложение может задавить остальные. Сложный процесс деплоя. Специфические для application server особенности.

Spring Boot развернул мир: **JAR со встроенным сервером** оказался гораздо удобнее для микросервисов. Один артефакт, `java -jar myapp.jar`, приложение работает. Никакого application server не нужно — сервер (Tomcat/Jetty/Undertow) встроен внутрь JAR.

В КНП все микросервисы — Spring Boot fat JAR со встроенным Tomcat. WAR редко встречается, обычно только в legacy интеграциях.

## Fat JAR: идея

Идея Fat JAR (иногда называют Uber JAR) — упаковать в один архив **и код приложения, и все его зависимости**. Один файл, `java -jar myapp.jar`, всё работает. Никакого внешнего classpath, никаких потерянных lib/ директорий.

Есть два принципиально разных подхода реализации.

**Наивный подход — Shadow/Shade**. Плагины Gradle Shadow или Maven Shade распаковывают каждую JAR-зависимость и складывают классы в один общий архив. Результат:

```
myapp-uber.jar
├── com/example/…               ← твои классы
├── org/springframework/…       ← Spring внутри
├── com/fasterxml/jackson/…     ← Jackson внутри
├── org/hibernate/…             ← Hibernate внутри
└── META-INF/…
```

Все classpath entries слиты воедино. Работает через стандартный system classloader — он видит все классы просто.

Проблемы этого подхода. **Конфликты имён**. Разные библиотеки могут содержать классы с одним пакетом+именем. При слиянии одни перезаписываются другими. Обычно это симптом плохо структурированных зависимостей, но случается.

**Конфликты ресурсов `META-INF/services/`**. Java Service Loader Mechanism использует эти файлы для регистрации service implementations. Spring использует `META-INF/spring.factories` для auto-configuration. Разные JAR-ы могут иметь эти файлы с разным содержимым. При простом слиянии одни перезаписывают другие, часть configuration теряется. Нужны специальные merger'ы (`ServicesResourceTransformer`), объединяющие эти файлы правильно.

**Конфликты licenses и notices**. `META-INF/LICENSE` и `META-INF/NOTICE` есть в большинстве open-source JAR-ов. При слиянии — какой оставить?

Всё это требует специальной обработки при сборке. Работает, но fragile.

**Spring Boot подход — JAR-in-JAR**. Радикально другое решение: не распаковывать зависимости, оставить их целыми JAR-ами внутри контейнера.

## Spring Boot fat JAR: устройство

Структура типичного Spring Boot fat JAR:

```
myapp.jar
├── META-INF/
│   └── MANIFEST.MF
│       Manifest-Version: 1.0
│       Main-Class: org.springframework.boot.loader.JarLauncher
│       Start-Class: kz.gov.kgd.isna.knp.KnpApplication
│       Spring-Boot-Version: 2.2.4.RELEASE
│       Spring-Boot-Classes: BOOT-INF/classes/
│       Spring-Boot-Lib: BOOT-INF/lib/
│
├── org/springframework/boot/loader/     ← классы лончера Spring Boot
│   ├── JarLauncher.class
│   ├── LaunchedURLClassLoader.class
│   └── … (десятки классов loader'а)
│
├── BOOT-INF/
│   ├── classes/                          ← ТВОИ .class-файлы
│   │   ├── kz/gov/kgd/isna/knp/
│   │   │   ├── KnpApplication.class
│   │   │   └── controller/…
│   │   └── application.yml               ← ресурсы
│   │
│   ├── lib/                              ← вложенные JAR-ы, целиком
│   │   ├── spring-boot-2.2.4.RELEASE.jar
│   │   ├── spring-web-5.2.3.jar
│   │   ├── postgresql-42.2.jar
│   │   ├── jackson-databind-2.10.jar
│   │   └── … (обычно сотни)
│   │
│   └── classpath.idx                     ← индекс порядка classpath
│
└── (иногда) BOOT-INF/layers.idx          ← для layered jar
```

Три ключевые директории. `org/springframework/boot/loader/` — код Spring Boot loader (JarLauncher, LaunchedURLClassLoader и вспомогательные). Он лежит в корне JAR, потому что должен быть доступен стандартному JVM classloader'у.

`BOOT-INF/classes/` — твои скомпилированные classes и ресурсы. Всё что было в `src/main/java` и `src/main/resources`.

`BOOT-INF/lib/` — все зависимости целыми JAR-файлами. Spring core, Jackson, PostgreSQL driver — каждый как отдельный JAR внутри контейнера. Обычно 100-300 JAR-ов для типичного микросервиса.

Такая структура даёт несколько преимуществ. Никаких конфликтов имён — каждый JAR живёт в своём пространстве. Никаких проблем со `META-INF/services` — они остаются внутри своих JAR-ов, читаются правильно при runtime. Ясная диагностика — `unzip -l myapp.jar | grep '^BOOT-INF/lib/'` показывает точный список зависимостей.

Проблема этого подхода — стандартный JVM classloader НЕ УМЕЕТ читать классы из JAR внутри JAR. Стандартный `URLClassLoader` понимает `jar:file:/path/to/lib/spring.jar!/org/springframework/...` (класс внутри JAR), но не `jar:file:/path/to/myapp.jar!/BOOT-INF/lib/spring.jar!/org/springframework/...` (класс во вложенном JAR, двойной `!`). Обычная JVM просто не найдёт classes.

## MANIFEST.MF Spring Boot fat JAR

Обратим внимание на манифест:

```
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: kz.gov.kgd.isna.knp.KnpApplication
Spring-Boot-Version: 2.2.4.RELEASE
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
```

`Main-Class` — это НЕ твой класс. Это специальный `JarLauncher` от Spring Boot. Он находится в корне JAR (не в `BOOT-INF/`), поэтому стандартный system classloader может его загрузить.

`Start-Class` — это твой класс с `main` методом. JarLauncher его запустит после подготовки правильного classloader'а.

`Spring-Boot-Classes` и `Spring-Boot-Lib` — префиксы путей, где находятся твои классы и библиотеки. Используются JarLauncher'ом.

## Что происходит при запуске

Разберём пошагово что происходит при `java -jar myapp.jar` для Spring Boot fat JAR.

**Шаг 1**: JVM стартует, открывает JAR как ZIP, читает манифест. Видит `Main-Class: JarLauncher`.

**Шаг 2**: JVM загружает JarLauncher через свой стандартный system classloader. JarLauncher лежит в корне JAR (`org/springframework/boot/loader/JarLauncher.class`), стандартный classloader его находит. Вызывает `JarLauncher.main(args)`.

**Шаг 3**: JarLauncher выполняет специальную инициализацию. Читает свой собственный JAR как ZIP. Ищет все вложенные JAR-ы в `BOOT-INF/lib/`. Строит список URL: `jar:file:/path/to/myapp.jar!/BOOT-INF/classes/` (твои classes), `jar:file:/path/to/myapp.jar!/BOOT-INF/lib/spring-web.jar!/`, `jar:file:/path/to/myapp.jar!/BOOT-INF/lib/jackson-databind.jar!/` и так далее.

**Шаг 4**: JarLauncher создаёт **LaunchedURLClassLoader** с этим списком URL. LaunchedURLClassLoader расширяет стандартный URLClassLoader, но переопределяет чтение — умеет открывать вложенные JAR-ы через кастомный URL-хендлер `jar:` с поддержкой двойного `!/`.

**Шаг 5**: JarLauncher читает `Start-Class` из манифеста. Через LaunchedURLClassLoader загружает `KnpApplication`. Теперь classloader знает про все зависимости — при запросе класса из Spring или Jackson находит его во вложенном JAR.

**Шаг 6**: JarLauncher вызывает через reflection метод `main(String[])` класса KnpApplication, передавая аргументы командной строки.

**Шаг 7**: Управление переходит в код приложения. `SpringApplication.run(...)` вызывается, начинается обычная жизнь Spring Boot.

С этого момента всё "работает как обычно". Classloader настроен правильно, все classes доступны, никаких ClassNotFoundException. Spring находит auto-configuration классы во вложенных JAR-ах через стандартный classloading механизм.

## LaunchedURLClassLoader изнутри

Технически LaunchedURLClassLoader — это extension standard URLClassLoader. Ключевая часть — регистрация custom URL Stream Handler для протокола `jar:`.

Стандартный `sun.net.www.protocol.jar.Handler` умеет открывать `jar:file:/path/lib.jar!/entry` — один уровень вложенности. Spring Boot регистрирует свой handler, парсящий цепочки `!/.../!/` и рекурсивно открывающий вложенные ZIP-стримы.

Когда classloader пытается загрузить, например, `org.springframework.web.servlet.DispatcherServlet`:

Проходит по своему списку URL. Для каждого URL пытается найти класс. URL вроде `jar:file:/path/myapp.jar!/BOOT-INF/lib/spring-webmvc.jar!/` требует открыть внешний JAR (myapp.jar), внутри него найти `BOOT-INF/lib/spring-webmvc.jar`, открыть его как inner ZIP, найти `org/springframework/web/servlet/DispatcherServlet.class`.

Custom URL handler делает эту рекурсивную работу. Открывает myapp.jar как ZIP, находит запись `BOOT-INF/lib/spring-webmvc.jar`, читает её как stream, оборачивает в `JarInputStream`, ищет нужный entry.

Всё это прозрачно для основного кода Java. С точки зрения приложения, classloading работает "как обычно". Просто есть magic, обеспечивающий это.

## Layered JAR: оптимизация для Docker

С Spring Boot 2.3 появилась ещё одна оптимизация — **layered JAR**. Это модификация структуры fat JAR, разделяющая содержимое на логические слои по частоте изменений.

Проблема, которую решает. Стандартный fat JAR — один большой файл (обычно 50-200 MB для микросервиса). При деплое в Docker весь JAR кладётся в один Docker layer. Меняешь одну строчку кода — весь layer инвалидируется, весь JAR пересобирается и передаётся заново. Медленно, дорого по bandwidth.

Идея layered JAR — разделить содержимое на несколько логически осмысленных групп, отсортированных по частоте изменений:

- **dependencies** — обычные зависимости (Spring, Jackson, etc.). Меняются редко — при обновлении версии Spring Boot или добавлении новой библиотеки.
- **spring-boot-loader** — код Spring Boot loader (JarLauncher, LaunchedURLClassLoader). Меняется очень редко — при обновлении Spring Boot версии.
- **snapshot-dependencies** — SNAPSHOT-версии зависимостей. Меняются чаще (если используются).
- **application** — твой код и ресурсы. Меняется постоянно.

В manifest добавляется `BOOT-INF/layers.idx`, описывающий слои:

```
- "dependencies":
  - "BOOT-INF/lib/"
- "spring-boot-loader":
  - "org/springframework/boot/loader/"
- "snapshot-dependencies":
- "application":
  - "BOOT-INF/classes/"
  - "BOOT-INF/classpath.idx"
  - "BOOT-INF/layers.idx"
  - "META-INF/"
```

Утилита `layertools` (встроена в Spring Boot loader) может извлечь эти слои в отдельные директории:

```bash
java -Djarmode=layertools -jar myapp.jar extract
```

Результат — четыре директории: `dependencies/`, `spring-boot-loader/`, `snapshot-dependencies/`, `application/`. Каждая — часть исходного JAR.

## Layered JAR в Dockerfile

Использование layered JAR в multi-stage Dockerfile:

```dockerfile
FROM amazoncorretto:21 AS builder
WORKDIR /workspace
COPY build/libs/myapp.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM amazoncorretto:21
WORKDIR /app
COPY --from=builder /workspace/dependencies/ ./
COPY --from=builder /workspace/spring-boot-loader/ ./
COPY --from=builder /workspace/snapshot-dependencies/ ./
COPY --from=builder /workspace/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

Каждый `COPY --from=builder` — отдельный Docker layer. Docker кэширует слои по content hash. Если ты изменил только код приложения (не зависимости), при пересборке:

`dependencies/` — content не изменился, layer из кэша. Не пересобирается, не передаётся.

`spring-boot-loader/` — то же.

`snapshot-dependencies/` — то же (если нет SNAPSHOT изменений).

`application/` — изменилось, layer пересобирается. Обычно 5-20 MB вместо 50-200 MB.

Docker push и pull работают только с изменёнными слоями. Экономия огромная при частых деплоях.

Включение layered JAR в build.gradle:

```gradle
bootJar {
    layered {
        enabled = true
    }
}
```

С Spring Boot 2.4+ по умолчанию включено.

## Практические команды

Полезные операции с Spring Boot JAR.

**Просмотр структуры**:

```bash
jar tf myapp.jar | head -30
```

Показывает первые 30 записей. Увидишь Spring Boot loader классы в корне, `BOOT-INF/classes/`, `BOOT-INF/lib/`.

**Просмотр манифеста**:

```bash
unzip -p myapp.jar META-INF/MANIFEST.MF
```

Проверить `Main-Class` (должен быть JarLauncher), `Start-Class` (твой класс).

**Подсчёт зависимостей**:

```bash
jar tf myapp.jar | grep '^BOOT-INF/lib/' | wc -l
```

Обычно 100-300 для микросервиса.

**Список зависимостей**:

```bash
jar tf myapp.jar | grep '^BOOT-INF/lib/' | sort
```

**Размер JAR**:

```bash
ls -lh myapp.jar
```

Типично 80-150 MB для среднего Spring Boot приложения. Большая часть — зависимости.

**Размер отдельных зависимостей**:

```bash
unzip -l myapp.jar | grep '^BOOT-INF/lib/' | sort -k1 -n | tail -20
```

Топ-20 самых больших JAR-ов внутри. Часто оказываются неожиданные "толстые" библиотеки.

**Извлечение файла**:

```bash
jar xf myapp.jar BOOT-INF/classes/application.yml
```

## Диагностика типовых проблем

**`ClassNotFoundException: com.example.SomeClass`** при запуске.

Класс не в classpath. Проверить наличие в JAR:

```bash
jar tf myapp.jar | grep 'SomeClass'
```

Если пусто — зависимость не попала в `BOOT-INF/lib/`. Проверить `./gradlew dependencies` — есть ли она вообще в проекте. Возможно `compileOnly` вместо `implementation`. Возможно исключена через `exclude`.

**`Invalid or corrupt jarfile`**.

MANIFEST повреждён. Причины: ручное редактирование с ошибками (нет пустой строки в конце, кривая кодировка), поврежденный файл при загрузке. Пересобрать через `./gradlew clean bootJar`.

**`NoSuchMethodError: com.example.SomeClass.method()`**.

Конфликт версий одного класса из разных JAR-ов. Разные JAR-ы содержат разные версии одного класса. При runtime classloader берёт первый попавшийся, у него не тот метод. Диагностика:

```bash
for f in $(jar tf myapp.jar | grep '^BOOT-INF/lib/.*\.jar$'); do
    unzip -p myapp.jar "$f" | jar tf /dev/stdin 2>/dev/null | grep 'SomeClass' && echo "  <- $f"
done
```

Покажет все JAR-ы, содержащие SomeClass. Обычно проблема в том, что две разные библиотеки принесли разные версии. Решение — exclude одной, force версии через Gradle.

Реальный кейс из КНП: `taxreport21-java21-runtime-regressions` — spring-security jose и core оказались разных версий. Одна библиотека принесла spring-security-jose 5.4, другая — 5.6 через транзитивную зависимость. Classloader брал одну, вызывался метод из другой. Fix — явный force версии в build.gradle.

**Приложение стартует, но 30+ секунд**.

Много условной auto-configuration с проверкой множества условий. Плюс много зависимостей. Actuator (Spring Boot 2.4+) даёт `/actuator/startup` endpoint — показывает что грузится долго. Или анализ через `-Ddebug` при запуске — покажет какие autoconfig-классы включились, какие пропущены.

## Executable JAR как shell script

Малоизвестный трюк Spring Boot — сделать JAR исполняемым как shell-скрипт.

```gradle
bootJar {
    launchScript()
}
```

Тогда:

```bash
chmod +x myapp.jar
./myapp.jar start
./myapp.jar stop
./myapp.jar status
```

Внутри — bash-заголовок с логикой управления (start/stop/status), потом бинарные байты JAR. Форма гибрид: shell-скрипт наверху, ZIP-архив снизу. Оба валидны с своих точек зрения.

`java -jar` тоже работает — JVM игнорирует shell prefix и парсит ZIP.

Использование редкое, обычно для standalone серверов без Docker. В контейнерах не нужно.

## Терминология: fat, uber, shaded, layered

Часто эти термины используют как синонимы, но есть нюансы.

**Fat JAR** — общее понятие "толстый JAR с зависимостями внутри". Собирательное.

**Uber JAR** — обычно про Maven Shade Plugin. Все зависимости распакованы и слиты в один архив.

**Shaded JAR** — uber JAR с **relocation** (переименованием) пакетов. Используется для избежания конфликтов с runtime-окружением. Например, библиотека использует свою версию Google Guava, при shade Guava пакеты переименовываются в `com.example.shaded.guava`, чтобы не конфликтовать с Guava в classpath приложения.

**Spring Boot fat JAR** — специфическая структура с BOOT-INF/lib/, зависимости целыми JAR-ами.

**Layered JAR** — Spring Boot fat JAR с добавлением `layers.idx` для Docker оптимизации.

Разница важна при обсуждении структуры. В КНП используется Spring Boot layered JAR — это стандартный выбор для Spring Boot микросервисов на 2020-х.

## Заключение

JAR — фундаментальный формат Java-экосистемы. Простой снаружи (ZIP с манифестом), с богатой историей и вариациями. Понимание внутреннего устройства помогает диагностировать production проблемы, оптимизировать deploy pipeline, разбираться в сложных случаях конфликтов зависимостей.

Ключевые моменты для запоминания.

Обычный JAR — ZIP с `META-INF/MANIFEST.MF`. Executable JAR имеет `Main-Class`, запускается через `java -jar`.

WAR — специальная структура для servlet application server, легаси в микросервисной эре.

Fat JAR решает проблему распространения приложения с зависимостями. Два подхода: shaded (uber, слияние классов) и Spring Boot (JAR-in-JAR).

Spring Boot fat JAR: код в `BOOT-INF/classes/`, зависимости в `BOOT-INF/lib/` целыми JAR-ами, custom `JarLauncher` + `LaunchedURLClassLoader` для чтения JAR-в-JAR.

Layered JAR оптимизирует Docker: разделение на слои dependencies/loader/snapshot/application, каждый — отдельный Docker layer. Изменение кода не пересобирает dependencies layer.

Типичные проблемы: ClassNotFoundException (зависимость не в classpath), NoSuchMethodError (конфликт версий), duplicate classes. Диагностика через `jar tf`, `unzip -l`, анализ дерева зависимостей.

Для КНП контекста практично: все микросервисы — Spring Boot layered JAR, деплой в Docker с proper layered structure для эффективного кэша. Diагностика конфликтов версий через анализ содержимого BOOT-INF/lib/.

Дальше — практика. Возьми любой fat JAR своего сервиса, разбери что там внутри: сколько зависимостей, какие самые тяжёлые, есть ли duplicate classes. Изучи манифест. Включи layered режим, посмотри разницу в размерах слоёв Docker image до и после. Каждое такое упражнение раскрывает картину лучше чем чтение.
