# 82. От `git push` до running Pod — полный процесс + Maven phases

Файл проводит **полный E2E путь** изменения от нажатия Enter после `git push` до момента когда новый Pod в K8s принимает трафик. Плюс глубоко разобраны **Maven lifecycle и phases** и **Gradle task graph**.

Связано с: `03-gradle-detailed.md` (Gradle базы), `09-docker-detailed.md` (Docker), `79-cicd-deploy-patterns.md` (CI/CD-паттерны), `80-kubernetes-internals-interview.md` (K8s), `81-build-vs-deployment-deep.md` (build vs deploy теория).

---

## 0. Полная карта

```
     ┌─────────────┐
     │ Developer   │
     │  git push   │
     └──────┬──────┘
            │  1. Git protocol (SSH/HTTPS)
            ▼
     ┌─────────────┐
     │  Git Server │ (GitLab/GitHub/Bitbucket)
     │  receives   │
     │  ref update │
     └──────┬──────┘
            │  2. Server-side hooks (pre-receive → update → post-receive)
            │  3. Webhook fired → CI system
            ▼
     ┌─────────────┐
     │ CI Runner   │ (GitLab Runner / GH Actions runner)
     │  picks job  │
     └──────┬──────┘
            │  4. Clone repo, prepare env
            ▼
     ┌─────────────┐
     │  Stage 1:   │  compile, unit tests
     │   BUILD     │  Maven/Gradle
     └──────┬──────┘
            │  Artifact: build/libs/app.jar
            ▼
     ┌─────────────┐
     │  Stage 2:   │  integration tests, static analysis
     │   TEST      │  Testcontainers, SonarQube
     └──────┬──────┘
            ▼
     ┌─────────────┐
     │  Stage 3:   │  docker build → image
     │   PACKAGE   │  layer caching
     └──────┬──────┘
            │  Push to Nexus/Harbor
            ▼
     ┌─────────────┐
     │  Stage 4:   │  update manifests in git repo
     │   DEPLOY    │  Kustomize/Helm
     └──────┬──────┘
            │  6. ArgoCD sees git change
            ▼
     ┌─────────────┐
     │  ArgoCD     │  kubectl apply to cluster
     │  syncs      │
     └──────┬──────┘
            │
            ▼
     ┌─────────────┐
     │ K8s cluster │  Deployment controller → ReplicaSet → Pod → Scheduler
     │  processes  │  Kubelet → CRI → containerd → runc → running container
     └──────┬──────┘
            │
            ▼
     ┌─────────────┐
     │  New Pod    │  Readiness passes → EndpointSlice updated → traffic
     │  serves     │
     │  traffic    │
     └─────────────┘
```

Каждая стрелка — отдельная сеть/протокол/технология. Разбираем каждое звено.

---

## 1. `git push` — что реально происходит

### 1.1 Клиентская сторона

`git push origin release-ISNA2-23651` — команда клиенту.

1. **Git читает `.git/config`** — находит remote `origin` URL (например `git@gitlab.1sc.kz:isna/knp.git`).
2. **Определяет protocol**: `git://` (deprecated), `https://`, `ssh://`, `file://`. Для SSH идёт по `ssh` бинарнику; для HTTPS — HTTP CONNECT.
3. **Handshake**: клиент говорит серверу «хочу push на этот repo».
4. **Ref discovery**: сервер отдаёт список ссылок (branches, tags) с их SHA. Клиент вычисляет что нужно послать.
5. **Pack negotiation**: клиент говорит «хочу отправить refs A, B; я знаю что у сервера есть C, D». Сервер: «пришли мне commits/trees/blobs которых нет».
6. **Pack file creation**: клиент собирает `.pack` файл (delta-compressed объекты) + `.idx` index.
7. **Upload**: pack streamed по сети.
8. **Ref update**: клиент говорит «обнови ref X с SHA Y на SHA Z».

Всё через **git wire protocol** (сейчас в основном v2, K8s 1.18+). Смотреть трейс: `GIT_TRACE=1 git push`.

### 1.2 Что летит по сети

