# 82. От `git push` до running Pod — полный процесс + Maven phases

## Зачем это знать

Каждый senior-инженер в микросервисном мире должен уметь провести цепочку от нажатия Enter после `git push` до момента, когда новый код принимает трафик — не пропуская ни одного звена. Не потому что это красиво выглядит на собесе, а потому что когда что-то ломается в проде — а ломается регулярно — нужно знать, где именно оно сломалось. Push прошёл, но pipeline не стартовал — где искать? Pipeline прошёл, но новый Pod не поднимается — что произошло? Pod поднялся, а трафик не идёт — какой контроллер ещё не сработал?

Каждая стрелка в цепочке — это отдельный протокол, отдельная система, отдельный класс проблем. Git wire protocol, server-side hooks, webhooks, CI runners, build tool lifecycles (Maven или Gradle), Docker layer caching, OCI registry, GitOps операторы, K8s controllers, kubelet, CRI, containerd, runc, CNI, iptables, EndpointSlices. Всё это работает, потому что каждое звено полагается на строго определённый интерфейс с соседним. И когда что-то падает — обычно виноват один слой, а не вся цепочка.

Есть ещё вторая причина. Многие оптимизации, о которых спорят на review'ах, имеют смысл только если понимаешь весь путь. Почему layered JAR быстрее? Потому что docker push шлёт только изменившиеся layers, и в layered JAR только application layer меняется на каждой сборке — dependencies остаются те же. Почему `maxSurge=1, maxUnavailable=0`? Потому что понимая как работает rolling update, ты знаешь: сначала поднимается новый Pod, потом убивается старый — и это возможно только если новых на 1 больше. Почему `preStop sleep 10`? Потому что между "Pod удалён из EndpointSlice" и "iptables обновились на всех нодах" есть окно 1-5 секунд, куда клиенты продолжают слать запросы на убитый Pod.

Мы разберём весь путь с адекватной глубиной. Git wire protocol и что реально летит по проводу. Server-side hooks и почему они запускают ваш CI. Как GitLab Runner подхватывает работу — long polling или webhook. Устройство pipeline через stages/jobs/artifacts/cache. Maven lifecycle — все 23 фазы, разница phase vs goal, плагиновые bindings. Gradle task graph — почему DAG, up-to-date checks, build cache, configuration cache. Docker build изнутри — layer caching, BuildKit, multi-stage, layered JAR специально для Spring Boot. Docker push через OCI Distribution Spec — content-addressable, digests, только delta летит. GitOps через ArgoCD и почему CI не должен `kubectl apply`. K8s Pod lifecycle — от apiserver до running контейнера, все контроллеры, scheduler, kubelet, CRI, containerd, runc. Readiness probes, EndpointSlices, kube-proxy iptables — как трафик находит новый Pod. Rolling update и graceful shutdown. И под конец — где что ломается в проде, как это диагностировать.

## Что такое `git push` — по проводам и по коду

`git push origin release-ISNA2-23651` — эта команда запускает довольно нетривиальный протокол. Git — это не «отправка файлов на сервер». Это синхронизация объектной базы между локальным и удалённым репозиторием, оптимизированная под то, чтобы передавать минимум байтов. Разберёмся, что реально происходит.

Первое: клиент читает `.git/config` и находит remote с именем `origin`. URL может быть в трёх форматах: `git@gitlab.1sc.kz:isna/knp.git` (SSH), `https://gitlab.1sc.kz/isna/knp.git` (HTTPS), или устаревший `git://` (без auth, не используется). SSH и HTTPS — два принципиально разных транспорта.

Для SSH клиент запускает `ssh git@gitlab.1sc.kz git-receive-pack 'isna/knp.git'` — это буквально запуск процесса на сервере через SSH. Аутентификация — по ключам (`~/.ssh/id_ed25519`). После handshake оба процесса (локальный git и удалённый git-receive-pack) общаются по stdin/stdout. TLS/SSL здесь не нужен — SSH сам шифрует.

Для HTTPS клиент делает HTTP запросы. Первый — `GET /isna/knp.git/info/refs?service=git-receive-pack` — это ref advertisement: сервер отдаёт список всех branch/tag ссылок с их SHA. Второй — `POST /isna/knp.git/git-receive-pack` — сюда клиент шлёт pack file и командует обновить refs. Аутентификация — Basic auth (username + PAT/пароль в заголовке `Authorization: Basic ...`), обычно через credential helper (Windows Credential Manager, macOS Keychain, `git-credential-store`).

Дальше — независимо от транспорта — идёт **wire protocol**. Актуальная версия — v2 (Git 2.18+), она умеет lazy fetch и другие оптимизации. Стороны обмениваются "capabilities" (что каждый умеет), потом идёт **ref discovery**: сервер шлёт список своих refs, клиент видит, что у него локально есть коммиты, которых нет на сервере, и решает, что нужно отправить.

Потом происходит самое интересное — **pack negotiation**. Git не отправляет отдельные файлы. Он собирает **pack file** — оптимизированный бинарный контейнер, содержащий commits, trees, blobs, которые нужны серверу. Клиент говорит: «я хочу обновить ref refs/heads/feature до SHA A. Я знаю, что у сервера есть коммит B (общий предок)». Сервер отвечает: «пришли мне всё между B и A, чего у меня нет». Клиент собирает пак — обычно delta-compressed: если blob X отличается от уже известного blob Y на 3 строки, пак содержит не полный X, а delta (Y → X). Плюс zlib-compression на всё. Итого: если ты изменил 3 строки в 1 файле, летит килобайты, а не мегабайты.

Наблюдать это можно через `GIT_TRACE_PACKET=1 GIT_TRACE=1 git push -v`. Увидишь, как открывается соединение, отправляется список refs, идут capability strings, потом raw pack data. Полезно понимать хотя бы раз в жизни — потом будешь понимать, почему `git push` иногда занимает секунды, а иногда минуты (зависит от размера дельты, не от количества коммитов).

После upload'а pack'а клиент отправляет **ref update command**: «на сервере: измени refs/heads/feature с SHA X на SHA Y». Сервер применяет команду атомарно (или отклоняет всё). Если ветка защищена или у клиента нет прав — reject.

## Server-side hooks — три штуки

Как только сервер получает push, он выполняет **три hook'а** в строгом порядке. Это ключевой extension point Git — здесь запускается всё остальное, включая ваш CI.

**pre-receive** — глобальный hook, выполняется один раз для всего push'а, независимо от количества обновляемых refs. Получает на stdin список обновлений в формате `<old-sha> <new-sha> <ref>`. Его задача — валидация: имеет ли пользователь право пушить в эти refs? Не защищена ли ветка (protected branches)? Не превышает ли push лимит по размеру? Валидны ли git-lfs pointers? Если pre-receive выходит с ненулевым exit code — весь push отклоняется, даже частично не применяется.

В GitLab pre-receive реализуется через Gitaly plugin systems. В GitHub Enterprise — через pre-receive hook scripts. В self-hosted Git — ручной shell script в `hooks/pre-receive`.

**update** — вызывается для каждого ref в отдельности. Получает `<ref> <old-sha> <new-sha>` через argv. Если update возвращает ненулевой exit для конкретного ref — только этот ref не применяется, остальные могут пройти. На практике используется редко (обычно всё делают в pre-receive).

**post-receive** — вызывается **после** того, как refs успешно обновлены. Именно этот hook запускает ваш CI. Получает список обновлений на stdin. Задачи: триггернуть pipeline, послать webhook'и наружу, обновить search index в GitLab, разослать email-уведомления, обновить мерж-реквесты (если push пришёл в feature-ветку с открытым MR — MR обновится с новым коммитом).

В GitLab post-receive делает намного больше, чем в чистом Git. Он читает `.gitlab-ci.yml` из вершины push'а, парсит его, определяет какие jobs должны запуститься для этой ветки/MR/тега (по правилам `rules`, `only`, `except`, `workflow`), создаёт запись `Pipeline` в БД с дочерними `Job` записями. Первые jobs (в первом stage) помечаются `pending` — они готовы к тому, чтобы runner их подхватил. Всё это происходит синхронно в рамках post-receive, поэтому если у GitLab-сервера БД тормозит, push тоже будет тормозить.

Аналогичный механизм в GitHub Actions — но с меньшим количеством ceremony. GitHub server парсит `.github/workflows/*.yml`, находит workflow с `on: push` (или другими триггерами), создаёт workflow run, отправляет job'ы в очередь для runners.

## Как CI-раннер подхватывает работу

Есть два принципиально разных механизма доставки job'ы до runner'а: long polling и webhook.

**Long polling** — используется GitLab Runner. Раннер сидит в бесконечном цикле и раз в несколько секунд шлёт HTTP GET к GitLab: «есть работа?». GitLab отвечает либо `204 No Content` (нет), либо `201 Created` с JSON job spec. Так работает и внутренний GitLab, и `gitlab.com`. Плюс: работает через любые NAT/firewall — раннер сам обращается наружу. Минус: небольшая latency (несколько секунд между появлением job и её подхватыванием).

**Webhook** — используется GitHub Actions self-hosted runners, ArgoCD, Jenkins. Здесь Git server сам вызывает HTTP endpoint на CI-системе с payload'ом события. Плюс: мгновенно. Минус: CI должен быть доступен снаружи (или через сложную VPN/tunnel setup).

Внутри enterprise стандартно используется long polling: runner'ы в закрытой сети, наружу выходят через прокси, наружу не пускают вообще. Runner подтягивает работу изнутри.

