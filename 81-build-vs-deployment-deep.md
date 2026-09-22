# 81. Build vs Deployment — глубокая теория и все стратегии деплоя

## Зачем это знать

В командной инженерной практике слова «build», «release» и «deploy» смешаны настолько, что почти никто не проводит между ними границу. Все три ставят в один ряд: «мы задеплоили новую сборку». Однако именно понимание разницы между этими концепциями определяет, будет ли ваш CI/CD надёжным, воспроизводимым и предсказуемым — или каждый релиз в прод будет сопровождаться молитвами.

Это не академическое различие. Из неё вытекают вполне практические вещи. Почему тот же самый Docker-образ должен идти от dev через staging в prod, а не пересобираться под каждое окружение? Потому что build → immutable, release = build + config, deploy = активация release. Если пересобрать под каждое окружение — вы не тестируете то, что попадёт в прод. Малейшее изменение (обновлённая транзитивная зависимость, изменение build-хоста) даёт другой binary — и тот прекрасно прошедший интеграционные тесты образ уже не тот, что попал в прод. Все инциденты «работало на staging» приходят отсюда.

Второй пласт — виды деплоя. Rolling, Recreate, Blue-Green, Canary, A/B, Shadow, Dark Launch, Ring — это не список синонимов, а набор ортогональных техник. Каждая решает свою проблему. Rolling — просто заменить версию без даунтайма. Blue-Green — мгновенный rollback. Canary — обкатать новую версию на маленьком проценте трафика. Shadow — прогнать новую версию через реальные запросы, не показывая ответ клиенту. Dark Launch — вкатить код в прод, но включить его для пользователя позже отдельным решением. Все они комбинируются: canary + feature flag, blue-green + shadow, ring + A/B. Senior-инженер должен понимать не только «как это работает в K8s», но и **когда что применять**.

Мы разберём три концепции 12-factor — build/release/deploy — с чётким разделением. Дальше — что делает `gradle build` изнутри, какие бывают Java-артефакты (JAR, fat JAR, WAR, EAR, native image), reproducible builds, build cache. Что такое deploy, топологии (bare metal, VM, container, K8s, serverless), immutable vs mutable, K8s деплой пошагово. Три оси классификации деплоя: как заменяем (in-place/recreate/rolling/blue-green/canary), кто видит новую версию (big bang/ring/A/B/feature flag/dark launch), куда идёт трафик (live/shadow/synthetic). Детальный разбор каждой стратегии — механика, плюсы, минусы, K8s-реализация, когда использовать. Отдельно — БД-миграции при deploy (expand-contract), потому что это самая сложная часть. Environments, registries, rollback стратегии, semver.

## Три концепции: Build, Release, Deploy

Начнём с фундамента. 12-factor app методология выделяет три отдельных фазы, и это разделение — ключевое для здорового CI/CD. Смешивание — источник половины операционных проблем.

**Build** — это процесс, где исходный код превращается в исполняемый артефакт. Из `src/main/java/*.java` получается `.class`-файлы, потом всё собирается в JAR, потом (если контейнеризация) — в Docker image. Ключевое свойство build: **immutable**. Тот же commit собранный дважды должен дать тот же артефакт. Если это не так — у вас supply chain проблема.

**Release** — это build плюс конфигурация для конкретного окружения. Один и тот же `myapp:1.0.42` для dev, staging и prod. Разница только в `application-dev.yml`, `application-staging.yml`, `application-prod.yml`. Release тоже immutable: `myapp:1.0.42 + prod-config-v3 = release-prod-42-v3`.

**Deploy** — это активация release в окружении. Развёртывание Pod'ов, направление трафика, обновление балансировщиков. Deploy эфемерен: можно передеплоить тот же release много раз, результат идентичен.

Практический смысл разделения. Один и тот же `myapp:1.0.42` идёт из dev в staging в prod. В каждом окружении применяется своя конфигурация. Значит, то, что вы протестировали в staging, — это буквально тот же binary, что попадёт в prod. Меняется только конфиг (URL БД, пароли, feature flags). Если что-то работает в staging и не работает в prod — это не разница в коде, а разница в конфигурации или окружении.

Обратный (антипаттерн) подход — «собираем под каждое окружение отдельно». Каждое окружение имеет свой Docker build с настройкой типа `-Denv=prod`. Даже если build детерминирован, транзитивные зависимости могут обновиться, build-хост поменяться, docker base image получить security-патч. В результате `myapp:1.0.42-dev` и `myapp:1.0.42-prod` — это разные бинарники, и тестирование в dev не гарантирует поведение в prod.

Правило: **одна сборка — много релизов**. `myapp:1.0.42` собран один раз, передвигается по environments изменением конфигурации. Именно это разделение позволяет иметь высокую уверенность в том, что попадает в prod.

## Что такое build детально

Build — это трансформация. Входы: исходный код, resource-файлы, build-скрипты, объявления зависимостей. Выходы: артефакт, готовый к запуску.

Что реально делает Gradle или Maven на команде `build`. Разберём по шагам, потому что каждый шаг может стать точкой failure или оптимизации.

**Dependency resolution** — читается `build.gradle` (или `pom.xml`), из него извлекаются все `implementation`, `compile`, `testImplementation` объявления. Идёт запрос в Maven Central или (для enterprise) в приватный Nexus/Artifactory. Скачиваются JAR-файлы всех зависимостей и их транзитивных зависимостей. Разрешаются конфликты версий — здесь Maven и Gradle расходятся: Maven использует nearest-wins (побеждает версия, объявленная ближе к корню графа зависимостей), Gradle — highest-wins (побеждает наибольшая версия). При enterprise разработке зависимости обычно кэшируются в приватном registry (Nexus в КНП), поэтому этот шаг быстрый после первого раза.

**Compile Java** — собственно компиляция. Java source (`.java`) превращается в bytecode (`.class`). Компилятор `javac` проверяет типы, разрешает импорты, применяет annotation processors (Lombok, MapStruct, Spring). Отдельно компилируются main-код (`compileJava`) и test-код (`compileTestJava`).

**Process resources** — копируются файлы из `src/main/resources/` в `build/resources/main/`. Здесь же делается replacement placeholder'ов, если настроен: `@version@` → фактическая версия проекта, например. Это позволяет embed'ить build metadata в приложение.

**Run tests** — task `test` собирает test classpath, запускает JUnit engine. Каждый `@Test` метод — отдельный test. Отчёты пишутся в `build/test-results/test/*.xml` (стандартный JUnit XML format). Если хоть один тест failed, build фейлится.

**Static analysis** — если настроены плагины: `checkstyle`, `spotbugs`, `pmd`, `jacoco`. Каждый анализирует код на свои паттерны: checkstyle — стиль (indentation, naming), spotbugs — potential bugs (null pointer, thread safety), jacoco — coverage. Их можно сделать build-breaking или warning-only.

**Packaging** — собственно создание артефакта. Для обычного JAR — упаковываются `.class` файлы плюс `META-INF/MANIFEST.MF`. Для Spring Boot fat JAR — `bootJar` task делает своё: кладёт main-код в `BOOT-INF/classes/`, все dependencies в `BOOT-INF/lib/`, добавляет Spring Boot loader. Итог — самодостаточный jar, который запускается через `java -jar`.