**Не файлы** — pack. Это оптимизированный бинарный формат:
- Комбинация полных blob'ов и delta'ей (разница от других blob'ов).
- zlib-compressed.
- Один pack на push.

Пример: изменил 3 строки в 1 файле → летит ~KB, не мегабайт.

### 1.3 Server-side hooks — три штуки

Git server (GitLab/GitHub/Bitbucket) выполняет hooks:

**pre-receive** (глобальный, для всего push'а):
- Проверка permissions.
- Проверка ссылок (protected branches — нельзя push в `main` напрямую).
- Git LFS validation.
- Может отклонить push целиком.

**update** (для каждого ref):
- Ветка-специфичные правила.

**post-receive** (после успешного push):
- **Триггерит webhook'и / CI pipelines**.
- Уведомления (Slack, email).
- Update mirror repos.

### 1.4 GitLab-специфично

После post-receive:
1. GitLab обновляет БД: `merge_requests`, `pipelines`, `commits`.
2. Если push в ветку с MR → MR обновляется (новый commit).
3. **Проверяется `.gitlab-ci.yml`** в вершине push'а:
   - Есть? Есть stages? Есть job'ы для этой ветки (правила `only:`, `rules:`)?
   - Создаётся `Pipeline` object в БД.
   - Создаются `Job` objects по stages.
4. **Jobs с `when: on_success` в первом stage** — сразу помечаются `pending`.
5. **GitLab Runner** через long polling или webhook получает job.

Аналогично для GitHub Actions — `on: push` triggers → workflow runs → jobs distributed to runners.

### 1.5 SSH vs HTTPS

**SSH**:
- Аутентификация по ключам (`~/.ssh/id_rsa`).
- Быстрее (нет TLS handshake каждый раз).
- Работает через firewall'ы плохо (специальный порт 22 или 443 shim).

**HTTPS**:
- Аутентификация по username/password или Personal Access Token (PAT).
- Universal (works через любой прокси).
- Медленнее из-за TLS + auth per request.
- Credential caching (git credential helpers).

Enterprise часто использует HTTPS через прокси. Разработчики — SSH.

---

## 2. Webhook и trigger CI

### 2.1 Webhook

Git server шлёт HTTP POST на CI URL с payload:
```json
{
  "object_kind": "push",
  "ref": "refs/heads/release-ISNA2-23651",
  "before": "abc123...",
  "after": "def456...",
  "commits": [...],
  "repository": {...},
  "user_name": "zainiddinov.b"
}
```

CI получает → парсит → решает: какие workflows запускать.

### 2.2 Внутренний GitLab CI

Если GitLab и Runner в одной инфре — не через HTTP webhook, а через **long polling**. Runner сидит и ждёт: «есть работа?». GitLab отвечает JSON с job'ой когда есть.

### 2.3 Runners

**GitLab Runner** — бинарь, зарегистрирован в GitLab с token'ом. Executors:
- **shell** — прямо на хост-машине runner'а.
- **docker** — каждый job в отдельном контейнере.
- **kubernetes** — каждый job = Pod в K8s кластере.
- **docker-machine** — spawns VM per job (auto-scaling).

**GitHub Actions runners**:
- **Hosted**: GitHub предоставляет (Ubuntu/Windows/macOS VMs).
- **Self-hosted**: свой runner (аналог GitLab Runner).

**Особенности K8s executor** (популярный для внутренних CI):
- Каждый job — Pod с несколькими контейнерами (build-container + services типа Postgres для tests).
- Ephemeral — Pod удаляется после job.
- Секреты как K8s Secrets, монтированные в Pod.

Runner для нашего job:
1. Опрашивает GitLab.
2. Получает job spec: image, script, artifacts, cache.
3. **Создаёт Pod** (в K8s executor).
4. Клонирует repo в Pod.
5. Выполняет script.
6. Загружает artifacts (jar, test reports).
7. Освобождает Pod.

---

## 3. Pipeline structure (`.gitlab-ci.yml`)

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
  allow_failure: false

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
      coverage_report:
        coverage_format: cobertura
        path: build/reports/jacoco/test/jacocoTestReport.xml
    when: always

integration-test:
  stage: test
  services:
    - name: postgres:16
      alias: db
    - name: rabbitmq:3.13
      alias: rabbit
  variables:
    POSTGRES_PASSWORD: test
    SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/test
    SPRING_RABBITMQ_HOST: rabbit
  script:
    - gradle integrationTest --no-daemon
  artifacts:
    reports:
      junit: build/test-results/integrationTest/*.xml

security-scan:
  stage: security
  script:
    - trivy fs --exit-code 1 --severity HIGH,CRITICAL .
    - gradle dependencyCheckAnalyze --no-daemon
  allow_failure: false

sonarqube:
  stage: security
  script:
    - gradle sonarqube -Dsonar.host.url=$SONAR_URL -Dsonar.login=$SONAR_TOKEN

build-image:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build 
        --cache-from $CI_REGISTRY/isnaknpuser:latest
        --tag $CI_REGISTRY/isnaknpuser:$CI_COMMIT_SHORT_SHA
        --tag $CI_REGISTRY/isnaknpuser:latest
        .
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY/isnaknpuser:$CI_COMMIT_SHORT_SHA
    - docker push $CI_REGISTRY/isnaknpuser:$CI_COMMIT_SHORT_SHA
    - docker push $CI_REGISTRY/isnaknpuser:latest

deploy-staging:
  stage: deploy-staging
  script:
    - git clone $MANIFESTS_REPO manifests
    - cd manifests/apps/isnaknpuser/staging
    - kustomize edit set image myapp=$CI_REGISTRY/isnaknpuser:$CI_COMMIT_SHORT_SHA
    - git commit -am "Staging: isnaknpuser $CI_COMMIT_SHORT_SHA"
    - git push
  only: [main, release]

smoke-test:
  stage: integration-test
  script:
    - ./scripts/wait-for-deploy.sh isnaknpuser staging 300
    - ./scripts/smoke.sh https://staging.knp.gov.kz/api

deploy-prod:
  stage: deploy-prod
  when: manual
  script:
    - git clone $MANIFESTS_REPO manifests
    - cd manifests/apps/isnaknpuser/prod
    - kustomize edit set image myapp=$CI_REGISTRY/isnaknpuser:$CI_COMMIT_SHORT_SHA
    - git commit -am "Prod: isnaknpuser $CI_COMMIT_SHORT_SHA"
    - git push
  only: [master]
```

### 3.1 Ключевые концепции pipeline

- **Stages** — последовательные фазы, все job'ы одного stage — параллельно.
- **Jobs** — единицы работы, привязаны к stage.
- **Artifacts** — файлы передаваемые между job'ами (jar, test reports).
- **Cache** — переиспользуемые файлы (dependency caches, Gradle cache).
- **Variables** — env vars, могут быть в pipeline / project / group / global.
- **Services** — дополнительные контейнеры рядом с job'ом (БД для тестов).
- **Rules / only / except** — когда job запускается.
- **when: manual** — job требует ручного триггера.

### 3.2 Артefacts vs Cache

**Artifacts** — результат job'а. Загружаются в GitLab, скачиваются следующими job'ами. Immutable, versioned.

**Cache** — временное хранилище для ускорения. Может быть invalid, не гарантирован. Обычно `.gradle/caches`, `node_modules`.

Разница: artifact пишет job, читает pipeline. Cache — читает и пишет job, между запусками.

---

## 4. Stage: Build — Gradle детально

### 4.1 Task graph

Gradle работает через **DAG задач** (directed acyclic graph). Каждая задача (`compile`, `test`, `jar`) имеет:
- **inputs** (файлы).
- **outputs** (файлы).
- **dependsOn** (другие задачи).

При запуске `gradle build`:
1. Читает `build.gradle` — конфигурирует все задачи (описываются).
2. **Определяет task graph** — какие задачи нужны для `build`.
3. **Смотрит up-to-date checks**: если inputs не менялись с прошлого запуска → skip (использует output).
4. Выполняет задачи в порядке DAG.

### 4.2 Стандартные задачи Java plugin

```
compileJava           .java → .class (main)
processResources      copy src/main/resources
classes               composite: compileJava + processResources
compileTestJava       .java → .class (test)
processTestResources  copy src/test/resources
testClasses           composite
jar                   pack classes + resources → JAR
test                  run unit tests (JUnit)
check                 test + all verification (checkstyle, spotbugs)
assemble              build all archives (jar + war if applicable)
build                 check + assemble  ← самая частая цель
```

### 4.3 Задачи Spring Boot plugin

Добавляет:
```
bootJar               ← собирает fat JAR
bootRun               ← запускает Boot app
bootBuildImage        ← собирает OCI image через Buildpacks (без Dockerfile!)
```

`bootJar` overrides `jar` — по умолчанию только `bootJar` активен (regular `jar` disabled).

### 4.4 Up-to-date check

Ключевая оптимизация Gradle. Task **skips** если:
- Все `inputs` не изменились (hashes/mtimes).
- Все `outputs` существуют.
- Task properties не изменились.

Проверить: `gradle compileJava --info`
```
> Task :compileJava UP-TO-DATE
```

Полный rebuild: `gradle clean build`. `clean` удаляет `build/`, force всё пересобрать.

### 4.5 Build cache

Есть **local cache** (`~/.gradle/caches/build-cache-1/`) и **remote build cache** (HTTP).

Task output кэшируется по hash inputs. Если тот же hash был раньше — output достаётся из cache, task **from-cache** (fast copy вместо real work).

Работает через границы machines: коллега собрал → ты пулишь → у тебя те же inputs → cache hit → мгновенно.

Enable:
```gradle
// gradle.properties
org.gradle.caching=true
```

Remote:
```gradle
// settings.gradle
buildCache {
    remote(HttpBuildCache) {
        url = 'https://build-cache.example.com/'
        push = true
    }
}
```

### 4.6 Configuration cache

Gradle 6.6+. Кэширует не только outputs, но и **конфигурацию** (парсинг `build.gradle`). Второй запуск — старт за миллисекунды.

Enable: `--configuration-cache` или `org.gradle.configuration-cache=true`.

Not all plugins support — check compatibility.

### 4.7 Parallel execution

`org.gradle.parallel=true` — независимые модули собираются параллельно. Для multi-module project — большой win.

Ограничение: параллельность **между** проектами, не задачами одного проекта.

### 4.8 Что реально исполняется на `gradle build`

Для типичного Spring Boot проекта:
```
1. compileJava            (2-30s, зависит от размера)
2. processResources       (миллисекунды)
3. classes                (composite, ничего не делает)
4. compileTestJava        (5-20s)
5. processTestResources
6. testClasses
7. test                   (сколько тестов + startup Spring, 30s-5min)
8. bootJar                (2-10s — сборка fat JAR)
9. jar                    (skipped, только bootJar активен)
10. assemble              (composite)
11. checkstyleMain        (5-15s)
12. checkstyleTest
13. spotbugsMain          (медленный, 10-60s)
14. spotbugsTest
15. jacocoTestReport      (5-10s)
16. check                 (composite)
17. build                 (composite → done)
```

Полный `gradle build` для 500KLOC проекта — 5-15 минут первый раз, 30 секунд-2 минуты с caches.

---

## 5. Maven lifecycle и phases — углублённо

### 5.1 Три встроенных lifecycles

Maven имеет **три lifecycle**, каждый — последовательность **phases**:

1. **default** — build цикл (compile, test, package, install, deploy).
2. **clean** — очистка (`mvn clean` = удалить `target/`).
3. **site** — генерация documentation site.

Ты пишешь `mvn clean install` — это выполнение **двух lifecycles**: сначала `clean` полностью, потом `default` до phase `install`.

### 5.2 Default lifecycle — 23 phases по порядку

Maven default lifecycle имеет 23 фазы. Не все используются часто, но знать порядок нужно:

```
validate            ← Проверка что проект корректный.
initialize          ← Инициализация build state.
generate-sources    ← Генерация исходников (например MapStruct annotation processors).
process-sources     ← Обработка исходников (filtering).
generate-resources  ← Генерация ресурсов.
process-resources   ← Обработка ресурсов (variable substitution, filter).
compile             ← .java → .class (main).
process-classes     ← Post-processing bytecode (например Lombok delombok).
generate-test-sources
process-test-sources
generate-test-resources
process-test-resources
test-compile        ← .java → .class (test).
process-test-classes
test                ← Unit tests (Surefire).
prepare-package     ← Подготовка к упаковке.
package             ← Создание JAR/WAR.
pre-integration-test
integration-test    ← Integration tests (Failsafe).
post-integration-test
verify              ← Проверка что integration tests прошли.
install             ← Копирование в local repo (~/.m2/repository).
deploy              ← Upload в remote repo (Nexus, Artifactory).
```

### 5.3 Ключевое правило Maven

**Выполнение phase X = выполнение ВСЕХ предыдущих phases**.

`mvn package` → validate + initialize + ... + compile + ... + test + prepare-package + package. Всё в порядке.

Нельзя запустить только `test`, пропустив `compile` — компиляция обязательна.

### 5.4 Phase vs Goal

**Phase** — точка в lifecycle. Сама по себе ничего не делает.
**Goal** — конкретное действие плагина. Пример: `compiler:compile`.

**Phases привязаны к goals через plugin bindings**. Например:
- `compile` phase → `maven-compiler-plugin:compile` goal.
- `test` phase → `maven-surefire-plugin:test` goal.
- `package` phase → `maven-jar-plugin:jar` goal (или `war` для WAR-проекта).

Ты можешь запустить goal напрямую:
```bash
mvn compiler:compile          # только компиляция, без предыдущих phases
mvn dependency:tree           # без ничего, показать зависимости
mvn spring-boot:run           # запустить приложение
```

**Разница**: `mvn compile` = run phase (со всеми предыдущими); `mvn compiler:compile` = run одну goal.

### 5.5 Default plugin bindings по packaging

Bindings зависят от `<packaging>` в `pom.xml`. Для JAR project (default):

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

Для WAR — `package` → `war:war`. Для EAR — `ear:ear`.

### 5.6 Практические команды

```bash
mvn clean                  # только clean lifecycle
mvn compile                # default до phase compile
mvn test                   # + test
mvn package                # + package (создать JAR/WAR в target/)
mvn verify                 # + integration tests
mvn install                # + install в ~/.m2
mvn deploy                 # + upload в Nexus/Artifactory

mvn clean package          # два lifecycles: clean, потом default до package
mvn -DskipTests package    # пропустить тесты (компиляция тестов есть)
mvn -Dmaven.test.skip=true # пропустить и компиляцию, и запуск
```

**Разница `-DskipTests` и `-Dmaven.test.skip`**:
- `-DskipTests` — test-classes компилируются, но не запускаются.
- `-Dmaven.test.skip` — вообще не компилируются.

### 5.7 Многомодульный build (reactor)

`pom.xml` с `<modules>`:
```xml
<modules>
    <module>api</module>
    <module>domain</module>
    <module>web</module>
</modules>
```

`mvn install` в корне → **reactor** определяет порядок модулей по зависимостям → выполняет для каждого модуля весь lifecycle.

Полезные флаги:
```bash
mvn install -pl web -am     # только модуль web + его зависимости
mvn install -pl web -amd    # только web + модули которые зависят от web
mvn install -rf domain      # resume от модуля domain (если что-то упало)
mvn install -T 4C           # threads: 4 per CPU core (параллельная сборка модулей)
```

### 5.8 Профили (profiles)

Условное выполнение или конфигурация. Например:
```xml
<profiles>
    <profile>
        <id>prod</id>
        <build>
            <finalName>myapp-prod</finalName>
        </build>
    </profile>
    <profile>
        <id>skip-frontend</id>
        <build>
            <plugins>
                <plugin>
                    <artifactId>frontend-maven-plugin</artifactId>
                    <configuration>
                        <skip>true</skip>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

Активация: `mvn -Pprod install` или через `<activation>` (по env var, OS, file existence, JDK version).

### 5.9 Effective POM

Реальный POM с applied inheritance от parent + super POM + resolved variables:
```bash
mvn help:effective-pom > effective-pom.xml
```

Смотреть когда: «почему у меня версия plugin'а не та что в моём pom?» — parent или default.

### 5.10 Dependency management

Maven резолвит транзитивные зависимости через **nearest-wins**: если A→B→C:1.0 и A→D→C:2.0, оба на одинаковой глубине — берётся **первое объявленное** (в pom.xml order).

Смотреть:
```bash
mvn dependency:tree
mvn dependency:tree -Dverbose  # покажет отсеянные версии
```

Force версии:
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

vs Gradle который **highest-wins** (выбирает наибольшую версию).

### 5.11 Собесное про Maven

**Q: Разница `install` и `deploy`?**  
`install` — копирует artifact в **local repo** `~/.m2/repository/`. Доступно только тебе.  
`deploy` — upload в **remote repo** (Nexus/Artifactory). Доступно всей команде.

**Q: Зачем нужен `mvn verify` отдельно от `mvn test`?**  
`test` — unit tests (Surefire). `verify` — integration tests (Failsafe). Разделение — чтобы unit-тесты гонять при каждом commit'е, integration — реже.

**Q: Что произойдёт если запустить `mvn install` без предварительного `mvn compile`?**  
Ничего плохого. Install требует compile → phase compile выполнится автоматически (все предыдущие phases).

**Q: Разница Maven и Gradle?**  
Maven — XML-based, opinionated (convention over configuration), plugin-based, medium скорость.  
Gradle — Groovy/Kotlin DSL, гибкий, task-based DAG, быстрее (incremental, cache), сложнее debug'ить билд.

Для новых Java projects — обычно Gradle. Для legacy — Maven.

---

## 6. Stage: Test — что реально проверяется

### 6.1 Иерархия тестов

```
    ┌─────────────────────┐
    │   E2E Tests         │  ← Selenium, Cypress
    │   (slow, few)       │     Тестируют полный flow через UI
    ├─────────────────────┤
    │   Integration Tests │  ← Testcontainers, Spring Boot Test
    │   (medium)          │     Реальная БД, HTTP endpoints
    ├─────────────────────┤
    │   Unit Tests        │  ← JUnit, Mockito
    │   (fast, many)      │     Классы в изоляции
    └─────────────────────┘
```

Test pyramid — правильное соотношение: много быстрых unit, средне integration, мало e2e.

### 6.2 Unit Tests

Каждый Java класс — свой test класс. Mock dependencies.

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

Runner (Surefire для Maven / Gradle test task): собирает test classpath, запускает JUnit engine, отчёт в `build/test-results/`.

### 6.3 Integration Tests

Реальный Spring context, реальная БД. Обычно **Testcontainers** — запускает Postgres/RabbitMQ/etc в Docker для теста.

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

При запуске:
1. Testcontainers через Docker API поднимает Postgres контейнер (~3-10 сек).
2. Spring context стартует, Liquibase/Flyway мигрирует schema.
3. Test runs.
4. Контейнер убивается после test.

**В CI** — нужен docker-in-docker или socket mount, чтобы Testcontainers мог поднимать контейнеры внутри runner-контейнера.

### 6.4 Spring Boot slices

`@SpringBootTest` — весь context, медленно. Slices — только часть:
- `@WebMvcTest` — только контроллеры (MockMvc).
- `@DataJpaTest` — только JPA layer (H2 in-memory по умолчанию).
- `@JsonTest` — только Jackson serialization.
- `@RestClientTest` — REST client + MockRestServer.

Быстрее чем full context, но не покрывают integration полностью.

### 6.5 Contract tests

Между сервисами (микросервисы). **Pact / Spring Cloud Contract** — provider публикует contract, consumer тестируется против mock generated из contract'а.

Идея: изменение API у provider'а ломает contract → CI консюмера падает **до** deploy provider'а.

### 6.6 Что делает CI на этапе test

```
1. gradle test (или mvn test) — unit тесты
2. Собирает reports (JUnit XML в build/test-results/test/*.xml)
3. Uploads в GitLab / GitHub Actions как artifact
4. GitLab парсит XML, показывает в UI:
   - Total / passed / failed / skipped
   - Duration
   - Failure messages
5. Если failed > 0 → job fails → pipeline stops

Затем:
6. gradle integrationTest (отдельный task, отдельный SourceSet)
7. Testcontainers поднимает БД в докере
8. Тесты гоняются
9. Reports собираются
```

**Test report artifact** — крайне полезен: разработчик видит какие именно тесты упали без залезания в CI runner.

---

## 7. Stage: Package — Docker build

### 7.1 Multi-stage Dockerfile для Spring Boot

```dockerfile
# Stage 1: build
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /workspace
COPY build.gradle settings.gradle gradlew ./
COPY gradle gradle
COPY src src
# Скачиваем deps в отдельный слой (кэшируется)
RUN ./gradlew dependencies --no-daemon
# Собираем
RUN ./gradlew bootJar --no-daemon

# Stage 2: runtime
FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app
# Копируем только jar из builder stage
COPY --from=builder /workspace/build/libs/*.jar app.jar
# Non-root user
RUN groupadd -r spring && useradd -r -g spring spring
USER spring
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

**Multi-stage**: только последний stage попадает в финальный image. Builder stage (JDK, source, deps) — отбрасывается. Image ~200MB вместо ~800MB.

### 7.2 Layer optimization

Правило: **редко меняющееся — раньше в Dockerfile**. Каждая RUN/COPY = слой. Слой кэшируется если inputs не изменились.

Плохо:
```dockerfile
COPY . /workspace           # весь source — часто меняется
RUN ./gradlew bootJar       # invalidates cache при любом изменении
```

Хорошо (Spring Boot layered JAR):
```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
# Deps редко меняются
COPY --from=builder /workspace/build/libs/dependencies/ ./
COPY --from=builder /workspace/build/libs/spring-boot-loader/ ./
COPY --from=builder /workspace/build/libs/snapshot-dependencies/ ./
# Твой код часто меняется — последний слой
COPY --from=builder /workspace/build/libs/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

Изменил Java-файл → повторный build кэширует все слои кроме `application/`. Push в registry — только этот слой (обычно ~10MB), не весь image.

### 7.3 Cloud Native Buildpacks — без Dockerfile

Spring Boot 2.3+ имеет `bootBuildImage`:
```bash
gradle bootBuildImage --imageName=myapp:1.0
```

Собирает OCI image без Dockerfile через **Paketo Buildpacks**. Автоматически:
- Detect language (Java).
- Install правильную JVM version.
- Optimize layers.
- Set non-root user.
- Set Java memory options based on container limits.

Плюсы: security patches автоматически (rebuild image с новым buildpack version → новый base). Best practices baked in.

Минусы: меньше контроля, cold start slower первый раз.

### 7.4 Что происходит на `docker build`

1. **Docker CLI** отправляет **build context** (текущая директория, кроме `.dockerignore`) на **Docker daemon**.
2. Daemon читает Dockerfile.
3. Для каждой инструкции:
   - Проверяет cache: есть ли слой с таким же hash inputs?
   - Cache hit → reuse.
   - Cache miss → создаёт новый контейнер, выполняет команду, commits слой.
4. Финальный image = список layer hashes + manifest.
5. Ты видишь тег `myapp:1.0.42` — это pointer на manifest.

### 7.5 Docker daemon vs Buildkit

Классический Docker — sequential build.
**BuildKit** (default с Docker 23) — parallel где возможно, лучше caching, mount secrets в build time.

Enable: `DOCKER_BUILDKIT=1 docker build ...` или в Docker daemon.

### 7.6 Push в registry

```bash
docker push registry.1sc.kz/isnaknpuser:1.0.42
```

Что происходит:
1. Docker CLI шлёт запрос daemon'у: «push этот image».
2. Daemon сравнивает layers с тем что уже в registry (по digest).
3. Отсылает **только те layers которые registry не имеет**.
4. Отправляет **manifest** (JSON со списком layer digest'ов + config).
5. Registry сохраняет layers (в S3/blob storage) + manifest.
6. Ставит tag.

Base image (eclipse-temurin:21-jre) — обычно уже в registry, не пересылается. Application layer (10-100 MB) — пересылается.

Total: обычно ~10-50 MB на push (только application layer), даже если image 300 MB.

### 7.7 OCI Distribution Spec

Docker Registry API v2 = OCI Distribution Spec. Endpoints:
- `PUT /v2/<name>/blobs/uploads/` — начать upload.
- `PATCH ...` — чанки.
- `PUT ...` — финальный.
- `PUT /v2/<name>/manifests/<tag>` — обновить tag.
- `GET /v2/<name>/manifests/<tag>` — pull.
- `GET /v2/<name>/blobs/<digest>` — pull layer.

Всё content-addressable (digest = sha256 контента). Immutable: layer с digest X никогда не меняется.

---

## 8. Stage: Deploy — GitOps flow

### 8.1 CI не деплоит в кластер

**Anti-pattern**: `kubectl apply` из CI. Проблемы:
- CI runner имеет admin права → атака на runner = уничтожение кластера.
- Нет audit trail в git.
- Drift между git и cluster.

**Правильно** — GitOps.

### 8.2 GitOps step-by-step

Repos:
- `application-repo` — source code Java-приложения.
- `manifests-repo` — K8s манифесты (Deployment, Service, ConfigMap YAML) для всех сервисов, всех окружений.

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
  2. Сравнивает: git ↔ etcd в кластере
  3. Видит diff: image changed
  4. kubectl apply -f deployment.yaml
       ↓
K8s начинает rolling update.
```

Только у **ArgoCD** есть kubectl права. CI runner имеет только git write права к manifests-repo. Атакер получил CI runner? Может делать commits, но commit review в manifests-repo может защитить.

### 8.3 Kustomize edit set image

Kustomize кладёт изменения в `kustomization.yaml`:
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

`kubectl apply -k .` → Kustomize:
1. Читает `base/deployment.yaml`, `base/service.yaml`.
2. Применяет image override.
3. Отсылает финальный manifest в apiserver.

### 8.4 ArgoCD sync

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

Каждые 3 минуты ArgoCD:
1. `git fetch` из repoURL.
2. Compare desired (from git) vs actual (from cluster etcd).
3. Если drift → `kubectl apply` в свою очередь.

`selfHeal: true` — если кто-то `kubectl edit` руками, ArgoCD возвращает из git.

---

## 9. K8s Pod update — что реально происходит

Момент истины: apply в apiserver → running Pod. Пошагово, с временными оценками.

### 9.1 T = 0: apiserver получает apply

```
kubectl apply -f deployment.yaml  (или ArgoCD apply)
     ↓
apiserver:
  - Auth (TLS cert / token) ~1ms
  - Authorization (RBAC check) ~1ms
  - Mutating admission (defaults, sidecar injection) ~5-50ms
  - Schema validation ~1ms
  - Validating admission (webhooks) ~5-50ms
  - etcd write (2-node Raft commit) ~5-20ms
  Total: ~10-100ms
     ↓
apiserver returns 200 to client
```

Возвращённый объект имеет `resourceVersion` — увеличилось на 1.

### 9.2 T = 100ms: Deployment controller видит event

controller-manager (leader-elected) держит watch на Deployments:
```
watch event: MODIFIED deployment/myapp
     ↓
Deployment controller reconcile:
  - Read current spec (new template)
  - Compute template hash
  - Search existing ReplicaSets — есть ли RS с этим hash?
    - Нет → создать новый ReplicaSet (RS-abc) с 0 replicas
    - Есть → использовать (случай rollback)
  - Update Deployment status: newReplicaSet: RS-abc
  - Scale RS-abc up (to 1 replica for maxSurge=1)
  - Scale old RS (RS-xyz) — оставляем 4 replicas пока
```

`kubectl apply` создал новый RS, но старый ещё активен.

### 9.3 T = 200ms: ReplicaSet controller создаёт Pod

```
watch event: MODIFIED replicaset/RS-abc (replicas 0 → 1)
     ↓
ReplicaSet controller:
  - Считает currentPods (0), desired (1) → нужен 1 pod
  - Создаёт Pod object в apiserver
```

Pod создан с `nodeName: ""` — ещё не назначен на ноду.

### 9.4 T = 300ms: Scheduler назначает ноду

Scheduler держит watch на Pod'ах с `nodeName == ""`:
```
watch event: ADDED pod/myapp-abc-xyz (unscheduled)
     ↓
Scheduler:
  - Filter phase: какие ноды подходят?
    - resource requests
    - taints/tolerations
    - node affinity
    - pod affinity/anti-affinity
    - PVC availability
  - Score phase: 0-100 для каждой feasible node
  - Best node = worker-3
  - Bind: PATCH pod with spec.nodeName=worker-3
```

Pod теперь имеет `nodeName`.

### 9.5 T = 400ms: Kubelet на worker-3 видит Pod

Kubelet на каждой ноде watch'ит Pod'ы с `nodeName == self`:
```
watch event: MODIFIED pod/myapp-abc-xyz (nodeName=worker-3)
     ↓
Kubelet syncLoop:
  1. Pull image (если нет локально):
     - CRI RunPodSandbox() → pause контейнер создан (~200ms)
     - Container runtime pulls image
       - Если image в locale cache → 0
       - Иначе HTTP GET к registry:
         - Manifest (~10KB, ~50ms)
         - Layers (10-500MB, зависит от network, обычно 5-60 sec)
     - Image готов
  2. Mount volumes:
     - ConfigMap → files в emptyDir
     - Secret → files в tmpfs
     - PVC → CSI attach + mount (может быть 30sec-2min для облачных disks)
     - Serviceaccount token → projected volume
  3. Create container:
     - CRI CreateContainer() — containerd создаёт config.json (OCI runtime spec)
     - Setup network (CNI plugin)
       - CNI creates veth pair
       - Assign IP from pool
       - Setup routes
     - runc create — Linux syscalls (clone, unshare, pivot_root)
  4. Start container:
     - CRI StartContainer()
     - runc start — main process runs
     - kubelet updates Pod status: containerStatuses[0].state.running
```

Общее время от step 1 (image pull) до step 4 (container running): 
- **Image в cache**: 1-3 сек.
- **Image cold** (pull from registry): 10-60 сек.

### 9.6 T = 5-30s: приложение стартует

Java Spring Boot cold start:
- JVM launch: ~1 sec.
- Class loading: 5-15 sec.
- Spring context init:
  - Bean discovery: 3-10 sec.
  - Component scanning: 2-5 sec.
  - Autoconfig: 5-15 sec.
- Database connect + migration check: 2-10 sec.
- Embedded Tomcat start: ~1 sec.
- Total: **20-45 sec** для типичного Spring Boot приложения.

**Startup probe** обычно настроен на это окно:
```yaml
startupProbe:
  httpGet: {path: /actuator/health/liveness, port: 8080}
  failureThreshold: 30
  periodSeconds: 10
  # 30 × 10 = 5 минут макс на старт
```

### 9.7 T = 30s: Readiness passes

Kubelet опрашивает `/actuator/health/readiness`:
```
GET http://<pod-ip>:8080/actuator/health/readiness
Response: 200 {"status": "UP"}
     ↓
Kubelet updates Pod status: conditions[Ready] = True
     ↓
Watch event: MODIFIED pod (Ready=True)
```

### 9.8 T = 30.1s: EndpointSlice controller добавляет Pod

```
EndpointSlice controller видит: Pod myapp-abc-xyz Ready, matches Service selector
     ↓
Update EndpointSlice endpointslices/myapp-abc:
  add {addresses: [10.244.1.5], targetRef: pod/myapp-abc-xyz, ready: true}
     ↓
etcd write
```

### 9.9 T = 30.2s: kube-proxy обновляет iptables

На каждой ноде kube-proxy держит watch на EndpointSlices:
```
watch event: MODIFIED endpointslice/myapp-abc
     ↓
kube-proxy sync loop:
  - Rebuild iptables rules
  - iptables-restore --wait (atomic swap)
     ↓
Правила для Service myapp теперь включают 10.244.1.5:8080
```

Occurs on **всех нодах кластера**, не только на worker-3.

### 9.10 T = 30.5s — 34s: iptables rules propagate

Разные ноды обновляют iptables в разное время. Максимум обычно 1-5 сек. С этого момента:
- Client шлёт на ClusterIP:80.
- Kube-proxy на его ноде — правило DNAT: с равной вероятностью на любой из Ready Pod'ов.
- Новый Pod получает трафик.

### 9.11 T = 34s: Deployment controller scale down старого

Deployment controller видит: newRS теперь 1 Pod Ready → можно убить 1 старый:
```
Scale RS-xyz (старый) с 4 до 3
     ↓
ReplicaSet controller выбирает Pod для удаления (по deletion cost, oldest first)
     ↓
Delete pod/myapp-xyz-abc
     ↓
apiserver: soft delete (deletionTimestamp set)
     ↓
Kubelet на worker-2 (где старый Pod) видит:
  1. Marks Pod as terminating
  2. EndpointSlice controller REMOVES Pod IP from endpoints
     - kube-proxy обновляет iptables — Pod исключён из балансировки
  3. Kubelet execute preStop hook (например sleep 10)
     - В это время новые запросы не приходят (Pod вне endpoints)
     - Но in-flight запросы завершаются
  4. Kubelet sends SIGTERM to main container
  5. Spring Boot: graceful shutdown
     - Stops accepting new requests
     - Waits for in-flight to complete
     - Closes datasource connections gracefully
     - Container exits 0
  6. Kubelet sees exit → CRI StopContainer, RemovePodSandbox
  7. Pod object removed from apiserver
```

### 9.12 T = 34s → 60s: продолжение rolling

Deployment controller продолжает:
- Scale RS-abc с 1 до 2 → new pod → ждать readiness → scale RS-xyz с 3 до 2.
- И так далее.

Общее время rolling update для 4 replicas × 30 sec startup = 2-3 минуты.

### 9.13 T = 3 min: rolling complete

```
kubectl rollout status deployment/myapp
> deployment "myapp" successfully rolled out
```

Все Pod'ы новой версии, старые убраны.

---

## 10. Полный timeline пример

Для типичного микросервиса в КНП:

| Time | Event |
|------|-------|
| 0:00 | `git push origin release-ISNA2-XXXXX` |
| 0:01 | GitLab receives push, triggers pipeline |
| 0:02 | Pipeline starts, stage `validate` |
| 0:15 | validate done (checkstyle, spotbugs) |
| 0:16 | Stage `build` starts |
| 0:45 | Gradle compile done, jar built |
| 0:46 | Stage `test` starts, parallel unit + integration |
| 2:30 | Unit tests done |
| 3:45 | Integration tests done (Testcontainers Postgres) |
| 3:46 | Stage `security` starts |
| 4:30 | Trivy + OWASP done |
| 4:31 | Stage `package` starts |
| 4:35 | docker build (layer cache good) |
| 4:40 | trivy image scan done |
| 4:50 | docker push to Nexus (application layer only) |
| 4:51 | Stage `deploy-staging` starts |
| 4:53 | Manifest repo updated, git push |
| 4:53 | ArgoCD webhook triggered |
| 4:54 | ArgoCD syncs staging |
| 4:54 | apiserver receives apply, Deployment updated |
| 4:54 | New ReplicaSet created, first Pod created |
| 4:54 | Scheduler assigns node |
| 4:55 | Kubelet pulls image (cache hit) |
| 4:56 | Container starts |
| 5:20 | Spring Boot fully started, readiness Ready |
| 5:20 | EndpointSlice updated, kube-proxy propagates |
| 5:21 | Old Pod terminated (rolling continues) |
| 5:45 | 2nd new Pod ready |
| 6:10 | 3rd |
| 6:35 | 4th, rolling complete |
| 6:36 | Smoke test job in CI |
| 6:45 | Smoke tests pass |
| Deploy-prod: manual button |

**End-to-end**: ~7 минут для staging автоматически. Prod — плюс ручной approve.

---

## 11. Где что может сломаться

Список failure points на пути:

### 11.1 Git push
- **Authentication failed**: SSH key not registered.
- **Push rejected**: protected branch, no force push allowed.
- **Repository size exceeded**: git LFS не настроен, большие файлы.

### 11.2 CI pipeline
- **Runner unavailable**: нет свободных runner'ов, ждём в очереди.
- **Compile error**: код не собирается. Смотри build log.
- **Test failure**: смотри JUnit XML в artifacts.
- **Testcontainers timeout**: Docker daemon в runner проблема, DinD misconfig.
- **Cache miss**: первый build ветки медленный, все зависимости качаются.
- **Static analysis failure**: checkstyle / spotbugs finding.
- **Registry push failed**: auth, network, disk full.

### 11.3 ArgoCD sync
- **Sync failed**: манифест невалидный, admission webhook отверг.
- **Health check failed**: Deployment не становится healthy.
- **RBAC**: ArgoCD SA не имеет прав на namespace.

### 11.4 K8s deployment
- **ImagePullBackOff**: image не найдено или auth failed. Проверь `imagePullSecrets`.
- **CrashLoopBackOff**: приложение падает. Смотри `kubectl logs --previous`.
- **Readiness never passes**: startup probe timeout, БД недоступна, порт не тот.
- **OOMKilled**: memory limit слишком мал для JVM heap + metaspace.
- **Pending forever**: нет ресурсов, taints без tolerations, PVC не привязался.
- **Scheduling failure**: смотри `kubectl describe pod` → Events.

### 11.5 Rolling stuck
- `kubectl rollout status` показывает `Waiting for deployment "myapp" rollout to finish`.
- Причина: новые Pod'ы не становятся Ready → maxUnavailable не даёт убить старые → rollout завис.
- Debug: `kubectl get pods`, `kubectl describe`, `kubectl logs`.
- `progressDeadlineSeconds` истечёт (default 10 min) → Deployment marked failed, но НЕ откатывается автоматически.

Fix: manually `kubectl rollout undo` или fix новую версию и redeploy.

---

## 12. Собесные вопросы

**Q1: Что делает `git push`?**

Клиент устанавливает соединение с сервером (SSH или HTTPS), обменивается списком refs (что есть у клиента и сервера), формирует pack файл (delta-compressed objects), заливает pack на сервер и просит обновить ref (branch pointer) на новый SHA. Сервер выполняет hooks (pre-receive, update, post-receive), обновляет свою БД, триггерит webhook'и / CI.

**Q2: Что происходит между `git push` и запущенным Pod'ом в проде?**

1. Git server receives push, triggers webhook.
2. CI pipeline runs: build (compile, jar), test (unit, integration), security scan, docker build, docker push to registry.
3. CI updates manifests в git repo с новым image tag.
4. ArgoCD sees change, `kubectl apply`.
5. K8s Deployment controller creates new ReplicaSet.
6. ReplicaSet creates new Pod.
7. Scheduler assigns node.
8. Kubelet pulls image (или cache hit), creates container через CRI/containerd/runc.
9. Container starts, Spring Boot boots.
10. Readiness probe passes → Pod added to EndpointSlice.
11. Kube-proxy updates iptables на всех нодах.
12. Rolling update продолжается: старые Pod'ы удаляются один за другим.

Общее время: 5-15 минут для типичного микросервиса.

**Q3: Какие фазы у Maven?**

Три lifecycles: `clean`, `default`, `site`. Default имеет 23 phase, ключевые:
- `validate` — валидность проекта.
- `compile` — .java → .class (main).
- `test-compile` — .java → .class (test).
- `test` — unit tests (Surefire).
- `package` — создание JAR/WAR.
- `verify` — integration tests (Failsafe).
- `install` — копирование в ~/.m2/repository.
- `deploy` — upload в Nexus/Artifactory.

**Правило**: выполнение phase X = выполнение всех предыдущих phases. `mvn package` выполнит и compile, и test, и всё что между.

**Q4: Разница `mvn install` и `mvn deploy`?**

`install` — копирует artifact в **local repo** `~/.m2/repository/`. Доступно только тебе.  
`deploy` — upload в **remote repo** (Nexus / Artifactory / Maven Central). Доступно всей команде.

`install` часто в CI не нужен (проще `package` + push image). Deploy используется для libraries которые другие проекты подтягивают.

**Q5: Разница Phase и Goal в Maven?**

Phase — точка в lifecycle (например `compile`). Сама не делает работу.  
Goal — конкретное действие plugin'а (например `compiler:compile`).

Phases привязаны к goals через plugin bindings. `mvn compile` = execute phase compile = execute goal `compiler:compile` + все предыдущие phases.

Можно запустить goal напрямую: `mvn dependency:tree` — без всех предшествующих phases.

**Q6: Как работает Gradle task graph?**

Gradle configures ВСЕ задачи при старте (независимо от того что запросил пользователь). Каждая задача имеет `inputs`, `outputs`, `dependsOn`.

При `gradle build`:
1. Определяет task graph (DAG) — какие задачи нужны для `build`.
2. Топологический sort.
3. Для каждой задачи проверяет **up-to-date**: inputs не изменились → skip.
4. Выполняет в порядке.

Ключевые оптимизации: up-to-date check, build cache (local + remote), configuration cache, parallel execution.

**Q7: Что происходит когда Pod удаляется во время rolling update?**

1. Pod marked `Terminating` (deletionTimestamp set).
2. EndpointSlice controller **удаляет Pod IP** — Pod out of load balancing.
3. Kube-proxy на всех нодах обновляет iptables — новые запросы не идут.
4. Kubelet execute **preStop hook** (если настроен, например `sleep 10`).
5. Kubelet отправляет **SIGTERM** в main процесс контейнера.
6. Приложение делает **graceful shutdown**: stops accepting new requests, waits for in-flight to complete.
7. Container exits 0.
8. Если не exit'ит в `terminationGracePeriodSeconds` (default 30s) → **SIGKILL**.
9. Kubelet removes container, then pod sandbox.
10. apiserver окончательно удаляет Pod object.

Проблема **in-flight requests**: между "Pod удалён из endpoints" и "iptables обновились везде" — 1-5 сек. `preStop sleep 10` даёт окно.

**Q8: Что такое Docker layer caching и как его оптимизировать?**

Каждая `RUN` / `COPY` / `ADD` в Dockerfile = отдельный layer. Layer кэшируется по hash inputs. При rebuild — Docker сравнивает: если input не изменился → reuse cached layer.

Оптимизация:
- **Редко меняющееся раньше** в Dockerfile.
- Deps раньше кода.
- В Spring Boot: **layered JAR** — dependencies в отдельных слоях от application code.
- `.dockerignore` чтобы не invalidate cache при изменении irrelevant files.

Плохо:
```dockerfile
COPY . /app                    # весь source
RUN mvn package                 # invalidates при любом коде
```

Хорошо:
```dockerfile
COPY pom.xml /app/
RUN mvn dependency:go-offline   # deps в свой слой
COPY src /app/src
RUN mvn package                  # invalidates только при code change
```

**Q9: Что такое GitOps и почему CI не должен `kubectl apply`?**

GitOps — git repo с манифестами = единственный источник правды. Оператор в кластере (ArgoCD, Flux) синхронизирует cluster state с git.

**Почему CI не должен `kubectl apply`**:
- CI runner имеет **admin credentials** к кластеру. Атака на runner = уничтожение кластера.
- **Drift**: то что в git ≠ то что в кластере (кто-то `kubectl edit` руками).
- **Нет audit trail**: непонятно кто/когда deploy'ил.
- **Rollback сложнее** — не через git revert.

**GitOps way**:
- CI пушит **только image** в registry.
- CI обновляет **manifest в git** (kustomize edit set image).
- ArgoCD с read-only pull из git → apply в кластер.

Compromised CI runner может пушить images и manifests, но не имеет прямого доступа к кластеру.

**Q10: Как обеспечить zero-downtime rolling update?**

1. **Правильные probes**:
   - `readinessProbe` — на реальный health endpoint приложения.
   - `livenessProbe` — независим от external systems.
   - `startupProbe` для медленного Spring Boot старта.

2. **Rolling strategy**:
   ```yaml
   strategy:
     rollingUpdate:
       maxSurge: 1
       maxUnavailable: 0
   ```

3. **Graceful shutdown**:
   - Spring Boot `server.shutdown: graceful`.
   - `preStop` hook `sleep 10`.
   - `terminationGracePeriodSeconds: 45` > Spring timeout.

4. **PodDisruptionBudget** `minAvailable: replicas - 1`.

5. **Backward compatibility**: обе версии могут работать одновременно.

6. **HTTP client retry** на consumer side для idempotent operations.

**Q11: Что такое multi-stage Docker build?**

Dockerfile с несколькими `FROM` — каждый начинает новую stage. Только последняя стадия попадает в финальный image.

Использование: build tools (JDK, Maven, npm) — в builder stage. Runtime image — только с JRE + jar.

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
# ... build
FROM eclipse-temurin:21-jre
COPY --from=builder /app/target/*.jar app.jar
```

Итог: image 200MB вместо 800MB. Плюс security — build tools не в prod image (нет compilers, package managers).

**Q12: Почему ArgoCD раз в 3 минуты pull'ит git?**

**Polling** — надёжный fallback. Также ArgoCD может подписаться на **webhook** от GitLab/GitHub для мгновенного sync.

3 минуты = compromise между reactivity и load. Меньше — больше load на git server. Больше — deploy latency.

Webhook + polling — рекомендованный setup. Webhook даёт мгновенный sync 99% времени, polling ловит edge case.

**Q13: Какая разница между `docker build` и `bootBuildImage`?**

`docker build` — использует `Dockerfile` для explicit instructions. Полный контроль.

`bootBuildImage` (Spring Boot Gradle/Maven plugin) — использует **Cloud Native Buildpacks** (Paketo). Автоматически:
- Detect language, install JRE.
- Optimize layers.
- Set non-root user.
- Set JVM memory options based on container limits.
- Include health probe support.

Плюсы Buildpacks: security patches автоматически (rebuild → новый base image), best practices.  
Минусы: меньше контроля, learning curve.

Для новых проектов — попробовать Buildpacks. Для custom needs — Dockerfile.

**Q14: Что произойдёт если `terminationGracePeriodSeconds` меньше чем graceful shutdown timeout приложения?**

Kubelet отправляет SIGTERM, ждёт `terminationGracePeriodSeconds`, потом отправляет **SIGKILL**.

Если приложение всё ещё делает graceful shutdown (waiting for in-flight requests) — оно **прерывается насильно**. In-flight requests обрываются (клиенты видят connection reset). Транзакции в БД могут откатиться (или commit'нуться частично, если consumer уже commit'нул message).

Правило: `terminationGracePeriodSeconds` > `spring.lifecycle.timeout-per-shutdown-phase` + margin (5-10 sec).

**Q15: Как ArgoCD обнаруживает drift?**

ArgoCD в reconcile loop:
1. Читает git manifests (desired state).
2. Читает current state из cluster etcd (через apiserver).
3. Сравнивает field-by-field.
4. Показывает diff в UI.

С `selfHeal: true` — автоматически apply'ит git state поверх cluster (undo manual changes).

Без `selfHeal` — только показывает "OutOfSync", ждёт ручного sync.

---

## 13. Мини-чеклист

За 3-5 секунд:

- [ ] Git protocol: SSH/HTTPS, wire protocol v2, pack file.
- [ ] Server-side hooks: pre-receive / update / post-receive.
- [ ] Webhook → CI runner → job.
- [ ] CI stages: validate → build → test → package → deploy.
- [ ] Maven lifecycles: clean / default / site.
- [ ] Maven default phases: validate → compile → test → package → verify → install → deploy.
- [ ] Phase выполнение = все предыдущие phases.
- [ ] Phase vs Goal: phase — точка, goal — действие plugin'а.
- [ ] Gradle DAG задач, inputs/outputs, up-to-date checks.
- [ ] Gradle build cache (local + remote).
- [ ] Test pyramid: unit → integration → e2e.
- [ ] Testcontainers для integration tests.
- [ ] Multi-stage Dockerfile, layer caching, .dockerignore.
- [ ] Layered JAR для Spring Boot.
- [ ] docker push — только новые layers по digest.
- [ ] OCI registry API v2.
- [ ] CI НЕ должен `kubectl apply` — GitOps.
- [ ] Kustomize edit set image.
- [ ] ArgoCD polling + webhook.
- [ ] apiserver flow: auth → admission → etcd.
- [ ] Deployment controller → ReplicaSet controller → Pod → Scheduler → Kubelet.
- [ ] Kubelet: image pull → volumes mount → CNI setup → CRI create/start.
- [ ] runc → clone/unshare/pivot_root syscalls.
- [ ] Startup probe → Readiness probe → EndpointSlice update → kube-proxy iptables.
- [ ] Rolling update: maxSurge/maxUnavailable, terminationGracePeriodSeconds.
- [ ] Zero-downtime: preStop sleep, graceful shutdown, PDB.

---

## Итог

От `git push` до running Pod:

1. **Git**: pack file летит на сервер, ref updated, post-receive hook fires webhook.
2. **CI Runner**: pulls job, clones repo, executes stages.
3. **Build** (Maven phases / Gradle task graph): compile → test-compile → test → package.
4. **Docker build**: multi-stage, layer caching.
5. **Registry push**: только новые layers by digest.
6. **GitOps**: CI пушит manifest в git repo, ArgoCD видит, `kubectl apply`.
7. **K8s controllers**: Deployment → ReplicaSet → Pod. Scheduler assigns node.
8. **Kubelet**: image pull, volumes mount, CNI network, CRI create container → containerd → runc → syscalls.
9. **App startup**: JVM launch, Spring Boot context, DB migrations, Tomcat.
10. **Readiness passes** → EndpointSlice updated → kube-proxy propagates iptables → traffic flows.
11. **Rolling** continues Pod by Pod до всех новых.

Каждое звено может сломаться — умение debug'ить всю цепочку и есть senior-level DevOps mindset.

**Maven phases** — 23 фазы default lifecycle, ключевые validate → compile → test → package → verify → install → deploy. Phase = все предыдущие phases выполняются. Phase ≠ Goal (goal — конкретное действие plugin'а).

**Gradle** — DAG задач, incremental через up-to-date checks, build cache для reuse cross-machine.

Дальше в теме: практика — прочитай `.gitlab-ci.yml` какого-то реального проекта, разбери каждый stage, потрейси один pipeline от начала до конца. Всё встанет на место.