Раннер получает job spec — это JSON с полями `image` (какой Docker image использовать для выполнения), `script` (список shell-команд), `artifacts` (что сохранить), `cache` (что закэшировать), `services` (какие sidecar-контейнеры поднять рядом), `variables` (env-переменные), `before_script`, `after_script`. Плюс secrets — они не в JSON, отдельным запросом раннер их выкачивает по job token'у.

Executor определяет, где именно выполнять команды. Возможные варианты:

**shell** — прямо на хост-машине runner'а. Опасно (нет изоляции между jobs), быстро, легко. Используется для admin-задач и legacy.

**docker** — каждый job в отдельном контейнере. Runner monts репозиторий в контейнер, запускает контейнер с указанным image, execute script. После job — контейнер удаляется. Изоляция хорошая, но нужен доступ к Docker daemon на хосте.

**kubernetes** — самый популярный для внутренних CI. Каждый job — это отдельный Pod в K8s кластере. Pod содержит несколько контейнеров: main build container (с image из job spec), возможно service containers (Postgres для тестов, RabbitMQ), возможно helper containers (для клонирования репозитория, для загрузки артефактов). Runner общается с Pod'ом через K8s API, стримит логи, следит за exit codes. После job — Pod удаляется. Идеально для эффемерных workload'ов: подняли, отработали, убрали.

**docker-machine** — самый старый auto-scaling механизм. Каждый job поднимает VM в облаке через docker-machine, выполняет job, гасит VM. Тяжело, дорого, но полная изоляция.

Особый случай — **shared runner** vs **specific runner**. Shared runner доступен всем проектам в GitLab, specific — только конкретному проекту (или группе). В enterprise чаще shared, потому что удобно централизованно управлять. В открытом GitLab.com shared runners платные, поэтому проекты часто разворачивают свои.

## `.gitlab-ci.yml` — как это устроено

Файл в корне репозитория, YAML, читается GitLab при каждом push. Определяет structure pipeline. Возьмём реальный пример:

```yaml
stages:
  - validate
  - build
  - test
  - security
  - package
  - deploy-staging
  - integration-test
  - deploy-prod

variables:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dorg.gradle.parallel=true"
  DOCKER_TLS_CERTDIR: "/certs"

default:
  image: registry.1sc.kz/build-tools:jdk21-gradle8
  cache:
    key: "${CI_COMMIT_REF_SLUG}"
    paths:
      - .gradle/caches
      - .gradle/wrapper

validate:
  stage: validate
  script:
    - gradle checkstyleMain spotbugsMain --no-daemon

compile:
  stage: build
  script:
    - gradle assemble --no-daemon
  artifacts:
    paths:
      - build/libs/*.jar
    expire_in: 1 day

unit-test:
  stage: test
  script:
    - gradle test jacocoTestReport --no-daemon
  artifacts:
    reports:
      junit: build/test-results/test/*.xml

integration-test:
  stage: test
  services:
    - name: postgres:16
      alias: db
    - name: rabbitmq:3.13
      alias: rabbit
  variables:
    SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/test
    SPRING_RABBITMQ_HOST: rabbit
  script:
    - gradle integrationTest --no-daemon

build-image:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build --cache-from $CI_REGISTRY/isnaknpuser:latest -t $CI_REGISTRY/isnaknpuser:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY/isnaknpuser:$CI_COMMIT_SHORT_SHA

deploy-staging:
  stage: deploy-staging
  script:
    - git clone $MANIFESTS_REPO manifests
    - cd manifests/apps/isnaknpuser/staging
    - kustomize edit set image myapp=$CI_REGISTRY/isnaknpuser:$CI_COMMIT_SHORT_SHA
    - git commit -am "Staging: isnaknpuser $CI_COMMIT_SHORT_SHA"
    - git push
  only: [main, release]

deploy-prod:
  stage: deploy-prod
  when: manual
  script: ...
  only: [master]
```

Разберём ключевые концепции.

**Stages** — последовательные фазы. Job'ы одного stage выполняются параллельно (если у runner'а есть свободные слоты). Следующий stage начинается только после того, как **все** job'ы предыдущего успешно завершились (кроме тех, у кого `allow_failure: true` или `when: manual`). Это ключевой order-of-operations.