**Publishing** (опционально) — если это библиотека, uploadится в Nexus/Artifactory для использования другими проектами. Для приложений обычно не делается — приложение упаковывается сразу в Docker image.

## Артефакты Java-мира

Знать все виды упаковок стоит, потому что за каждым — свои trade-offs.

**`.class` файл** — минимальная единица. Один класс = один файл. Содержит bytecode (JVM instruction set), constant pool (строковые константы, ссылки на классы), метаданные (annotations, generics info). Обычный `.class` не запускается напрямую — нужен classpath со всеми зависимыми классами.

**JAR (обычный)** — Java Archive. По сути zip-архив с `.class` файлами внутри плюс `META-INF/MANIFEST.MF` (metadata) плюс возможно resources. Используется как **библиотека**: если ваш проект зависит от `guava-33.jar`, вы подключаете этот JAR в classpath, JVM classloader находит нужные классы внутри. Для запуска приложения одного JAR обычно недостаточно — нужен весь classpath.

**Fat JAR / Uber JAR** — JAR со всеми зависимостями внутри, самодостаточный. Запускается через `java -jar app.jar` — JVM видит `Main-Class` в манифесте, запускает его. Проблема: обычный ClassLoader не умеет читать JAR внутри JAR (nested JARs). Разные подходы:

- **Shade plugin (Maven)** — распаковывает все JARs и склеивает в один плоский JAR. Проблема: конфликты имён файлов (два JARs имеют `META-INF/services/java.sql.Driver`).
- **Spring Boot approach** — свой custom loader (`org.springframework.boot.loader.JarLauncher`), который умеет читать nested JARs. Оригинальные JARs остаются как есть внутри `BOOT-INF/lib/`.

Spring Boot fat JAR структура:

```
myapp.jar
├── META-INF/
│   └── MANIFEST.MF          Main-Class: org.springframework.boot.loader.JarLauncher
├── BOOT-INF/
│   ├── classes/              ← твой код
│   │   ├── com/isna/knp/...
│   │   └── application.yml
│   ├── lib/                  ← все transitive deps
│   │   ├── spring-boot-3.2.0.jar
│   │   └── ... (150+ файлов)
│   └── classpath.idx
└── org/springframework/boot/loader/  ← Boot launcher
    └── JarLauncher.class
```

`java -jar myapp.jar` — JVM launcher находит `Main-Class` — это `JarLauncher`. JarLauncher настраивает custom classloader, который может читать nested JARs, потом запускает актуальный main метод приложения. Всё прозрачно, но под капотом — нетривиальная gymnastika.

**WAR (Web Application Archive)** — специализированный формат для web-приложений. Структура фиксирована: `WEB-INF/classes/` (код), `WEB-INF/lib/` (зависимости), `WEB-INF/web.xml` (deployment descriptor). Разворачивается во внешний сервлет-контейнер (Tomcat, Jetty, WildFly). Сейчас устаревающий формат — Spring Boot fat JAR со встроенным Tomcat практически заменил его. WAR остаётся для legacy систем и там, где есть отдельная operations команда, поддерживающая пул application-серверов.

**EAR (Enterprise Application Archive)** — для JavaEE application servers (WebSphere, WebLogic, JBoss EAP). Может содержать несколько WAR + EJB modules + shared libraries. Практически мёртв — только очень legacy.

**Native Image (GraalVM)** — совсем другой подход. Не bytecode, а полноценный native executable, скомпилированный AOT (Ahead-Of-Time). Запускается за десятки миллисекунд вместо секунд. Проблема: reflection, dynamic classloading, ClassLoaders — всё, что делает Java гибким — требует специальных hints (файлы `reflect-config.json`, `resource-config.json`). Не все библиотеки поддерживают. Идеально для serverless (быстрый cold start), embedded (маленький footprint), CLI-инструментов. Не универсальная замена обычного JVM.

## Reproducible builds — зачем и как

Reproducible build — свойство: тот же commit собранный дважды на разных машинах даёт **бинарно идентичный** артефакт. `diff -q built1.jar built2.jar` даёт empty output.

Зачем это нужно. Первое — security. Если ваш prod содержит `myapp-1.0.42.jar` с определённым SHA-256, вы можете взять commit `abc123`, собрать локально на своей машине, и проверить: получился ли тот же SHA-256? Если да — код в проде действительно из этого commit. Если нет — где-то supply chain compromise, и prod содержит нечто, что не из вашего git.

Второе — воспроизводимость. Каждый раз, когда пересобираете старый релиз для hotfix'а, получаете **точно** тот же артефакт. Меньше surprise'ов.

Проблема: обычный build не reproducible. Причины:

- **Таймстампы в файлах**: `Last-Modified` в MANIFEST.MF, mtime записей в zip.
- **Порядок файлов в архиве**: зависит от файловой системы (какой inode, какие file listing operations).
- **Build metadata**: `Build-Time`, `Built-By` (username), `Build-Jdk` (JDK version) — включаются автоматически.
- **Локаль/timezone build-хоста**: строки может форматироваться по-разному.

Как достичь reproducible:

- Gradle: `-Dorg.gradle.reproducible-archives=true`.
- Установить `SOURCE_DATE_EPOCH` env var (стандарт GNU для reproducibility).
- Убрать `Build-Time`, `Built-By` из MANIFEST через `manifest.attributes.remove(...)`.
- Использовать `preserveFileTimestamps = false` и `reproducibleFileOrder = true` в Gradle jar task.
- Docker: `--build-arg SOURCE_DATE_EPOCH=<timestamp>` плюс BuildKit reproducible mode.

Практически в enterprise reproducible builds — nice-to-have, но не критично. Если вы используете private registry с immutable tags и content-addressable digests, вы уже имеете доказательство «то, что в проде, — то, что pushed в registry». Reproducibility даёт дополнительный уровень: «то, что в registry — то, что из git».

## Build caching — где ускорение

Build чистого проекта с нуля может занимать минуты. При инкрементальной работе — нужно секунды. Разница обеспечивается кэшированием.

**Локальный Gradle cache** (`~/.gradle/caches`). Каждая Gradle task имеет `inputs` (файлы, свойства) и `outputs` (файлы). Cache сохраняет outputs по hash inputs. Второй запуск с теми же inputs — outputs достаются из cache, task не выполняется. Помечается как `UP-TO-DATE` или `FROM-CACHE` в выводе.

**Remote build cache** — тот же принцип, но кэш доступен всей команде через HTTP. Gradle Enterprise, Bazel remote cache, self-hosted. Коллега собрал task X → cache в remote → вы делаете тот же коммит локально → cache hit, task достаётся почти мгновенно.

Реальный эффект: чистый build одного микросервиса КНП без cache — 5-8 минут. С local cache — 30 секунд-2 минуты. С remote cache — секунды.

**CI cache** — GitLab CI кладёт `.gradle/caches` в артефакт между pipeline runs. Второй pipeline на той же ветке — быстрее. Настраивается через `cache:` секцию в `.gitlab-ci.yml`.

**Docker layer cache** — каждая инструкция в Dockerfile (`FROM`, `COPY`, `RUN`) создаёт отдельный layer. Layer identified через hash своих inputs. Если инструкция и её inputs не менялись — layer переиспользуется из cache. Правило: **редко меняющееся раньше, часто меняющееся позже**. Пример для Spring Boot layered JAR:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
# Deps редко меняются
COPY build/libs/dependencies/ ./
COPY build/libs/spring-boot-loader/ ./
COPY build/libs/snapshot-dependencies/ ./
# Твой код часто меняется — последний слой
COPY build/libs/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

Изменил Java-файл → все слои до `application/` — cache hit, только последний rebuild. `docker push` шлёт только новый layer (~10MB), не весь image.

Spring Boot 2.3+ имеет `bootBuildImage` — использует Cloud Native Buildpacks (Paketo) для сборки image без Dockerfile. Buildpacks автоматически применяют layered JAR structure, оптимизируют memory settings для JVM, устанавливают non-root user. Плюс — security patches автоматически (rebuild с новым buildpack version = новый base image). Минус — меньше контроля.

## Build vs Compile vs Package vs Assemble

Термины часто смешивают, но в Gradle/Maven они имеют точные значения.

**Compile** — только `.java → .class`. Это одна конкретная фаза. `compileJava` task в Gradle. Не включает тесты, не создаёт JAR.

**Package** — упаковка `.class` файлов и resources в JAR/WAR. `jar` или `bootJar` task в Gradle. Требует compile перед собой.

**Assemble** (Gradle-специфичный термин) — build без tests. `gradle assemble` компилирует, собирает JAR, но не запускает тесты. Полезно, когда тесты гоняются в отдельном CI stage.

**Build** — полный жизненный цикл: resolve dependencies → compile → test → package → static analysis → publish (optionally). `gradle build` включает всё. В Maven аналогично — `mvn package` до фазы package, `mvn install` до install и т.д.

Для Spring Boot проектов в enterprise правильная последовательность в CI обычно такая: `gradle assemble` (для быстрой сборки jar в отдельном job) → `gradle test` (параллельно unit тесты) → `gradle integrationTest` (параллельно integration через Testcontainers) → `docker build` (в отдельном job после успеха всех тестов).

## Что такое deploy детально

Deploy — процесс размещения и активации артефакта в целевом окружении. Разбивается на три подзадачи.

**Provisioning** — подготовить где будет работать. Bare metal — установить OS, настроить сеть. VM — создать виртуалку через Terraform. Контейнер — запустить контейнер на хост-машине. K8s — Pod создаётся автоматически через Deployment controller.

**Distribution** — доставить артефакт. Bare metal — `scp jar user@host:/opt/app/`. Контейнер — `docker pull registry/image`. K8s — kubelet вызывает CRI, containerd pull'ит image из registry.

**Activation** — запустить, направить трафик, обновить DNS/load balancer. Bare metal — `systemctl start app`. K8s — Pod становится Ready, EndpointSlice controller добавляет его в endpoints, kube-proxy обновляет iptables, трафик пошёл.

## Уровни абстракции deployment

За последние 30 лет каждый следующий уровень абстракции убирает часть операционной работы. Понимать историю полезно, потому что все уровни ещё живы в проде.

**Bare metal** (1990s) — физический сервер в стойке. Deploy: `scp jar на server, systemctl start`. Всё вручную. Каждый сервер уникален (снежинка), состояние накапливается. Плюс — полный контроль, нет overhead виртуализации. Минус — provisioning долгий (заказ железа — недели), scaling тяжёлый. Сейчас — только для performance-critical (HFT, gaming, embedded), плюс legacy.

**VM (2000s)** — виртуализация. Один физический сервер держит десятки VM. Provisioning быстрее (минуты). Terraform + Ansible / Packer автоматизирует. Immutable VM подход: Packer собирает AMI (Amazon Machine Image) → new VMs from image → drop старые. Auto-scaling groups (AWS ASG, Azure VMSS) — управляемое масштабирование. Ещё много enterprise на VM — Java-приложение работает так же, как в контейнере.

**Container (2013+)** — Docker и Linux namespaces + cgroups. Легче VM (нет виртуализации ядра), быстрый старт (секунды vs минуты), reproducible (image = состояние). Docker daemon на VM + `docker-compose` — простой setup для мелких проектов, не production-grade для scale.

**Kubernetes (2020s стандарт)** — orchestration для контейнеров. Одна платформа для сотен сервисов. Self-healing (Pod упал → рестарт), service discovery, rolling updates встроены, autoscaling. Complexity: cluster нужно поддерживать. Managed K8s (EKS, GKE, AKS) убирает control plane заботы.

**Serverless / FaaS** — AWS Lambda, Cloud Functions. Deploy = загрузить код (zip). Провайдер сам управляет запуском. Pay-per-invocation. Cold start проблема для Java исторически (2-10 секунд), лучше с AWS SnapStart (2022, 100-500 ms). Подходит для event-driven, не для long-running.

**Managed platforms** — Heroku, Cloud Run, Vercel. Deploy = `git push`. Никаких манифестов, часто dockerfile тоже не нужен (Buildpacks делают image из source). Скорость DX за счёт vendor lock-in.

Каждый уровень — компромисс между контролем и удобством. В КНП стек — Kubernetes через Helm/Helmsman + ArgoCD, приложения в Docker images в приватном Nexus. Стандартный enterprise pattern.

## Immutable vs mutable deployment

Один из самых важных архитектурных выборов.

**Mutable** — «обновление на месте». SSH на сервер → скопировал новый JAR → перезапустил systemd. Стейт сервера меняется, накапливается drift. Разные серверы могут быть в немного разных состояниях. Работает пока работает. Плохо становится когда: три сервера якобы одинаковые, но у одного install'ена пакета X, у другого поменяны настройки Y, у третьего был applied hotfix Z — и уже никто не помнит, что и когда. Инциденты на такой инфре — кошмар для диагностики.

**Immutable** — «пересборка вместо обновления». Собрал новую версию image → развернул новую VM/Pod из этого image → убил старую. Никаких SSH, никаких updates in place. Стейт сервера = image. Одинаковые images → идентичное поведение. Инфраструктурная метафора: cattle vs pets. Сервера — крупный рогатый скот, не домашние животные. Заболел — заменяешь, не лечишь.

K8s — immutable by design. Нельзя «редактировать Pod», можно только удалить и создать новый. Deployment изменил spec — старые Pod'ы удаляются, новые создаются. Ни один Pod не существует «в изменённом состоянии».

Плюс immutable — воспроизводимость, простота debugging (все Pods одинаковые), простота rollback (просто переключить обратно).

## Что происходит при K8s deploy (детально)

Уже разобрано в 82-м файле про полный pipeline, здесь коротко ключевые звенья.