**Jobs** — единицы работы. У каждой обязательный `stage` и `script`. Один job = один shell (или несколько shell'ов), выполняющихся последовательно.

**Artifacts** — файлы, которые job создаёт и передаёт следующим stages. Загружаются в GitLab после успешного (или всегда, с `when: always`) завершения job. Скачиваются следующими job'ами перед их запуском. Пример: `compile` job создаёт `build/libs/app.jar`, `unit-test` его скачивает и использует. Артефакты immutable, versioned внутри pipeline. Обычно хранятся в S3 (или локальном GitLab object storage) с TTL (`expire_in`).

**Cache** — временное хранилище для ускорения. Не гарантирован (может быть invalid при инвалидации, при переезде на другой runner). Обычно `.gradle/caches`, `~/.m2/repository`, `node_modules`. Cache key определяет, когда cache reuse работает — обычно `${CI_COMMIT_REF_SLUG}` (кэш на branch) или `${CI_JOB_NAME}` (кэш на job).

Разница artifact vs cache важна: artifact — обещание («этот файл будет доступен в следующем stage»), cache — оптимизация («если повезёт, эти файлы уже будут»). Artifact всегда работает или job упадёт. Cache может миссануть и job просто скачает всё с нуля.

**Variables** — env vars. Могут быть в pipeline-level (`variables:` в корне), job-level, project-level (в GitLab UI Settings > CI/CD), group-level, instance-level. Secrets — тоже variables, но с флагом `masked: true` (в логах заменяются на `[MASKED]`) и `protected: true` (доступны только на protected branches).

**Services** — sidecar-контейнеры рядом с job. Классика — Postgres для интеграционных тестов. GitLab поднимает `postgres:16` рядом с job container, соединяет их сетью, service доступен по alias'у как hostname. Так интеграционные тесты гоняют реальную БД, но эфемерную (после job — контейнер убивается).

**Rules / only / except** — когда job запускается. `only: [main, release]` — только на этих ветках. `rules: - if: $CI_PIPELINE_SOURCE == "merge_request_event"` — только для MR. Правила `rules` умеют больше и постепенно вытесняют `only/except`.

**when: manual** — job требует ручного триггера через UI. Классика для `deploy-prod`.

## Maven lifecycle — 23 phases, phases vs goals, plugin bindings

Maven — самый распиаренный Java build tool. Спорят с Gradle бесконечно, но 40% enterprise проектов до сих пор на нём. Понимать Maven надо не поверхностно.

Maven строится вокруг понятия **lifecycle**. Их три:

**default lifecycle** — build цикл (validate, compile, test, package, install, deploy). 23 фазы. Это то, что используется в 99% случаев.

**clean lifecycle** — очистка. `mvn clean` = удалить `target/` директорию (default output dir Maven).

**site lifecycle** — генерация HTML-документации по проекту (`mvn site`). Мало кто пользуется, кроме open-source проектов, где site — часть релиз-процесса.

Три lifecycles независимы. `mvn clean install` — это два раздельных вызова: сначала весь `clean` lifecycle, потом `default` до фазы `install`.

23 фазы default lifecycle:

```
validate
initialize
generate-sources
process-sources
generate-resources
process-resources
compile
process-classes
generate-test-sources
process-test-sources
generate-test-resources
process-test-resources
test-compile
process-test-classes
test
prepare-package
package
pre-integration-test
integration-test
post-integration-test
verify
install
deploy
```

Ключевое правило Maven: **вызов phase X = выполнение всех предыдущих phases**. `mvn package` не запускает только package. Он запускает: validate → initialize → generate-sources → process-sources → generate-resources → process-resources → compile → process-classes → generate-test-sources → process-test-sources → generate-test-resources → process-test-resources → test-compile → process-test-classes → test → prepare-package → package. Всё в порядке.

Это отличает Maven от Gradle. В Gradle `gradle bootJar` запустит только те задачи, от которых зависит bootJar (что часто меньше). В Maven же нельзя «пропустить compile и сразу test» — не поддерживается.

Явное отделение: **phase — это точка**, а не действие. Сама по себе фаза не делает ничего. Реальную работу выполняют **plugin goals**. Goal — это конкретное действие плагина, например `compiler:compile` (goal `compile` из плагина `maven-compiler-plugin`) или `surefire:test` (goal `test` из `maven-surefire-plugin`).

Phases связаны с goals через **plugin bindings**. Стандартные bindings для JAR-проекта:

| Phase | Plugin Goal |
|-------|-------------|
| process-resources | `resources:resources` |
| compile | `compiler:compile` |
| process-test-resources | `resources:testResources` |
| test-compile | `compiler:testCompile` |
| test | `surefire:test` |
| package | `jar:jar` |
| install | `install:install` |
| deploy | `deploy:deploy` |

Bindings зависят от `<packaging>` в pom.xml. JAR (default) — как выше. WAR — `package` binding'ится к `war:war`. EAR — `ear:ear`. Ваш кастомный тип packaging может иметь свои bindings.

Плагины могут добавить свои bindings в phases. Spring Boot Maven plugin биндит goal `repackage` к phase `package` — так что после того как `jar:jar` соберёт обычный JAR, Spring Boot repackage'ит его в fat JAR, добавляя loader и dependencies.

Разница между `mvn compile` и `mvn compiler:compile`: первое запустит phase compile (со всеми предыдущими: validate → ... → compile). Второе запустит только goal `compile` из compiler plugin — без предыдущих phases. Ты пропустишь validate, initialize и т.д. Второй режим полезен для очень целевых действий: `mvn dependency:tree` — показать граф зависимостей без предварительной работы.

Практические команды и их следствия:

```bash
mvn clean                  # только clean lifecycle
mvn compile                # default до phase compile
mvn test                   # + test-compile + test
mvn package                # + package (jar в target/)
mvn verify                 # + integration-test + verify
mvn install                # + install в ~/.m2/repository (local)
mvn deploy                 # + upload в remote Nexus/Artifactory
mvn clean package          # два lifecycles: сначала clean, потом default до package
mvn -DskipTests package    # компиляция тестов есть, запуск — нет
mvn -Dmaven.test.skip=true # ни компиляции, ни запуска тестов
```

Разница `-DskipTests` vs `-Dmaven.test.skip` тонкая, но важная. `-DskipTests` пропускает **запуск** тестов, но `test-compile` (компиляция test classes) выполняется. Полезно, когда ты хочешь убедиться, что тесты хотя бы компилируются, но не гонять их. `-Dmaven.test.skip=true` пропускает и компиляцию, и запуск. Максимально быстро, но не гарантирует, что тесты вообще работают.

`install` vs `deploy` — классический вопрос собеса. `install` копирует artifact в **локальный** `~/.m2/repository/`. Твой jar доступен только тебе (или другим Maven-проектам на этой же машине). `deploy` заливает artifact в **remote repository** (Nexus, Artifactory, Maven Central). Доступно всей команде. В CI-процессе `install` почти никогда не нужен — CI не публикует в глобальный `.m2`, потому что раннер эфемерен. Используется либо `package` (собрать JAR, положить в artifacts) для приложений, либо `deploy` (публиковать в Nexus) для библиотек.

Мультимодульные проекты Maven строятся через **reactor**. Корневой pom.xml с `<modules>`:

```xml
<modules>
    <module>api</module>
    <module>domain</module>
    <module>web</module>
</modules>
```

`mvn install` в корне запускает reactor: он определяет граф зависимостей между модулями (по `<dependency>` внутри), топологически сортирует, выполняет lifecycle для каждого модуля в правильном порядке. Полезные флаги: `-pl web -am` (только web + модули, от которых он зависит), `-pl web -amd` (только web + модули, зависящие от него), `-rf domain` (resume from — начать с domain, если раньше упал), `-T 4C` (parallel: 4 threads на CPU core).

Профили — механизм условной конфигурации. Пример:

```xml
<profiles>
    <profile>
        <id>prod</id>
        <build>
            <finalName>myapp-prod</finalName>
        </build>
    </profile>
</profiles>
```

Активируется через `mvn -Pprod install` или через `<activation>` (по env var, OS, наличию файла, версии JDK). В enterprise профили используются для окружений (prod vs staging), для CI vs local, для разных JDK versions.

Effective POM — реальный pom, после того как inheritance от parent, super POM (встроенный в Maven), profiles и variable resolution применены. Смотреть: `mvn help:effective-pom`. Используется, когда «почему у меня версия плагина не такая, как в моём pom» — обычно ответ в parent или super POM.

Dependency resolution в Maven — **nearest-wins**. Если A→B→C:1.0 и A→D→C:2.0, оба на одинаковой глубине — берётся тот, что **первый по объявлению** в pom.xml. Force версию через `<dependencyManagement>`:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>...</groupId>
            <artifactId>...</artifactId>
            <version>2.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Внутри `<dependencies>` этот блок не добавляет зависимость — он только фиксирует версию. Реальную зависимость всё равно надо объявить, но уже без `<version>` — Maven возьмёт из management. Это полезно для BOM (Bill of Materials): Spring Boot Starter Parent — это огромный BOM, фиксирующий версии всех dependencies, чтобы у тебя было согласованное дерево.

Смотреть реальное дерево: `mvn dependency:tree` (или `-Dverbose` для скрытых версий). Использовать, когда конфликт версий: увидишь, кто какую версию тянет, и решишь через `<exclusions>` или `dependencyManagement`.

## Gradle — task graph, up-to-date, build cache

Gradle идеологически другой. Вместо жёсткого lifecycle Gradle работает через **directed acyclic graph** задач. Каждая задача имеет:

- `inputs` (файлы или properties, от которых зависит выполнение)
- `outputs` (файлы, которые задача создаёт)
- `dependsOn` (другие задачи, которые должны отработать до неё)

При запуске `gradle build`:

1. **Configuration phase** — Gradle читает все `build.gradle` (или `.kts`) файлы, конфигурирует все задачи. Всё описывается — но не выполняется. Задачи добавляются в task registry.

2. **Task graph resolution** — Gradle смотрит, что запросил пользователь (`build`), находит задачу с этим именем, рекурсивно собирает все `dependsOn` — получается граф.

3. **Up-to-date check** — для каждой задачи в графе Gradle проверяет: если inputs не изменились с прошлого запуска (по hash), и outputs существуют — задача **UP-TO-DATE**, пропускается. Реально запускаются только те задачи, чей input изменился.

4. **Execution** — задачи выполняются в топологическом порядке. Независимые могут параллельно, если `org.gradle.parallel=true`.

Стандартные задачи Java plugin:

```
compileJava           .java → .class (main)
processResources      copy src/main/resources
classes               composite: compileJava + processResources
compileTestJava       .java → .class (test)
processTestResources
testClasses           composite
jar                   pack classes + resources → JAR
test                  run unit tests
check                 test + верификация (checkstyle, spotbugs)
assemble              build all archives
build                 check + assemble  ← самая частая цель
```

Spring Boot Gradle plugin добавляет свои задачи:

```
bootJar               fat JAR со всеми dependencies и loader
bootRun               запуск Spring Boot локально
bootBuildImage        сборка OCI image через Cloud Native Buildpacks
```

Важно: `bootJar` override'ит обычный `jar`. По умолчанию активна только `bootJar` — regular `jar` disabled. Если нужны оба (например, для библиотек), включаются явно:

```gradle
jar {
    enabled = true
}
```

Up-to-date check — фундамент производительности Gradle. Пример: ты изменил один Java-файл. `gradle build`:

- `compileJava` — inputs изменились (1 файл) → задача выполняется, но **incremental**: перекомпилируется только этот файл и его зависимости (не всё).
- `processResources` — resources не менялись → **UP-TO-DATE**, skipped.
- `test` — если inputs (compiled classes) изменились и есть тесты, зависящие от изменённых классов → run relevant tests.
- `bootJar` — inputs изменились (classes и resources) → repack.
- `check` — composite, только заново выполнит те подзадачи, чьи inputs изменились.

Итог: перезапуск занимает секунды, а не минуты. Полный rebuild после `gradle clean` — минуты, но inкрементально — считанные секунды.

Реальная работа: `gradle build --info` покажет для каждой задачи, была ли она UP-TO-DATE, FROM-CACHE, EXECUTED, SKIPPED. Отладка производительности начинается с этого.

**Build cache** — следующий уровень оптимизации. Task output кэшируется по hash inputs. Есть local cache (`~/.gradle/caches/build-cache-1/`) и remote cache (HTTP).

Работает так: ты собираешь проект, `compileJava` производит classes. Gradle хэширует inputs (source files + compile options + JDK version + ...), сохраняет outputs (`.class` файлы) в cache под этим хэшем. Твой коллега пуллит те же commits, запускает `gradle build`. У него те же inputs — тот же hash — Gradle **не выполняет `compileJava`**, а достаёт classes из cache (local или remote). Задача помечается **FROM-CACHE**. Мгновенно.

Enable в `gradle.properties`:

```properties
org.gradle.caching=true
```

Remote build cache настраивается в `settings.gradle`:

```gradle
buildCache {
    remote(HttpBuildCache) {
        url = 'https://build-cache.example.com/'
        push = true
    }
}
```

В enterprise это огромный win. Merge коллеги смерджился, CI собрал всё — все inputs теперь в remote cache. Ты пуллишь, запускаешь `gradle build` локально — сразу FROM-CACHE для всего, что не менялось. Секунды вместо минут.

**Configuration cache** — Gradle 6.6+. Кэширует не только outputs, но и результат configuration phase. Первый запуск: полная configuration (парсинг build.gradle, evaluation, task registration) — секунды. Второй запуск: configuration cache reused → сразу execution — миллисекунды на старт. Enable: `--configuration-cache` или в `gradle.properties`. Не все плагины поддерживают — проверь compatibility.

**Parallel execution** — `org.gradle.parallel=true`. Независимые модули (в multi-module проекте) собираются параллельно. Внутри одного модуля параллелизма нет.

Что реально выполняется на `gradle build` в типичном Spring Boot проекте:

```
compileJava            (2-30s зависит от размера)
processResources       (мс)
classes                (composite)
compileTestJava        (5-20s)
processTestResources
testClasses
test                   (30s-5min зависит от количества тестов и Spring context startup)
bootJar                (2-10s)
jar                    (skipped, если bootJar active)
assemble               (composite)
checkstyleMain         (5-15s)
checkstyleTest
spotbugsMain           (10-60s — медленный)
spotbugsTest
jacocoTestReport       (5-10s)
check                  (composite)
build                  (composite → done)
```

Первая сборка большого проекта (500KLOC) — 5-15 минут. С caches — 30 секунд до 2 минут.

## Test — pyramid и что реально исполняется

Тесты в CI — это не «один блок». Это несколько уровней с разной семантикой, разной скоростью и разными задачами. Классическая **тестовая пирамида**:

```
    E2E Tests         ← Selenium, Cypress, playwright — медленные, мало
    Integration Tests ← Testcontainers, Spring Boot Test — средние, средне
    Unit Tests        ← JUnit, Mockito — быстрые, много
```

Соотношение объёмов — тысячи unit'ов, десятки integration, единицы e2e. Обратная пирамида (мало unit, много e2e) — «ice cream cone», антипаттерн: тесты медленные, flaky, дорого поддерживать.

**Unit tests** — самое базовое. Один Java класс — свой test класс. Все dependencies mock'ированы. Тестируется логика в изоляции.

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;
    @InjectMocks UserService service;

    @Test
    void findById_returnsUser() {
        when(repo.findById(1L)).thenReturn(Optional.of(new User(1L, "Berik")));
        User u = service.findById(1L);
        assertEquals("Berik", u.getName());
    }
}
```

Test runner (Surefire для Maven, `test` task для Gradle) собирает test classpath, запускает JUnit engine, каждый `@Test` метод — отдельный test. Отчёт пишется в `build/test-results/test/*.xml` (JUnit XML format). Обычно тысяча unit тестов гоняется секунды-минуты.

**Integration tests** — тестируют взаимодействие компонентов, обычно с реальной БД. Классика: **Testcontainers** — библиотека, которая через Docker API запускает контейнеры на время теста.

```java
@SpringBootTest
@Testcontainers
class UserRepositoryIT {
    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.password", pg::getPassword);
    }

    @Autowired UserRepository repo;

    @Test
    void save_returnsIdAssigned() {
        User u = repo.save(new User(null, "Berik"));
        assertNotNull(u.getId());
    }
}
```

Что происходит при запуске: Testcontainers смотрит `@Container` поля, обращается к Docker API (сокет `/var/run/docker.sock` на Linux или `\\.\pipe\docker_engine` на Windows), запускает `postgres:16` контейнер, ждёт readiness. Экспозит порт наружу с рандомным mapping'ом (5432 в контейнере → 32768 на хосте, например). Spring Boot стартует, `@DynamicPropertySource` подставляет реальный JDBC URL Postgres контейнера. Flyway/Liquibase мигрирует schema. Тест запускается, работает с реальной БД. После теста контейнер убивается.

Плюсы Testcontainers: реальная БД → реальные интеграции проверены (в отличие от H2, где SQL диалект отличается). Легко воспроизвести prod-версию БД (`postgres:16`, тот же, что в проде).

Минусы: медленнее (стартап контейнера — 3-10 секунд, миграции — 1-30 секунд). Требует Docker daemon.

**В CI** — либо runner уже с доступом к Docker (docker executor или host mount), либо **DinD** (Docker-in-Docker) — service container docker daemon рядом с job. GitLab classic:

```yaml
integration-test:
  image: eclipse-temurin:21-jdk
  services:
    - docker:24-dind
  variables:
    DOCKER_HOST: tcp://docker:2375
    TESTCONTAINERS_HOST_OVERRIDE: docker
  script:
    - ./gradlew integrationTest
```

DinD — отдельный контейнер, работающий как Docker daemon. Твой Testcontainers обращается к нему через `DOCKER_HOST`, поднимает Postgres через него. Работает, но с нюансами: сеть между твоим job container и Postgres container идёт через daemon. Часто DinD tricky в K8s — приходится настраивать security context, privileged mode.

**Spring Boot Test slices** — способ ускорить integration tests, не поднимая полный context. Каждая slice поднимает только часть Spring beans:

- `@WebMvcTest` — только контроллеры + MockMvc, без service layer.
- `@DataJpaTest` — только JPA (Repositories + EntityManager), с H2 in-memory по умолчанию.
- `@JsonTest` — только Jackson serialization.
- `@RestClientTest` — REST client + MockRestServer.

Плюс: старт 2-5 секунд вместо 20-30. Минус: не проверяет integration полностью (например, `@WebMvcTest` не поднимет ваш actual Security config, только моки).

**Contract tests** — отдельная категория, для микросервисной архитектуры. Pact или Spring Cloud Contract. Идея: provider публикует **contract** (описание своего API — какие запросы принимает, что возвращает). Consumer тестирует своё поведение против **mock provider'а, сгенерированного из contract'а**. Provider же тестирует, что реально соответствует своему contract'у.

Плюсы: если provider ломает contract, его CI это ловит **до deploy**. Consumer не узнаёт о breaking change в 3 часа ночи в проде.

Минусы: сложность setup'а, нужен shared broker (Pact Broker) для хранения contracts.

Что делает CI на этапе test:

1. `gradle test` — unit тесты.
2. Собирает JUnit XML в `build/test-results/test/*.xml`.
3. Загружает как artifact в GitLab.
4. GitLab парсит XML, показывает в UI: total / passed / failed / skipped, duration, failure messages, stack traces.
5. Если хоть один тест failed → job fails → pipeline stops (следующие stages не запускаются).
6. Если passed → следующий stage: `gradle integrationTest` (обычно отдельный task в отдельном source set).

Test report artifact — критично для productivity. Разработчик видит failed tests прямо в MR view, без залезания в CI runner logs. Failure message + stack trace позволяют понять причину за секунды.

## Docker build — layers, BuildKit, multi-stage

После test'ов идёт package. Для JVM-приложений это обычно Docker image. Разбираем детально, потому что здесь много производительности можно потерять или выиграть.

**Multi-stage Dockerfile** — паттерн, где один Dockerfile содержит несколько `FROM` секций. Каждая — отдельная stage. В финальный image попадает только последняя (или те, что явно `COPY --from=<stage>` цитируются). Промежуточные stage выбрасываются.

```dockerfile
# Stage 1: build
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /workspace
COPY build.gradle settings.gradle gradlew ./
COPY gradle gradle
COPY src src
RUN ./gradlew dependencies --no-daemon
RUN ./gradlew bootJar --no-daemon

# Stage 2: runtime
FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app
COPY --from=builder /workspace/build/libs/*.jar app.jar
RUN groupadd -r spring && useradd -r -g spring spring
USER spring
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Что тут важно. Builder stage использует `-jdk` image (~450MB) — там есть javac, gradle wrapper, всё для сборки. Runtime stage использует `-jre` (~180MB) — только runtime. Финальный image содержит только JRE + jar (~200MB), builder stage выброшен. Без multi-stage image был бы ~800MB (JDK + source + all cached deps + jar).

Дополнительный security-плюс: в prod image нет javac, gradle, curl, sh (в distroless вариантах). Даже если атакер сможет выполнить код в контейнере, у него нет инструментов для build phase. Меньше surface для эксплуатации.

**Layer caching** — фундамент Docker performance. Каждая инструкция (`FROM`, `RUN`, `COPY`, `ADD`, `ENV`, ...) — отдельный **layer**. Layer — это дифф файловой системы (создали новые файлы, изменили существующие, удалили). Layers хранятся как read-only. При старте контейнера они складываются в union filesystem (overlayfs), плюс сверху read-write layer для контейнера.

При build Docker вычисляет hash каждой инструкции по её входам:

- Для `FROM alpine:3.19` — hash по image reference.
- Для `RUN <command>` — hash по command string.
- Для `COPY src dest` — hash по содержимому файлов в src.

Если такой hash уже есть в cache (потому что раньше строили — этот layer сохранён) — Docker переиспользует. Инструкция не выполняется. Layer уже готов.

Ключевое правило оптимизации: **редко меняющееся раньше, часто меняющееся позже**. Каждая инструкция инвалидирует cache для всех **последующих**. Если ты изменишь `COPY . /app` (весь source) в начале Dockerfile — все инструкции после этого будут выполнены заново, даже если они делают ровно то же самое.

Плохо:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY . /app                    # весь source — часто меняется
RUN ./gradlew bootJar          # cache invalidated при любом коде change
```

Каждое изменение любого Java-файла заставляет пересобирать всё, включая скачивание зависимостей.

Хорошо (Spring Boot **layered JAR**):

Spring Boot 2.3+ умеет собирать JAR не как один монолитный файл, а разложенным на слои. `bootJar` с `layered` config (по дефолту включен в новых версиях):

```
BOOT-INF/
    dependencies/           ← стандартные dependencies (Spring, Jackson, ...)
    spring-boot-loader/     ← loader classes
    snapshot-dependencies/  ← SNAPSHOT deps (в внутренние библиотеки под разработкой)
    application/            ← твой код + твои resources
```

Разделение важно, потому что application layer меняется на каждой сборке (ты пишешь код), а dependencies меняются редко (Spring Boot версию обновляешь раз в месяц). Docker layers маппятся на эти слои:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
# Deps редко меняются — early layer, часто cache hit
COPY --from=builder /workspace/build/libs/dependencies/ ./
COPY --from=builder /workspace/build/libs/spring-boot-loader/ ./
COPY --from=builder /workspace/build/libs/snapshot-dependencies/ ./
# Твой код часто меняется — последний слой, единственный invalidated
COPY --from=builder /workspace/build/libs/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

Изменил Java-файл → повторный build reuses все слои кроме `application/`. При `docker push` в registry — уходит только application layer (10-20MB), не 200MB image.

**`.dockerignore`** — must-have. Файл в корне (аналог `.gitignore`), исключает пути из build context. Без него `docker build` отправит всё содержимое директории в daemon — включая `.git`, `node_modules`, `build/`, `target/`. Это медленно (сотни MB) и invalidates cache (`.git` меняется на каждый commit, даже если ты не менял код).

Типичный `.dockerignore` для Java-проекта:

```
.git
.gradle
build
target
*.md
.idea
.vscode
node_modules
```

**BuildKit** — новый движок сборки, default с Docker 23. Замена классическому `docker build`. Что даёт:

- **Parallel execution** независимых steps (например, если у тебя два `FROM` stages, они могут строиться параллельно).
- **Better caching** — inline cache metadata в image, cache-from remote registry.
- **Mount secrets** без включения в layer (нельзя было в classic — секрет всегда попадал в image через RUN with env).
- **Mount SSH keys** для git clone внутри build.
- **Cache mounts** — persistent между builds (например, Maven `~/.m2` можно mount'ить как cache).

Enable: `DOCKER_BUILDKIT=1 docker build ...` или установить как default через Docker daemon config.

Пример с secret mount:

```dockerfile
# syntax=docker/dockerfile:1.4
FROM alpine
RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
```

Ты передаёшь секрет через `docker build --secret id=mysecret,src=./secret.txt` — он mount'ится только на время `RUN`, не остаётся в image.

Cache mount для Maven:

```dockerfile
RUN --mount=type=cache,target=/root/.m2 mvn package
```

Между сборками `.m2` сохраняется — dependencies не переcкачиваются при каждом build.

## Docker push и OCI registry

После build у тебя image — набор layers + manifest. Push загружает их в registry (Nexus, Harbor, Docker Hub, GitLab Container Registry, AWS ECR, ...).

```bash
docker push registry.1sc.kz/isnaknpuser:1.0.42
```

Что реально происходит:

1. Docker CLI шлёт запрос daemon'у: «push этот image».
2. Daemon читает manifest image'а — там list layer digests (sha256).
3. Daemon **сравнивает layers с тем, что уже есть в registry** через API `HEAD /v2/<name>/blobs/<digest>`. Если layer уже есть (потому что pushed раньше или потому что базовый image `eclipse-temurin:21-jre` уже там) — не пересылать.
4. **Только отсутствующие layers** отправляются: `POST /v2/<name>/blobs/uploads/` → `PATCH ...` (чанки) → `PUT ...` (финализировать).
5. После всех layers — daemon шлёт **manifest** (JSON): `PUT /v2/<name>/manifests/<tag>`.
6. Registry сохраняет layers в blob storage (обычно S3), manifest в metadata store.
7. Обновляется tag (pointer на manifest by digest).

Ключевое: **content-addressable**. Каждый layer идентифицируется по sha256 своего содержимого. Если контент один и тот же — digest один и тот же — не хранится дважды. Layer из `eclipse-temurin:21-jre` шарится между всеми твоими image'ами, использующими эту базу.

Итого на push: base image (~180MB) уже был в registry, никогда не пересылается. Middle layers (spring-boot deps, ~50MB) не менялись — reused. Только application layer (~10MB) пересылается. Push занимает секунды даже на медленной сети.

**OCI Distribution Spec** — стандарт API для registries. Docker Registry v2 = OCI Distribution. Endpoints:

- `HEAD /v2/<name>/blobs/<digest>` — проверить, есть ли layer.
- `POST /v2/<name>/blobs/uploads/` — начать upload layer.
- `PATCH /v2/<name>/blobs/uploads/<uuid>` — стримить чанки.
- `PUT /v2/<name>/blobs/uploads/<uuid>?digest=<digest>` — финализировать.
- `GET /v2/<name>/blobs/<digest>` — pull layer.
- `PUT /v2/<name>/manifests/<tag>` — обновить tag (или создать).
- `GET /v2/<name>/manifests/<tag>` — pull manifest.

Всё content-addressable, всё immutable. Layer с digest X никогда не меняется. Tag — только pointer на manifest by digest, может меняться (можно переуказать `latest` на другой manifest). Но конкретный `sha256:abc...` — навсегда.

Про **image tagging strategy**. Три подхода:

**Semantic versioning** — `myapp:1.2.3`. Стандарт для библиотек. Для приложений — если у тебя есть релиз-цикл.

**Git SHA** — `myapp:abc12345`. Каждый commit — свой image tag. Immutable, reproducible: точно знаешь, что за код в контейнере. Стандарт для CI-driven deployments.

**Latest** — `myapp:latest`. Удобно для dev/testing, **никогда** для prod. Ты не знаешь, что реально запущено. Rollback невозможен без пересборки. K8s ImagePullPolicy по умолчанию `IfNotPresent` — если Pod рестартует, кеш `latest` может быть устаревшим.

В prod стандарт — Git SHA (или combo `1.2.3-abc12345`), с pin на конкретный digest в манифесте:

```yaml
image: registry.example.com/myapp@sha256:def456...
```

Digest даёт **абсолютную гарантию** содержимого. Tag может быть переуказан (злонамеренно или по ошибке), digest — нет.

## Deploy — GitOps и почему CI не должен `kubectl apply`

Теперь image в registry. Дальше — deploy в кластер. Здесь важное архитектурное решение: **GitOps** vs **push-based deploy**.

**Push-based (антипаттерн)**: CI job подключается к K8s кластеру и делает `kubectl apply`. Или использует `helm upgrade`. Простой и очевидный подход, но с несколькими проблемами:

- **CI runner имеет kubeconfig с admin правами**. Компрометация runner'а = компрометация кластера. Атакер может создать что угодно, удалить что угодно.
- **Drift**: то, что в кластере, может отличаться от того, что в git. Кто-то `kubectl edit deployment` руками, никакой git-истории.
- **Нет audit trail**: непонятно, кто когда deploy'ил. Есть только CI logs, но кто пушнул код — не всегда очевидно.
- **Rollback сложнее**: надо снова запускать CI на старый commit. А если код уже сломался?

**GitOps** — паттерн, где git repo с манифестами становится **единственным источником правды**. Отдельный оператор в кластере (ArgoCD или Flux) синхронизирует cluster state с git.

Реальная организация:

- **application-repo** — исходный код Java-приложения. Тут pipeline собирает jar, docker image, пушит image в registry.
- **manifests-repo** — отдельный git repo с K8s манифестами (Deployment, Service, ConfigMap, Secret, HPA, ...) для всех сервисов и всех окружений (staging, prod).

Flow deploy:

```
Developer в application-repo:
  git push feature branch
       ↓
CI собирает jar → docker build → docker push (registry.example.com/myapp:abc123)
       ↓
CI stage "deploy-staging":
  1. git clone manifests-repo
  2. cd manifests/apps/myapp/staging
  3. kustomize edit set image myapp=registry.example.com/myapp:abc123
  4. git commit -am "Staging: myapp abc123"
  5. git push
       ↓
manifests-repo обновлён.
       ↓
ArgoCD в кластере (раз в 3 минуты или по webhook):
  1. git pull manifests-repo
  2. Сравнивает: git state ↔ etcd state в кластере
  3. Видит diff: image tag changed
  4. kubectl apply -f deployment.yaml (сам, изнутри кластера)
       ↓
K8s начинает rolling update.
```

Только у **ArgoCD** есть kubectl права. CI runner имеет только git write права к manifests-repo. Если атакер получил CI runner — максимум может пушить манифесты в git. Manifests-repo может быть защищен MR reviews — тогда даже это не проходит без approval.

**Kustomize edit set image** — сохраняет override в `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
images:
  - name: myapp
    newName: registry.example.com/myapp
    newTag: abc123
```

`kubectl apply -k .` заставляет Kustomize:

1. Прочитать `base/deployment.yaml`, `base/service.yaml` (общая конфигурация для всех окружений).
2. Применить image override (заменить image в контейнере Deployment).
3. Применить env-specific patches (resources, replicas, env vars).
4. Собрать финальный manifest.
5. Отправить в apiserver.

**Helm** — альтернатива Kustomize. Templating engine + package manager. Deployments становятся Helm charts с values.yaml. GitOps с Helm: values.yaml в git, ArgoCD рендерит chart с этими values и apply'ит.

Kustomize vs Helm — вечный спор. Kustomize проще (нет templating), декларативнее (overlay patches). Helm мощнее (loops, conditionals, functions), но сложнее (Go templates). В enterprise часто используют оба: инфраструктурный слой (ingress, cert-manager, monitoring) через Helm, application deployments через Kustomize.

**ArgoCD Application**:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
spec:
  source:
    repoURL: https://gitlab.1sc.kz/infra/manifests.git
    targetRevision: HEAD
    path: apps/myapp/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: knp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

ArgoCD в reconcile loop каждые 3 минуты (или по webhook от GitLab):

1. `git fetch` из repoURL.
2. Compare desired (from git) vs actual (from cluster etcd).
3. Если drift — `kubectl apply` (собственный клиент, не CLI).

`prune: true` — если ресурс есть в кластере, но нет в git, удалить его. `selfHeal: true` — если кто-то руками поменял в кластере, ArgoCD возвращает git state.

Без `selfHeal` — только показывает "OutOfSync" в UI, ждёт ручного `Sync`. Иногда полезно (не хочешь автоматических apply в prod без человеческого подтверждения).

**Flux** — альтернатива ArgoCD. Другой оператор, тот же подход. Разница в acking UX (Flux более low-level, CLI-oriented; ArgoCD имеет rich web UI).

## K8s Pod lifecycle — от apiserver до running контейнера

Апifierver получил `kubectl apply` — что происходит дальше, детально, с реальными временами.

**T = 0**: apiserver получает apply.

```
kubectl apply -f deployment.yaml
     ↓
apiserver:
  - Аутентификация (TLS client cert / bearer token / OIDC): ~1ms
  - Авторизация (RBAC): ~1ms
  - Mutating admission webhooks (defaulting, sidecar injection типа Istio): ~5-50ms
  - Schema validation (соответствует ли OpenAPI schema Deployment): ~1ms
  - Validating admission webhooks (кастомные policies, PSP replacement): ~5-50ms
  - etcd write (2-node Raft commit из 3-node cluster): ~5-20ms
Total: ~10-100ms
     ↓
apiserver returns 200 to client с обновлённым object
```

Обновлённый Deployment имеет `resourceVersion++`. Это оптимистическая concurrency: если два клиента одновременно update'ят, второй получит `409 Conflict` (resourceVersion mismatch), должен pull+retry.

Важно: apiserver **не создаёт Pod'ы сам**. Он просто сохраняет Deployment спеку в etcd. Реальная работа делается контроллерами.

**T = 100ms**: Deployment controller видит event.

Controller-manager — отдельный процесс, содержит все встроенные controllers (Deployment, ReplicaSet, StatefulSet, Job, Node, ServiceAccount, ...). Каждый controller держит **watch** на своих ресурсах — long-poll HTTP к apiserver, который стримит events по мере их появления.

```
watch event: MODIFIED deployment/myapp
     ↓
Deployment controller reconcile loop:
  - Читает current spec (новый template)
  - Считает template hash (SHA-256 от pod spec)
  - Ищет существующие ReplicaSets: есть ли RS с этим hash?
    - Нет → создать новый ReplicaSet (RS-abc) с 0 replicas
    - Есть → использовать (кейс rollback: rollback возвращает к предыдущему template hash, соответствующий RS ещё есть)
  - Update Deployment.status: newReplicaSet = RS-abc
  - Scale RS-abc up (до 1 replica for maxSurge=1)
  - Оставляет старый RS с текущими replicas
```

После этого шага в etcd:

- Deployment (обновлённый спек)
- Старый ReplicaSet (например, RS-xyz, 4 replicas)
- Новый ReplicaSet (RS-abc, 1 replica desired)

**T = 200ms**: ReplicaSet controller создаёт Pod.

```
watch event: MODIFIED replicaset/RS-abc (replicas 0 → 1)
     ↓
ReplicaSet controller:
  - Считает currentPods (0), desired (1) → нужен 1 pod
  - Создаёт Pod object в apiserver с nodeName=""
```

Pod создан. Всё, что о нём известно — spec (containers, volumes, resources), labels, ownerReference на RS. `nodeName` пустой — ещё не назначен на ноду.

**T = 300ms**: Scheduler назначает ноду.

Scheduler — отдельный процесс. Watch'ит Pod'ы с `spec.nodeName == ""` (unscheduled).

```
watch event: ADDED pod/myapp-abc-xyz (unscheduled)
     ↓
Scheduler reconcile:
  - Filter phase: какие ноды feasible?
    - resource requests (CPU/memory доступны)
    - taints/tolerations (у ноды нет taint'а без соответствующей toleration)
    - node affinity (labels match)
    - pod affinity/anti-affinity (расположение относительно других Pod'ов)
    - PVC (нода имеет доступ к нужному storage class)
    - nodeSelector, hostname, ...
  - Score phase: 0-100 для каждой feasible node
    - LeastRequestedPriority (менее загруженная = выше score)
    - BalancedResourceAllocation (balance CPU/mem usage)
    - NodeAffinityPriority (soft affinity)
    - InterPodAffinity, TaintTolerationPriority, ...
  - Выбирает node с максимальным score → worker-3
  - Bind: PATCH pod с spec.nodeName=worker-3 (через apiserver)
```

Pod теперь имеет `nodeName`. Scheduler не создаёт контейнер — он только назначает Pod на ноду.

**T = 400ms**: Kubelet на worker-3 видит Pod.

На каждой ноде запущен **kubelet** — агент, отвечающий за Pod'ы этой ноды. Kubelet держит watch на apiserver, фильтруя по `spec.nodeName == self`.

```
watch event: MODIFIED pod/myapp-abc-xyz (nodeName=worker-3)
     ↓
Kubelet syncLoop (запускается для этого Pod'а):
```

Дальше — многошаговый процесс.

**Image pull**. Kubelet проверяет: есть ли image локально в container image store? Если да и `imagePullPolicy: IfNotPresent` (default) — использует локальный. Если нет или `imagePullPolicy: Always` — pull.

CRI call `PullImage`:

- containerd (реализация CRI) обращается к registry.
- HTTP GET `/v2/<name>/manifests/<tag>` — получает manifest (список layer digests).
- Для каждого layer: `HEAD /v2/<name>/blobs/<digest>` — проверяет наличие локально. Если нет — `GET`.
- Скачивает и распаковывает (обычно gzip).
- Собирает в container image (union of layers).

Время:

- Image в cache: 0-500ms (просто проверка manifest).
- Cold pull: 5-60 секунд (зависит от размера image, скорости сети, скорости registry).

**Pause container**. Первым делом kubelet создаёт **pause container** — крохотный контейнер, единственная задача которого — держать открытыми Linux namespaces (network, PID, IPC) для всей Pod. Все реальные контейнеры Pod'а разделяют namespaces pause контейнера.

CRI call `RunPodSandbox`:

- containerd создаёт pause container с настройками из PodSpec (hostname, DNS config, etc).
- Настраивает network через **CNI plugin**.

**CNI plugin** — интерфейс к network. Реализации: Calico, Flannel, Cilium, weave. Что делает:

1. Создаёт `veth` pair (virtual ethernet): один конец в новом network namespace (Pod'а), другой — в host namespace.
2. Присваивает IP из pool этой ноды (например 10.244.1.5).
3. Устанавливает routes: внутри Pod'а — default через veth к host bridge. На host'е — bridge подключен к overlay network (VXLAN, BGP) для роутинга к другим нодам.
4. Настраивает iptables/nftables для NAT (если нужно) и network policies.

После CNI Pod имеет IP, доступный со всех нод кластера.

**Volumes**. Kubelet mount'ит volumes, объявленные в Pod spec:

- **ConfigMap**: kubelet читает ConfigMap object через apiserver, создаёт файлы в emptyDir на ноде, mount'ит в контейнер.
- **Secret**: то же, но в tmpfs (in-memory), чтобы secret не оседал на диске.
- **PVC**: kubelet вызывает CSI driver для attach (если PV network-attached типа AWS EBS — это может занять 30 секунд до 2 минут) и mount.
- **ServiceAccount token**: projected volume, kubelet генерирует JWT token и кладёт в файл.
- **DownwardAPI**: kubelet кладёт Pod metadata в файлы (например, `podIP`, `podName`).

**Container create**. Для каждого контейнера в Pod:

CRI call `CreateContainer`:

- containerd создаёт config.json (OCI runtime spec) с path'ами rootfs, command, args, env, mounts, resources.
- Вызывает **runc** (низкоуровневый runtime): `runc create`.
- runc делает Linux syscalls:
  - `clone(CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWUTS | ...)` — создаёт новые namespaces.
  - `mount` — bind mounts, tmpfs.
  - `pivot_root` — меняет root filesystem на rootfs image'а.
  - `setresuid/setresgid` — понижает privileges (если non-root user).
  - Устанавливает cgroups (CPU/memory limits).
- Контейнер создан, но не запущен.

**Container start**. CRI call `StartContainer` → containerd → `runc start`. runc execute main process (entrypoint) внутри созданного namespace. Первый процесс контейнера — PID 1 в его PID namespace.

Kubelet update'ит Pod status: `containerStatuses[0].state.running`, `startTime`.

От момента, когда kubelet увидел Pod, до момента, когда контейнер стартовал:

- Image в cache + fast volumes: 1-3 секунды.
- Cold image + network PVC: 30-90 секунд.

**T = ~5-30s**: Spring Boot стартует.

JVM запускается за секунду. Дальше Spring:

1. Class loading — Spring сканирует classpath, находит все `@Component`, `@Service`, `@Configuration` классы: 5-15 секунд.
2. Bean discovery — Spring resolve'ит dependency graph, определяет порядок создания beans: 3-10 секунд.
3. Autoconfig — Spring Boot запускает `@ConditionalOnClass`, `@ConditionalOnProperty` для тысяч возможных auto-configurations, включает те, что подходят: 5-15 секунд.
4. Database connect + migrations — HikariCP создаёт connection pool, Flyway/Liquibase проверяет migrations: 2-10 секунд.
5. Embedded Tomcat / Netty стартует, слушает на порту 8080: 1 секунда.

Total cold start: **20-45 секунд** для типичного Spring Boot приложения. Native image (GraalVM) — секунды. Standard JVM — десятки секунд.

Startup probe часто настроен на это окно:

```yaml
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
  # 30 × 10 = 5 минут макс на старт
```

Пока startup probe не passed, kubelet не начинает liveness/readiness checks. Это защита медленно стартующих приложений от преждевременного kill'а.

**T = ~30s**: Readiness passes.

Readiness probe — kubelet опрашивает `/actuator/health/readiness`:

```
GET http://<pod-ip>:8080/actuator/health/readiness
Response: 200 {"status": "UP"}
     ↓
Kubelet update'ит Pod status: conditions[Ready] = True
     ↓
Watch event: MODIFIED pod (Ready=True)
```

Разница liveness vs readiness:

- **Liveness** — «приложение живо». Fail → kubelet рестартует контейнер. Проверяет только критические внутренние состояния (deadlock detection).
- **Readiness** — «готово принимать трафик». Fail → Pod исключается из EndpointSlice, но не рестартуется. Может fail из-за отсутствия БД, warming caches, etc.

Правило: readiness может быть stricter (проверять зависимости), liveness — только critical failure (обычно /actuator/health/liveness с минимумом проверок).

**T = ~30.1s**: EndpointSlice controller добавляет Pod.

EndpointSlice controller — часть controller-manager. Watch'ит Services и Pods. Когда Pod становится Ready и его labels match Service selector — Pod добавляется в EndpointSlice.

```
EndpointSlice controller видит: Pod myapp-abc-xyz Ready, labels app=myapp совпадают с Service myapp selector
     ↓
Update EndpointSlice endpointslices/myapp-abc:
  add {addresses: [10.244.1.5], targetRef: pod/myapp-abc-xyz, conditions: {ready: true}}
     ↓
etcd write
```

EndpointSlice — новая API (K8s 1.17+, GA в 1.21), заменяет старый Endpoints. Разница: EndpointSlice поддерживает до тысяч endpoints через шардирование (несколько EndpointSlice objects на один Service), лучше scales.

**T = ~30.2s**: kube-proxy обновляет iptables на всех нодах.

**kube-proxy** — DaemonSet, запущен на каждой ноде. Держит watch на EndpointSlices. Задача — обеспечить, чтобы клиенты, обращающиеся к ClusterIP Service, попадали на реальные Pod'ы.

```
watch event: MODIFIED endpointslice/myapp-abc
     ↓
kube-proxy sync loop:
  - Rebuild iptables rules (или ipvs, если kube-proxy в ipvs mode)
  - iptables-restore --wait (atomic swap)
```

Правила выглядят примерно так:

```
-A KUBE-SERVICES -d 10.96.5.10/32 -p tcp -m tcp --dport 80 -j KUBE-SVC-XYZ
-A KUBE-SVC-XYZ -m statistic --mode random --probability 0.25 -j KUBE-SEP-P1
-A KUBE-SVC-XYZ -m statistic --mode random --probability 0.33 -j KUBE-SEP-P2
-A KUBE-SVC-XYZ -m statistic --mode random --probability 0.50 -j KUBE-SEP-P3
-A KUBE-SVC-XYZ -j KUBE-SEP-P4
-A KUBE-SEP-P1 -p tcp -m tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-P2 -p tcp -m tcp -j DNAT --to-destination 10.244.2.7:8080
-A KUBE-SEP-P3 -p tcp -m tcp -j DNAT --to-destination 10.244.3.9:8080
-A KUBE-SEP-P4 -p tcp -m tcp -j DNAT --to-destination 10.244.1.5:8080
```

Через statistic module — случайный (approximately even) выбор одного из Pod'ов. DNAT переписывает destination address с ClusterIP на реальный Pod IP.

Sync loop kube-proxy периодический (по умолчанию раз в секунду) плюс triggered on watch event. Между «Pod Ready» и «iptables updated на этой ноде» — 100ms-1s. Между «на этой ноде» и «на всех нодах кластера» — может быть 1-5s (все kube-proxy получают event независимо).

С этого момента трафик от клиентов внутри кластера может достигать нового Pod'а.

**IPVS mode**. Альтернатива iptables. Использует Linux IPVS (IP Virtual Server) — более эффективная L4-балансировка. Быстрее при тысячах Services. Enable в kube-proxy config. Работает так же, но использует IPVS rules вместо iptables.

**T = ~34s**: Deployment controller scale down старого.

Deployment controller видит: новый ReplicaSet (RS-abc) имеет 1 Ready Pod → можно scale down старый на 1 (если `maxUnavailable=0` — только тогда, когда добавили нового).

```
Scale RS-xyz с 4 до 3
     ↓
ReplicaSet controller выбирает Pod для удаления
```

Выбор — по deletion cost (кастомизируемый label), потом по AGE (старший умирает первым).

```
     ↓
Delete pod/myapp-xyz-abc (soft delete: deletionTimestamp set)
     ↓
Kubelet на worker-2 (где старый Pod) видит event:
  1. Marks Pod as Terminating
  2. Отправляет event удаления в EndpointSlice controller
     - EndpointSlice controller REMOVES Pod IP из endpoints
     - kube-proxy на всех нодах update'ит iptables — Pod больше не участвует в балансировке
  3. Kubelet execute preStop hook (если настроен)
     - Например `exec: [sleep, 10]` — sleep 10 секунд
     - В это время новые запросы не приходят (Pod вне endpoints)
     - Но in-flight запросы продолжают обрабатываться
  4. Kubelet sends SIGTERM to main container process
  5. Приложение graceful shutdown:
     - Stops accepting new connections (в Spring Boot: server.shutdown=graceful)
     - Waits for in-flight requests to complete (spring.lifecycle.timeout-per-shutdown-phase)
     - Closes datasource connections gracefully
     - Publishes deregistration event (если service discovery — Consul, Eureka)
     - Process exits 0
  6. Kubelet sees exit → CRI StopContainer, RemovePodSandbox
     - runc kills остатки процессов в контейнере
     - Container теardown
     - Network cleanup (CNI DEL)
     - Volumes unmount
  7. Pod object removed from apiserver (после `terminationGracePeriodSeconds`)
```

**Критичное окно**: между шагом 2 (endpoint removed) и tim `iptables updated everywhere` (обычно 1-5s) — клиенты продолжают слать запросы на этот Pod, потому что их kube-proxy ещё не обновился. Если Pod уже начал shutdown (Spring отвергает соединения) — клиенты видят connection refused.

**Fix**: `preStop sleep 10`. Даёт окно, где Pod ещё не начал shutdown (продолжает обрабатывать), но уже удалён из endpoints (новых запросов не должно быть). За 10 секунд iptables everywhere пропагируется. После preStop kubelet шлёт SIGTERM, начинается настоящий shutdown, in-flight уже мало.

**T = 34s → 60s**: continuation rolling.

Deployment controller продолжает:

- Scale RS-abc (новый) с 1 до 2 → new pod created → wait until Ready → scale RS-xyz с 3 до 2.
- И так далее до полного replacement всех replicas.

Общее время rolling update для 4 replicas × 30 sec startup = ~2-3 минуты для sequential rollout (по одному Pod'у). Можно parallel через `maxSurge: 2` (одновременно 2 новых), но 2× resources требуется в момент rollout.

**T = ~3 min**: rolling complete.

```
kubectl rollout status deployment/myapp
> deployment "myapp" successfully rolled out
```

Все Pod'ы новой версии, старые убраны.

## Timeline полный

Реальный пример для типичного микросервиса в КНП:

| Time | Event |
|------|-------|
| 0:00 | `git push origin release-ISNA2-XXXXX` |
| 0:01 | GitLab receives push, post-receive hook fires |
| 0:02 | Pipeline created, first jobs pending |
| 0:03 | GitLab Runner picks validate job |
| 0:15 | validate done (checkstyle, spotbugs) |
| 0:16 | Stage `build` starts |
| 0:45 | Gradle compile done, jar in artifacts |
| 0:46 | Stage `test` starts, unit + integration в parallel |
| 2:30 | Unit tests done |
| 3:45 | Integration tests done (Testcontainers Postgres) |
| 3:46 | Stage `security` (Trivy, OWASP) |
| 4:30 | Security done |
| 4:31 | Stage `package` — docker build |
| 4:35 | Docker build done (layer cache hit) |
| 4:40 | Trivy image scan done |
| 4:50 | Docker push to Nexus (только application layer) |
| 4:51 | Stage `deploy-staging` |
| 4:53 | Manifest repo updated, git push |
| 4:53 | ArgoCD webhook triggered |
| 4:54 | ArgoCD syncs staging |
| 4:54 | apiserver receives apply, Deployment updated |
| 4:54 | Deployment controller creates new ReplicaSet |
| 4:54 | ReplicaSet controller creates Pod |
| 4:54 | Scheduler assigns node worker-3 |
| 4:55 | Kubelet pulls image (cache hit, ~1s) |
| 4:56 | Container created, CNI network up |
| 4:56 | Container starts, JVM launch |
| 5:20 | Spring Boot fully started, readiness Ready |
| 5:20 | EndpointSlice updated |
| 5:21 | kube-proxy on all nodes updated |
| 5:22 | Old Pod terminated (rolling continues) |
| 5:45 | 2nd new Pod ready |
| 6:10 | 3rd |
| 6:35 | 4th, rolling complete |
| 6:36 | Smoke test job runs |
| 6:45 | Smoke tests pass, staging deploy успешен |
| Deploy-prod: manual button |

**End-to-end**: ~7 минут для staging автоматически. Prod — плюс ручной approve (несколько минут-часов зависит от процесса).

## Где что ломается — типичные failures в проде

Каждое звено может сломаться. Знать типичные failures — экономия часов при инцидентах.

**Git push failures**:

- `Authentication failed`: SSH key не зарегистрирован в GitLab. Или в HTTPS не тот PAT.
- `remote: push declined by pre-receive hook`: protected branch, нет permissions. Смотреть logs GitLab.
- `pack exceeds maximum allowed size`: слишком большой push. Обычно из-за случайно закоммиченного binary (dump БД, лог). Fix: `git filter-repo` или `bfg`.

**CI pipeline failures**:

- **Runner unavailable / stuck pending**: нет свободных runners. В UI GitLab — Settings > CI/CD > Runners: сколько активных? Если все busy, ждать. Если нет вообще — runner фейлится, kubectl логи runner Pod'а (в case k8s executor).
- **Compile error**: код не собирается. Смотреть build log, обычно очевидно.
- **Test failure**: JUnit XML в artifacts, GitLab UI показывает failed tests с stack traces.
- **Testcontainers timeout starting**: DinD или socket mount неправильно настроены. Kubectl exec в runner pod, попробовать `docker ps`.
- **Cache miss**: первый build ветки медленный, все dependencies качаются. Норма.
- **Static analysis findings**: checkstyle / spotbugs. Смотреть отчёт, чинить в коде.
- **Registry push failed**: `docker login` не сработал (secrets rotation?), диск registry полон, network issue.

**ArgoCD sync failures**:

- `ComparisonError`: невалидный YAML в manifests. Kustomize/Helm error.
- `SyncFailed`: admission webhook отверг apply. Смотреть ArgoCD event log — там будет reason от apiserver.
- `RBAC forbidden`: ArgoCD ServiceAccount не имеет прав на namespace / resource type. Fix — обновить ClusterRole/RoleBinding.

**K8s Pod failures**:

**ImagePullBackOff**. Image не найден в registry (typo в tag?), или auth failed (imagePullSecrets не настроены или неверные). Debug: `kubectl describe pod` → Events, там `Failed to pull image`. Смотреть URL, tag. Проверить: `docker pull` с той же машины срабатывает? Если да — проблема с imagePullSecrets. Kubectl get secret, base64 decode, проверить credentials.

**CrashLoopBackOff**. Контейнер запустился и упал, kubelet рестартует, снова падает. Debug: `kubectl logs <pod> --previous` — логи предыдущего запуска (текущий upcoming). Часто NullPointerException при старте, недоступная БД, missing config. Kubectl describe покажет exit code (137 = SIGKILL по OOM, 139 = segfault, 143 = SIGTERM cleanup, application-defined).

**Readiness never passes**. Startup probe eventually failure threshold, kubelet считает Pod broken, рестартует. Причины: HTTP endpoint не отвечает (порт не тот?), приложение не может подключиться к БД, sidecar не запустился. Debug: `kubectl port-forward <pod> 8080:8080` локально, `curl localhost:8080/actuator/health/readiness` — что отвечает? Смотреть Spring логи — что не запустилось.

**OOMKilled**. Container exit 137 с reason `OOMKilled`. Kubernetes убил Pod, потому что использовал больше памяти, чем в `resources.limits.memory`. Debug: смотреть JVM logs, размер heap (`-Xmx`), плюс metaspace, direct memory, threads (каждый thread — 512KB stack). Fix: увеличить `limits.memory` или уменьшить `-Xmx`.

Классическая ошибка: `limits.memory: 512Mi`, `-Xmx=512m` в JVM. JVM использует не только heap — metaspace, code cache, direct buffers, thread stacks. Реально нужно ~30% сверх heap. Rule of thumb: `-Xmx = 0.75 * limits.memory`.

**Pending forever**. Pod создан, но никогда не assigned на ноду. Причины:

- Нет ноды с достаточным CPU/memory для requests.
- Taint без tolerations.
- Node selector match'ит несуществующие ноды.
- PVC не bound (нет доступных PV).

Debug: `kubectl describe pod` → Events, там `FailedScheduling` с reason (`0/5 nodes are available: 3 Insufficient memory, 2 node(s) had taint {...}`).

**Rolling stuck**. `kubectl rollout status deployment/myapp` показывает `Waiting for deployment "myapp" rollout to finish`. Новые Pod'ы не становятся Ready → `maxUnavailable` не даёт убить старые → rollout висит. `progressDeadlineSeconds` истечёт (default 10 минут) → Deployment marked failed, но НЕ откатывается автоматически.

Debug: `kubectl get pods -l app=myapp`, найти non-Ready, `kubectl describe`, `kubectl logs`. Fix: либо `kubectl rollout undo deployment/myapp` (откатить на предыдущую ReplicaSet), либо фиксить новую версию и redeploy.

**In-flight requests обрываются во время rolling**. Client видит 502/503 или connection reset ровно во время rolling. Причина: между «Pod deleted from endpoints» и «iptables updated everywhere» есть окно 1-5 секунд, куда клиенты слали запросы на убитый Pod. Fix: `preStop sleep 10` + Spring Boot graceful shutdown + `terminationGracePeriodSeconds` > shutdown timeout.

## Сборка полной картины — что нужно уметь как senior

Когда прод горит и вам звонят в 3 часа — можно быстро локализовать проблему, если понимаете weight каждого звена.

**Симптом**: pipeline не запускается после push. Локализация: git server side. `curl` на GitLab health, посмотреть свою job в UI — есть ли она вообще? Post-receive hook отработал? Иногда GitLab сам болеет — Sidekiq queue огромная, jobs pending.

**Симптом**: pipeline упал в build. Локализация: код. Смотреть build log в CI, обычно очевидно — compile error, test failure. Иногда — flaky test (integration test с timing dependency), retry.

**Симптом**: pipeline прошёл, но образ не в реестре. Локализация: docker push. Смотреть логи `build-image` job, часто registry unavailable или auth issue.

**Симптом**: манифест обновлён, но ArgoCD не sync'нул. Локализация: ArgoCD. UI ArgoCD → Application → Sync status. Manual sync попробовать. Проверить permissions.

**Симптом**: ArgoCD sync'нул, Pods созданы, но не Ready. Локализация: kubectl. `kubectl get pods -n <ns>` — статусы. Failed Pod → `kubectl describe` → Events, там причина. `kubectl logs` для application logs.

**Симптом**: Pods Ready, но трафик не идёт. Локализация: networking. `kubectl get endpointslice -n <ns>` — есть ли Pod IP в endpoint'ах? `kubectl get svc -n <ns>` — есть ли service? Kube-proxy pod на ноде клиента — здоров? Ingress controller — здоров? Часто DNS issue: `kubectl exec <pod> -- nslookup <service>`.

**Симптом**: rolling stuck, старые Pod'ы не убираются. Локализация: readiness новых Pod'ов не проходит, `maxUnavailable=0` держит старые. Смотреть новые Pod'ы, что мешает readiness.

## Заключение

Full-stack понимание deploy pipeline — это не просто «знать все шаги». Это интуиция, где что живёт и как коммуницирует. Git wire protocol понимать нужно, чтобы диагностировать push проблемы. Server-side hooks — чтобы понимать, где расположить проверки. CI runners — потому что 80% инцидентов в разработке связаны с pipeline'ом. Maven и Gradle различия — потому что legacy проекты на Maven, новые на Gradle, знать оба стандарт. Docker layers — потому что оптимизация push/pull времени напрямую зависит от их устройства. GitOps через ArgoCD — стандарт enterprise. K8s Pod lifecycle — базовое знание для любого, кто деплоит в кластер.

Практический совет для становления: **прогоняйте pipeline руками, разбирая каждый stage**. Возьмите `.gitlab-ci.yml` реального проекта (например, вашего сервиса в КНП), пройдите каждый job, каждую shell-команду. Запустите build локально (`gradle build`), сравните с CI logs — что там ещё делается. Запустите `docker build`, посмотрите на layers через `docker image inspect`. Разверните ArgoCD в minikube, попробуйте sync-цикл. `kubectl describe` каждый ресурс — Deployment, ReplicaSet, Pod, Service, EndpointSlice. Читайте `kubectl get events`, чтобы видеть timeline создания Pod'а в реальном времени.

Когда следующий раз что-то не задеплоится — вы будете знать, куда смотреть. Не потому что читали, а потому что руками прошли через каждый слой. Это и есть senior-level понимание infrastructure.

**Maven phases** — 23 фазы default lifecycle, ключевые validate → compile → test → package → verify → install → deploy. Phase = все предыдущие phases выполняются. Phase ≠ Goal (goal — конкретное действие plugin'а).

**Gradle** — DAG задач, incremental через up-to-date checks, build cache для reuse cross-machine. Configuration cache — второй запуск в миллисекунды.

**Docker** — layer caching через content-addressable digests. Layered JAR для Spring Boot чтобы application layer был отдельно от dependencies. `.dockerignore` чтобы cache не invalidate от нерелевантных изменений. BuildKit — parallel steps, secret mounts, cache mounts.

**GitOps** — CI пушит только image в registry и manifest в manifest-repo. ArgoCD sync'ит manifest → cluster. CI никогда не имеет kubectl прав.

**K8s Pod lifecycle** — apiserver → Deployment controller → ReplicaSet controller → Pod creation → Scheduler → Kubelet → CRI → containerd → runc → Linux syscalls. Startup probe → readiness probe → EndpointSlice → kube-proxy iptables → traffic flows. Graceful shutdown через preStop + SIGTERM + terminationGracePeriodSeconds.

Понимая всю цепочку, вы понимаете, почему конкретные best practices важны — не потому что "так принято", а потому что каждая деталь решает конкретную проблему в конкретном месте цепочки.