```
kubectl apply -f deployment.yaml
     ↓
apiserver: auth + admission webhooks + etcd write
     ↓
Deployment controller видит event → создаёт новый ReplicaSet
     ↓
ReplicaSet controller создаёт Pod'ы (с nodeName="")
     ↓
Scheduler назначает nodeName по filter + score
     ↓
Kubelet на ноде через CRI → containerd → runc → clone/unshare/pivot_root
     ↓
Container стартует, приложение поднимается
     ↓
Readiness probe passes → Pod готов
     ↓
EndpointSlice controller добавляет Pod IP в endpoints
     ↓
Kube-proxy обновляет iptables на всех нодах
     ↓
Трафик идёт в новый Pod
```

Каждое звено — отдельная система, отдельный класс failure. Знать полный путь необходимо для диагностики любого deploy-инцидента.

## Три оси классификации деплоя

Стратегии деплоя часто мешают в одну кучу, но они лежат в разных плоскостях. Разделим по трём осям.

**Ось 1: как заменяем старую версию новой** — техническая механика.

- **In-place** — обновление старой версии в том же процессе/контейнере. Практически мёртв в контейнерной эре.
- **Recreate** — снести всё старое → поднять всё новое. Downtime.
- **Rolling** — постепенно (по одному Pod'у) заменяем. Стандарт.
- **Blue-Green** — параллельно две среды, атомарный switch трафика.
- **Canary** — постепенно перенаправляем трафик 5% → 20% → 50% → 100%.

**Ось 2: кто видит новую версию** — управление аудиторией.

- **Big Bang** — все пользователи разом.
- **Ring / Wave** — сначала внутренние, потом beta, потом всех.
- **A/B Testing** — часть на v1, часть на v2 по criteria.
- **Feature flags** — фича в проде, но за флагом.
- **Dark Launch** — код в проде, но невидим пользователю.

**Ось 3: куда идёт реальный трафик** — тип нагрузки.

- **Live traffic** — реальные пользователи попадают в новую версию.
- **Shadow / Mirror** — реальный трафик копируется в новую версию, ответ не показывается клиенту.
- **Synthetic** — только искусственные тесты.

Реальный deploy — комбинация этих осей. «Canary + feature flag + shadow copy для 10% пользователей» — обычная стратегия в крупных системах.

## Rolling Update — стандарт для 99% случаев

Начнём с самого распространённого. Rolling — постепенная замена: сначала новый Pod поднимается, потом старый убивается, и так по одному, пока все не заменятся. Всегда есть работающие Pod'ы, никакого downtime.

Схема для replicas=4, maxSurge=1, maxUnavailable=0:

```
Step 0:  [v1][v1][v1][v1]                       4 v1 ready
Step 1:  [v1][v1][v1][v1][v2 starting]          4 v1 + 1 v2 starting
Step 2:  [v1][v1][v1][v1][v2]                   4 v1 + 1 v2 ready
Step 3:  [v1][v1][v1]    [v2]                   3 v1 + 1 v2 (killed old)
Step 4:  [v1][v1][v1][v2 starting][v2]          3 v1 + 1 v2 + 1 v2 starting
...
Step N:                [v2][v2][v2][v2]         4 v2 ready
```

K8s Deployment по умолчанию использует Rolling. Настройка:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

`maxSurge` — сколько можно поднять сверху нормы (`replicas + maxSurge` — максимум Pod'ов в моменте). `maxUnavailable` — сколько может быть недоступно (`replicas - maxUnavailable` — минимум Ready). Оба могут быть числом или процентом (`25%` = ceil(0.25 × replicas)). Оба нулём — deploy не сдвинется.

Для zero-downtime в проде:

```yaml
rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0
```

Всегда есть минимум `replicas` работающих Pod'ов. Никогда не убиваем без нового Ready.

Дополнительные параметры:

- `progressDeadlineSeconds` (default 600) — если новый ReplicaSet не стабилизировался за это время, Deployment помечается `Progressing=False`. Автоматического rollback **нет** — только помечается. Rollback делается вручную.
- `minReadySeconds` — сколько секунд Pod должен быть Ready перед тем, как считаться «пройдённым». Полезно, если приложение стартует, но нестабильно первые секунды.
- `revisionHistoryLimit` — сколько старых ReplicaSets хранить для rollback (default 10).

Плюсы rolling. Zero downtime при правильной настройке. Постепенная обкатка новой версии — если что-то ломается на первых Pod'ах, rollout stuck, старая версия ещё работает. Встроено в K8s, ничего дополнительного. Экономно по ресурсам (максимум `replicas + maxSurge`).

Минусы. Обе версии работают одновременно — обязательна backward compatibility (API, БД схема, message contracts). Долго — 20 replicas × 30-секундный startup = 10+ минут. Rollback тоже rolling (медленный).

Rollback: `kubectl rollout undo deployment/myapp` — переключается на предыдущий ReplicaSet, тоже rolling.

Основная сложность rolling — миграции БД. Разберём отдельно ниже.

## Recreate — big bang с downtime

Recreate — простейшая стратегия. Все старые Pod'ы удаляются одновременно, потом создаются новые.

```
Time →
  Old: [v1][v1][v1][v1]
                     ↓
                    [_][_][_][_]  ← downtime
                                  ↑
                                  [v2][v2][v2][v2]
```

K8s config:

```yaml
strategy:
  type: Recreate
```

Downtime гарантирован — минимум 30-60 секунд (пока новые Pod'ы Spring Boot стартуют). Клиенты получают 5xx или connection refused в это окно.

Когда использовать. Dev / staging — норм. Batch-приложения без пользователей. Приложения с эксклюзивным ресурсом (только один Pod может держать distributed lock). Внутренние админ-панели с plan window. **Никогда** — user-facing prod без planned downtime.

Плюсы: простой (одно состояние в моменте, не думать о совместимости версий), освобождает ресурсы (не нужны maxSurge мощности), проще для несовместимых миграций БД (нет момента когда обе версии работают).

## Blue-Green — мгновенный rollback

Blue-Green — принципиально другая механика. Две параллельные среды. Blue — текущий prod. Green — новая версия. Оба полностью развёрнуты. Свитч трафика — атомарный, за секунду.

```
    Load Balancer / Service
             │
        ┌────┴────┐
        │         │
       BLUE     GREEN
       v1.5     v1.6
       4 pods   4 pods
       (active) (idle/testing)

── deploy step ──►

        ┌────┴────┐
        │         │
       BLUE     GREEN
       v1.5     v1.6
       4 pods   4 pods
       (idle/    (active)
       rollback)
```

Реализация в K8s через label selector Service:

```yaml
# Two Deployments
apiVersion: apps/v1
kind: Deployment
metadata: {name: myapp-blue}
spec:
  replicas: 4
  selector: {matchLabels: {app: myapp, version: blue}}
  template:
    metadata: {labels: {app: myapp, version: blue}}
    spec: {containers: [{name: app, image: myapp:1.5}]}
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: myapp-green}
spec:
  replicas: 4
  selector: {matchLabels: {app: myapp, version: green}}
  template:
    metadata: {labels: {app: myapp, version: green}}
    spec: {containers: [{name: app, image: myapp:1.6}]}
---
# One Service, selects by version
apiVersion: v1
kind: Service
metadata: {name: myapp}
spec:
  selector:
    app: myapp
    version: blue     # switch to green when ready
  ports: [{port: 80, targetPort: 8080}]
```

Deploy шаги:

1. `myapp-blue` работает на v1.5, Service указывает на blue.
2. Создаём `myapp-green` с v1.6, 4 replicas. Стоит рядом, трафик не идёт (Service не селектит).
3. **Smoke testing на green** — через direct Pod IP или отдельный dev-Service.
4. Атомарный switch: `kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'`.
5. Через 1-5 секунд kube-proxy обновляет iptables на всех нодах → весь трафик идёт в green.
6. Blue остаётся 15-60 минут как rollback target. Если проблема → флип обратно за секунду.
7. Убираем blue (или оставляем как idle для следующего цикла).

Плюсы. Мгновенный rollback — секунда, не десятки минут rolling. Полный тест новой версии перед экспозицией (пока green idle, гоняем на нём smoke tests). Нет «обе версии одновременно» — либо всё blue, либо всё green. Проще для несовместимых изменений (одномоментный switch).

Минусы. 2× ресурсов во время deploy (два полных стека). In-flight соединения при switch: активные HTTP keep-alive или long-polling могут остаться на blue — нужно правильное draining, plus preStop hook. БД проблема остаётся — обе версии подключены к той же БД, миграция должна быть compatible. Сложнее реализовать (два Deployment'а, кастомный workflow, ручной switch).

Когда использовать. Критичные системы, где 5xx на 30 секунд неприемлемы. Когда нужен мгновенный rollback (финтех, health, ecommerce peak). Когда есть бюджет на 2× ресурсов.

**Argo Rollouts** превращает blue-green в one-CRD experience:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    blueGreen:
      activeService: myapp
      previewService: myapp-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 3600
```

`activeService` — Service, куда идёт живой трафик. `previewService` — отдельный Service для preview (тестирование). Argo сам создаёт два ReplicaSets, обновляет selectors при promotion. `autoPromotionEnabled: false` требует ручного подтверждения. `scaleDownDelaySeconds: 3600` держит старое ReplicaSet час после switch — окно для rollback.

## Canary — минимальный blast radius

Canary — маленькая часть трафика идёт на новую версию. Название — от канарейки в шахте: шахтёры брали птицу под землю, если газ утёк, птица умирала первой, шахтёры узнавали. Аналогия: если новая версия сломана, страдают 5% пользователей, не 100%.

```
              Load Balancer
                    │
        95% ────────┴──────── 5%
         ↓                     ↓
       BLUE                   CANARY
       v1.5                   v1.6
       19 pods                1 pod

Наблюдение метрик (5-30 минут)
         ↓
Всё хорошо?
   ↓                    ↓
   yes                  no
   ↓                    ↓
  25/75, 50/50,      rollback
  75/25, 100/0       canary → 0
```

Реализация без service mesh — через ratio replicas:

Один Service селектит `app: myapp` (без version). Два Deployment'а:

- `myapp-stable` — 19 replicas, image v1.5.
- `myapp-canary` — 1 replica, image v1.6.

Один общий label `app: myapp` → Service селектит все 20 Pod'ов. Kube-proxy балансирует случайно → 1 из 20 = 5% трафика на canary.

Минус подхода — гранулярность = 1/replicas. Хочешь 1% canary при 20 replicas — не выйдет, нужно 100 replicas. Плюс трафик не reliable 5% — statistical (может быть 3% или 8% в реальности).

С service mesh (Istio) — точный процент:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata: {name: myapp}
spec:
  http:
  - route:
    - destination: {host: myapp, subset: stable}
      weight: 95
    - destination: {host: myapp, subset: canary}
      weight: 5
```

Можно роутить по headers, cookies, geolocation:

```yaml
http:
- match:
  - headers:
      x-user-tier: {exact: beta}
  route:
  - destination: {host: myapp, subset: canary}
- route:
  - destination: {host: myapp, subset: stable}
    weight: 100
```

Beta-users всегда получают canary, остальные — stable. Комбо canary + A/B по criteria.

**Argo Rollouts canary** — production стандарт:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause: {duration: 5m}
      - analysis:
          templates: [{templateName: success-rate}]
      - setWeight: 20
      - pause: {duration: 10m}
      - analysis:
          templates: [{templateName: success-rate}]
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
```

AnalysisTemplate опрашивает Prometheus:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: {name: success-rate}
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.99
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status!~"5.."}[2m]))
          /
          sum(rate(http_requests_total[2m]))
```

На каждой паузе Argo опрашивает Prometheus. Success rate ≥ 99% → следующий weight. Меньше → **автоматический rollback**. Это уже progressive delivery — автоматизация canary через метрики.

Плюсы canary. Реальный трафик — не искусственные тесты. Минимальный blast radius — 5% пользователей vs 100%. Автоматическая остановка при деградации. Идеально для рискованных изменений и ML-моделей.

Минусы. Долго — 30 минут до часов. Не подходит для срочных фиксов. Обе версии работают одновременно (compatibility требуется). Нужна observability (без метрик canary — просто rolling с задержкой). Сложность (Istio setup, Argo Rollouts, analysis templates).

Когда использовать. Крупные критичные системы (миллионы пользователей). Экспериментальные фичи. ML-моделей deploy — валидированные на subset. Финансовые transactions.

## A/B Testing — про бизнес-эксперименты, не про деплой

Часто путают с canary. A/B — не deployment strategy в чистом виде. Это technique для продуктовых экспериментов.

Отличия:

- **Canary** — временный, цель — safe rollout. Все в итоге получат новую версию.
- **A/B** — постоянный (недели-месяцы), цель — измерить какая версия лучше по бизнес-метрикам (conversion rate, time-on-site, revenue per user). Может кончиться выбором варианта или разделением по сегментам.

Роутинг обычно по criteria — user ID (hash → bucket A или B), geolocation, device (mobile vs desktop), random assignment сохранённый в cookie.

Типичная реализация — через feature flag сервис (LaunchDarkly, Unleash) внутри приложения:

```java
if (unleash.isEnabled("new-checkout", context)) {
    return newCheckoutFlow();
} else {
    return oldCheckoutFlow();
}
```

Один Deployment, один Pod, вся логика в приложении. Метрики (conversion) считаются по группам A/B.

Или через Istio routing по header'у — как canary, но с постоянным разделением.

Плюсы. Data-driven решения о продукте. Реальные пользователи, реальные метрики.

Минусы. Долгий цикл (2-4 недели минимум для статистической значимости). Сложность анализа. Загрязняет код (обе ветки логики живут). Требует инструмента (feature flag service + analytics).

## Shadow / Traffic Mirroring — реальная нагрузка, никакого риска

Shadow — реальный live трафик копируется в новую версию, но её ответ **не возвращается клиенту**. Клиент получает ответ старой версии, а новая работает на тех же запросах для наблюдения.

```
  User request
       │
       ▼
   Load Balancer / Mesh
       │
   ┌───┴────────┐
   │            │
   ▼            ▼ (mirror copy)
  BLUE        SHADOW
  v1.5 ──►    v1.6
   │          │
   │          └─── response DISCARDED
   ▼
  User (получил ответ от blue)
```

Реализация в Istio:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
spec:
  http:
  - route:
    - destination: {host: myapp, subset: stable}
      weight: 100
    mirror:
      host: myapp
      subset: shadow
    mirrorPercentage:
      value: 50.0
```

50% всего трафика дублируется в shadow. Пользователи видят только stable, но shadow обрабатывает половину нагрузки.

Плюсы. Zero risk для пользователей — они не видят shadow ответы. Реальная нагрузка на новую версию — проверка производительности. Сравнение ответов: логируешь оба, сравниваешь offline. Идеально для рефакторинга — убедиться, что новая имплементация даёт те же ответы, что старая.

Минусы. **Side effects проблема**. Shadow-запросы делают реальные writes → двойная запись в БД. Нужно либо:

- Shadow работает read-only (специальный mode в коде).
- Отдельная БД для shadow (репликация от stable).
- Feature flag внутри приложения: «если shadow request — skip DB write».

Overhead — двойная нагрузка на систему. Не подходит для API с side effects (payments, order creation) без изоляции.

Когда использовать. Read-heavy сервисы (search, recommendations). Рефакторинг существующего сервиса без изменения API. Load testing на реальном трафике перед canary. ML models — сравнить предсказания старой и новой модели на одних и тех же данных.

## Dark Launch — код в проде, но невидим

Dark launch — код **уже в проде**, но **невидим пользователю**. Активация — через feature flag, без нового деплоя.

Отличие от feature flag в целом: dark launch — специально про то, что вся инфраструктура готова, приложение развёрнуто, но UI/фича не показана пользователю до конкретного момента.

```
Неделя 1: deploy v1.6 с новой фичей за флагом OFF
         (flag=false → dead code, но всё работает)
         ── PROD OK, никто не видит фичу ──►

Неделя 2: включаем флаг для internal users (test accounts)
         ── команда проверяет ──►

Неделя 3: включаем для 1% реальных пользователей
         ── мониторим ──►

Неделя 4: 100%
```

Реализация:

```java
@Value("${feature.new-payment.enabled:false}")
private boolean newPaymentEnabled;

@Value("${feature.new-payment.beta-users:}")
private Set<String> betaUsers;

@PostMapping("/pay")
public PaymentResult pay(PaymentRequest req, @AuthPrincipal User user) {
    boolean useNew = newPaymentEnabled || betaUsers.contains(user.getId());
    if (useNew) {
        return newPaymentGateway.charge(req);
    }
    return oldPaymentGateway.charge(req);
}
```

Флаг из ConfigMap / Consul / feature flag сервиса. Изменение флага — без нового деплоя.

Плюсы. Разделение deploy и release (12-factor III). Deploy в пятницу можно (код там, но неактивен), release в понедельник. Instant rollback (флаг OFF, не redeploy). Скрытая доработка — код в проде тестируется тихо.

Минусы. Dead code накапливается, если флаг не убирают после rollout. Тесты усложняются (матрица флагов). Логика if/else в коде.

Anti-patterns: флаги никогда не выключаются, накапливается 200 «временных» флагов. Каждая фича за флагом (перебор). Правило — каждый флаг имеет expiration date. Через 3 месяца после enable для всех — код старой ветки удаляется.

## Ring / Wave — постепенное расширение аудитории

Несколько «колец» пользователей, deploy идёт постепенно от кольца к кольцу.

Классический пример — Microsoft Windows Insider программа:

```
Ring 0: internal Microsoft (десятки инженеров)      ← первая неделя
Ring 1: Fast Ring (миллион insiders)                ← вторая неделя
Ring 2: Slow Ring (10 миллионов beta users)         ← через месяц
Ring 3: Release Preview                             ← через 2 месяца
Ring 4: General Availability                        ← через 3 месяца
```

Каждое кольцо — время наблюдения. Если проблема на ring 1, не докатывается до ring 2.

По сути canary с явными группами вместо процентов. Через feature flag сервис + user attributes:

- Employee → Ring 0.
- Beta signup → Ring 1.
- Country: US → Ring 2 (сначала америка).
- Rest → Ring 3.

Плюсы. Каждое кольцо — контролируемое окно feedback'а. Разные SLA (внутренние толерантнее к багам). Регуляторная compliance (некоторые регионы обязаны получить позже).

Минусы. Долго (месяцы). Обе версии работают месяцами — сложно поддерживать compatibility. Управление сложное.

Когда. Крупный ecosystem (Windows, Chrome, iOS, крупные SaaS). Не для внутренних сервисов.

## БД миграции при deploy — самая сложная часть

Все стратегии, где обе версии работают одновременно (rolling, canary, blue-green с overlap), сталкиваются с одной проблемой: **обе версии обращаются к той же БД**. Схема должна поддерживать обе.

Плохой сценарий (одним релизом):

- v1 читает `user.name`.
- Release: `RENAME COLUMN name TO full_name`.
- v2 читает `user.full_name`.
- Rolling: v1 (ещё живой в rolling) → читает `name` → column не существует → 500.

Правильно — **expand-contract pattern**. Двухфазный (или трёхфазный) релиз при любом breaking изменении.

**Release 1 (expand)** — добавляем новое, сохраняем старое:

```sql
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
UPDATE users SET full_name = name WHERE full_name IS NULL;

CREATE OR REPLACE FUNCTION sync_name_columns()
RETURNS trigger AS $$
BEGIN
    IF NEW.name IS DISTINCT FROM OLD.name THEN
        NEW.full_name := NEW.name;
    ELSIF NEW.full_name IS DISTINCT FROM OLD.full_name THEN
        NEW.name := NEW.full_name;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sync_name_full_name 
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION sync_name_columns();
```

Теперь БД имеет обе колонки, они синхронизируются через триггер. v1 читает `name`, v2 (пока не задеплоена) читала бы `full_name`. Обе работают.

**Release 2 (transition)** — deploy v2:

- v2 пишет и читает `full_name`.
- v1 всё ещё пишет `name` (триггер синхронизирует).
- Обе версии совместимы.

Rolling deploy v1 → v2 работает нормально.

**Release 3 (contract, через 1-4 недели)** — только v2 в проде уже, удаляем старое:

```sql
DROP TRIGGER sync_name_full_name ON users;
DROP FUNCTION sync_name_columns();
ALTER TABLE users DROP COLUMN name;
```

Три релиза, недели времени. Но zero downtime и safe rollback возможен на каждом этапе.

Правила safe migrations в PostgreSQL:

**Всегда безопасно**:

- `ADD COLUMN` с nullable или `DEFAULT` (PG 11+ хранит default в метаданных).
- `ADD TABLE`.
- `ADD INDEX CONCURRENTLY` (берёт SHARE UPDATE EXCLUSIVE, не блокирует DML).
- `ADD CHECK CONSTRAINT NOT VALID` + отдельный `VALIDATE CONSTRAINT`.

**Опасно**:

- `ADD COLUMN NOT NULL DEFAULT volatile_expr` — full table rewrite.
- `DROP COLUMN` — если код ещё использует.
- `RENAME COLUMN / TABLE` — атомарно, но код не готов сразу.
- `ALTER COLUMN TYPE` — часто lock + rewrite.
- `ADD INDEX` без `CONCURRENTLY` — эксклюзивный lock всей таблицы.

Migration как отдельный этап в K8s через ArgoCD PreSync hook:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-2026-09-17
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: liquibase
        image: myapp-liquibase:1.0.42
        command: [liquibase, --url=..., update]
```

ArgoCD Sync последовательность: PreSync jobs → Sync (Deployment/Service) → PostSync. Миграция запускается перед tokens deployment, приложение стартует уже на новой схеме.

Rollback после несовместимой миграции — тяжёлая ситуация. Deploy v2 → migration → в проде проблема → нужно на v1.

Варианты:

1. **Быстрая обратная миграция** — reverse SQL, если возможно. Не всегда (DROP COLUMN с потерей данных — не обратить).
2. **Hot fix** (roll-forward) — v2.1 с исправлением, катим вперёд. Быстрее, чем rollback + reverse migration.
3. **PIT restore из backup** — потеря данных за время между backup и restore. Последнее средство.

Правило: не допускать breaking migration в одном релизе. Всегда expand-contract.

## Deployment environments

Каноническая пирамида:

```
        ┌───────────┐
        │   PROD    │  real users
        ├───────────┤
        │ PRE-PROD  │  final validation
        │(staging)  │  clone of prod config
        ├───────────┤
        │    QA     │  QA team тесты
        ├───────────┤
        │    DEV    │  developer sandbox
        └───────────┘
```

Продвижение (promotion) артефакта: dev → QA → staging → prod. Тот же image идёт через все стадии, конфиг меняется.

В КНП структура:

- `dev` — разработчики.
- `test` — QA.
- `release` (staging) — препрод, зеркало prod конфига.
- `prod` — production.

Namespaces: `knp`, `fno`, `fo`, `tax-report` (плюс `21` варианты для Java 21 версий сервисов). Каждая среда — отдельный K8s namespace или отдельный cluster.

Environments as code. Все окружения описаны в git repo. Пример Kustomize:

```
manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── configmap-patch.yaml
    ├── staging/
    │   └── ...
    └── prod/
        ├── kustomization.yaml
        ├── configmap-patch.yaml
        └── deployment-patch.yaml   # больше replicas, resources
```

`kubectl apply -k overlays/prod` — базовые манифесты плюс prod-специфичные patches. Аналог в Helm — один chart + разные `values.yaml` per env: `helm install -f values-prod.yaml myapp ./chart`.

**Ephemeral environments** — для каждого PR свой временный env. `PR-1234` → namespace `pr-1234` с полным стеком. Плюс — изолированное тестирование, реалистичный integration. Минус — ресурсы, БД / stateful нужно clone'ить.

Автоматизация: ArgoCD ApplicationSet per branch, или CI создаёт namespace при PR open, удаляет при close. Полезная практика для teams с высокой velocity.

## Artifact registries

Артефакты по слою:

- **Source**: git repo, commit SHA.
- **Compiled**: `.class` файлы, native binary.
- **Package (Java)**: JAR, WAR, EAR — в Maven Central, Nexus, Artifactory, GitHub Packages, GitLab Packages.
- **Container (OCI)**: image в Docker Hub, Harbor, AWS ECR, GCR, Azure ACR, quay.io.
- **Chart (Helm)**: OCI registry (Helm 3.8+ стандарт), Artifact Hub, ChartMuseum (устаревающий).
- **Deploy manifest**: raw YAML или Kustomize base+overlay в git repo.
- **Release bundle**: image + config + manifests, tagged with version.

В КНП стек — Nexus для Java + Docker images (self-hosted).

Image tagging strategies:

- **`:latest`** — never в prod. HTML `<blink>` инфраструктуры. Каждый Pod может подтянуть разное.
- **Semantic version** (`:1.0.42`, `:v2.3.1`) — human-readable, но иммутабельность зависит от дисциплины (кто-то может перепушить).
- **Git commit SHA** (`:abc123def`) — immutable by definition, идеально для CI. Проблема: не читаемо.
- **Комбинация** (`:1.0.42-abc123def`) — best of both worlds.
- **Digest** (`@sha256:abc...`) — content-addressable, гарантированно immutable. Prod best practice.

Реальный prod manifest указывает digest, не tag:

```yaml
image: registry.example.com/myapp@sha256:abcd1234...
```

Digest даёт абсолютную гарантию содержимого. Tag может быть переуказан (злонамеренно или по ошибке), digest — нет.

## Rollback strategies

Типы rollback:

**`kubectl rollout undo`** — быстро, но не через git. Если работаете в GitOps через ArgoCD, ArgoCD увидит diff между git и cluster и вернёт вперёд:

```bash
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=3
```

Используется в аварийных случаях, когда нет времени идти через git flow.

**Git revert (GitOps way)** — правильный подход:

```bash
git revert HEAD
git push
# ArgoCD синхронизирует → rollback
```

Rollback как обычный commit в manifest repo. Auditable, reproducible.

**Redeploy previous version tag**:

```bash
kustomize edit set image myapp=myapp:1.0.41  # предыдущий tag
git commit -am "Rollback to 1.0.41"
git push
```

Явный откат через изменение image tag. Часто предпочитают revert, потому что revert восстанавливает всю конфигурацию (не только image).

**Blue-Green flip** — секунды:

```bash
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

Мгновенно. Только если у вас blue-green setup.

**Feature flag off** — rollback фичи, не артефакта:

```yaml
# ConfigMap: feature.new-payment.enabled=false
```

Не нужен redeploy, флаг перечитывается через `@RefreshScope` / Consul / feature flag сервис. Часто быстрее и безопаснее, чем rollback артефакта.

Roll-back vs Roll-forward — стратегический выбор.

**Roll-back** — вернуться к предыдущей версии. Используется, когда:

- Не знаешь причину, нужно время на диагностику.
- Есть stable known good state.
- Проблема серьёзная (finance impact, data corruption).

**Roll-forward** — исправить в новой версии, deploy v2.1 быстро. Используется, когда:

- Проблема мелкая, знаешь фикс.
- БД мигрирована несовместимо (rollback невозможен без потерь).
- v1 тоже имеет проблему (просто другую).

Первое, что делают senior teams при инциденте — **rollback first, debug later**. Меньше stress, больше времени. Диагностика — потом, когда prod уже стабилен.

Проблема необратимого deploy. Некоторые изменения не откатываются:

- Sent emails (нельзя «отменить» email).
- Migration которая `DROP COLUMN` с потерей данных.
- External API calls (записали в чужую систему).
- Log messages, audit trails.

Отсюда правило: **опасные изменения — за feature flag**, дай возможность instant rollback без redeploy.

## Versioning — semver и альтернативы

Semantic Versioning: `MAJOR.MINOR.PATCH` (например `1.4.2`).

- **MAJOR** — breaking changes (API убрал/изменил). Клиенты должны адаптироваться.
- **MINOR** — новые фичи, backward compatible.
- **PATCH** — bug fixes only.

Extensions: `1.4.2-rc.1` (release candidate), `1.4.2-alpha.3` (alpha build), `1.4.2+build.abc123` (build metadata).

Semver — для libraries и API contracts. Для internal сервисов часто overkill.

**CalVer (Calendar Versioning)** — `YYYY.MM.PATCH`: `2026.01.5` — январь 2026, пятый хотфикс.

Плюсы: сразу видно возраст. Не надо думать «minor или major». Минусы: не показывает breaking changes. Использует Ubuntu (`24.04`), JetBrains IDE.

**KNP style** — ticket-based:

- `release-ISNA2-23651-...` — ветка от задачи.
- `ISNA2-23651: описание` — commit сообщение.
- Deployment описывается как «релиз ветки ISNA2-XXXXX».

Semver номер обычно не используется — фиксируется тикет. Работает потому что release-ветки строго линейны, история tickets — источник истины.

## Deployment tooling landscape

CI (Continuous Integration) — собирает и тестирует:

- **Jenkins** — Java, самый старый, plugin ecosystem огромный. Гибкий, но complex.
- **GitLab CI** — интегрирован с GitLab. `.gitlab-ci.yml`. Стандарт для GitLab пользователей.
- **GitHub Actions** — интегрирован с GitHub. YAML workflows. Огромный marketplace actions.
- **CircleCI**, **Travis CI** — SaaS исторические.
- **Buildkite** — hybrid (agents on your infra, control на SaaS).
- **Tekton** — K8s-native CI на CRD.

CD (Continuous Delivery/Deployment):

- **Continuous Delivery** — готово к deploy в любой момент, но кто-то жмёт кнопку.
- **Continuous Deployment** — автоматически в prod при merge в main.

Инструменты:

- **Argo CD** — GitOps для K8s. Watches git repo, syncs kubectl apply. Стандарт.
- **Flux** — тоже GitOps, без UI. Weave Works.
- **Spinnaker** — Netflix, мощный, для мульти-cloud, blue-green из коробки. Сложный.
- **Argo Rollouts** — расширение Argo для progressive delivery (canary, blue-green с metrics).
- **Octopus Deploy** — коммерческий, для .NET экосистемы часто.
- **Harness** — коммерческий, ML-based deploys.

Package managers для K8s:

- **Helm** — templating + release management. Chart = template + values. `helm install` создаёт release. Стандарт де-факто.
- **Kustomize** — overlay-based, no templates (patches). Проще Helm, но менее мощно. Встроен в kubectl.
- **jsonnet / Tanka** — программируемая конфигурация (Grafana).
- **Pulumi**, **CDK8s** — configuration в TypeScript/Python/Go (не YAML).

Helm vs Kustomize — вечный спор. Kustomize если нужен просто patch (overrides для env). Helm если нужны loops, conditionals, computed values, versioned releases.

**Helmsman** (используется в КНП) поверх Helm — declarative describe of releases. Один файл описывает все установленные chart'ы:

```yaml
apps:
  isnaknpuser:
    namespace: knp
    chart: nexus/isnaknpuser
    version: 1.0.42
    valuesFile: ./values/prod/isnaknpuser.yaml
```

`helmsman apply` — синхронизирует с реальным кластером. Плюс: declarative над imperative `helm install`. Минус: устаревающий (сейчас чаще GitOps через ArgoCD).

## Zero-downtime deploy: полный чек-лист для Spring Boot

Собираем всё вместе. Реальный prod deployment Spring Boot приложения в K8s с zero downtime требует настроек на нескольких уровнях.

Spring Boot side:

```yaml
server:
  shutdown: graceful           # ждём in-flight requests
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
management:
  endpoint.health.probes.enabled: true
  endpoints.web.exposure.include: health,info,prometheus
```

K8s Deployment side:

```yaml
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0        # никогда не убить без нового Ready
  template:
    spec:
      terminationGracePeriodSeconds: 45   # > Spring timeout + preStop
      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 10"]  # окно для iptables
        readinessProbe:
          httpGet: {path: /actuator/health/readiness, port: 8080}
        livenessProbe:
          httpGet: {path: /actuator/health/liveness, port: 8080}
        startupProbe:
          httpGet: {path: /actuator/health/liveness, port: 8080}
          failureThreshold: 30
          periodSeconds: 10
```

Плюс PodDisruptionBudget:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp
spec:
  minAvailable: 3
  selector:
    matchLabels: {app: myapp}
```

Что происходит при deploy:

1. New Pod поднимается, startup passes, readiness Ready → трафик идёт.
2. Old Pod → Terminating → удалён из EndpointSlice → kube-proxy update (1-5 сек).
3. preStop `sleep 10` — ждём распространения iptables на всех нодах.
4. `SIGTERM` → Boot прекращает принимать новые запросы, ждёт in-flight до 25 секунд.
5. Boot exits → container done.

Все in-flight запросы завершены, никакие клиенты не видят 5xx.

Клиенты (consumers) должны иметь retry на 5xx / connection reset для idempotent операций — на всякий случай.

## Заключение

Build/Release/Deploy — три отдельных концепции, смешивание которых даёт половину проблем. Build — код в артефакт. Immutable, тот же commit → тот же artifact. Release — build + конфиг для окружения. Deploy — активация. Правило: один build, много releases для разных сред.

Виды деплоя лежат в трёх плоскостях. Как заменяем (in-place / recreate / rolling / blue-green / canary). Кто видит (big bang / ring / A/B / feature flag / dark launch). Куда идёт трафик (live / shadow / synthetic). Реальные стратегии комбинируют оси.

**Rolling** — стандарт 99% случаев. Zero downtime, но требует backward compatibility.

**Recreate** — только dev/staging или batch-приложения. Downtime гарантирован.

**Blue-Green** — мгновенный rollback ценой 2× ресурсов. Для критичных систем.

**Canary** — минимальный blast radius для рискованных изменений. Progressive delivery через Argo Rollouts автоматизирует переходы по метрикам Prometheus.

**Shadow** — реальная нагрузка без риска. Для рефакторинга и load testing.

**Dark Launch + Feature flags** — deploy отделен от release. Код в проде, включается флагом. Instant rollback фичи без redeploy.

**БД миграции** — самая сложная часть при rolling/canary/blue-green. Expand-contract pattern обязателен: сначала add колонку, потом переключаем код, потом удаляем старое. Три релиза, недели времени, но zero downtime и safe rollback возможен.

**Rollback strategies**: git revert (правильно в GitOps), kubectl rollout undo (быстро), feature flag off (мгновенно, без redeploy), blue-green flip (секунды). Правило: rollback first, debug later.

Практический совет: возьмите один сервис в КНП, разберитесь, какая стратегия деплоя применяется у вас (обычно rolling через ArgoCD). Попробуйте локально в minikube настроить Argo Rollouts с canary strategy и AnalysisTemplate против фейкового Prometheus. Разверните blue-green через Argo Rollouts, попробуйте атомарный switch. Играйте с maxSurge/maxUnavailable, наблюдайте что происходит через `kubectl get replicasets -w`. Не читайте про это — попробуйте в песочнице. Тогда следующая production проблема с rolling stuck или несовместимой миграцией решится за минуты, а не часы.
