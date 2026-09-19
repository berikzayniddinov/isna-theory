# 81. Build vs Deployment — глубокая теория + все виды деплоя

Файл про фундаментальную разницу между **build** (сборкой) и **deployment** (деплоем), про артефакты, окружения, registries — и **все виды деплойментов** с детальным разбором каждого (Rolling, Recreate, Blue-Green, Canary, A/B, Shadow, Dark launch, Ring, Big Bang).

Связано с: `03-gradle-detailed.md` (build система), `04-jar-fatjar-detailed.md` (артефакты), `09-docker-detailed.md` (Docker), `58-helm-helmsman.md` (Helm), `79-cicd-deploy-patterns.md` (CI/CD, GitOps — там уже есть детали canary/blue-green, здесь тоже, но с другого угла), `80-kubernetes-internals-interview.md` (внутренности K8s).

---

## 0. Ментальная модель: три разных этапа

**Build ≠ Release ≠ Deploy.** Это три отдельных концепции. Смешивание — источник половины проблем в CI/CD.

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  SOURCE  │────►│  BUILD   │────►│ RELEASE  │────►│  DEPLOY  │
│  (git)   │     │(artifact)│     │(bundle+  │     │(run in   │
│          │     │          │     │ config)  │     │env)      │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
   commit          .jar              docker image     Pod running
   diff            .class             tag :v1.0.42     на кластере
                   deps                                 в проде
```

**Build** — код (source) превращается в бинарный артефакт (JAR, image). Одинаковый source → одинаковый artifact (реproducible).

**Release** — artifact + configuration (для конкретной среды) → релизный bundle. `myapp:1.0.42` + `application-prod.yml` = `release-prod-1.0.42`.

**Deploy** — активация release в среде: запуск процессов, направление трафика.

Это модель **12-factor app**. Смысл разделения:
- Build → immutable. Один раз собрал → артефакт неизменен.
- Release → immutable. Собранный конфиг + артефакт → неизменен.
- Deploy → эфемерен. Можно перезапустить много раз с тем же release.

**Практическое следствие**: `myapp:1.0.42` собирается **один раз**. Один и тот же образ идёт в dev → staging → prod. Разница — только configuration (env vars, ConfigMap, Secret). Если для каждой среды собираешь свой образ — ты **не** практикуешь 12-factor и потерял бонус «то что тестировали — то и в проде».

---

## 1. Что такое BUILD

### 1.1 Определение

Build — процесс трансформации **исходного кода** в **исполняемый артефакт**, готовый к запуску. Для Java это:

```
.java files                          Compiled classes
+                                    +
dependency descriptors (pom/gradle)  Resolved deps
+                                ─►  +
resources (yml, xml, sql)            Resources packaged
+                                    +
build config (Gradle scripts)        Metadata (MANIFEST.MF)
                                     
                    ↓ packaging (jar/war/native)
                                     
                                     app.jar
                                     (single deployable file)
```

### 1.2 Что реально делает Gradle/Maven на `build`

1. **Dependency resolution**: читаем `build.gradle`, ходим в Maven Central / Nexus, скачиваем транзитивные зависимости, разрешаем конфликты версий (Gradle — highest-wins; Maven — nearest-wins).
2. **Compile Java**: `.java` → `.class` (bytecode). Отдельный шаг `compileJava` и `compileTestJava`.
3. **Process resources**: копирование `src/main/resources` → `build/resources/main`, подстановка placeholder'ов.
4. **Run tests**: `test` task. Собирает test-classpath, запускает JUnit.
5. **Static analysis** (если настроено): checkstyle, spotbugs, PMD, jacoco coverage.
6. **Packaging**: JAR/WAR/native image. Spring Boot `bootJar` собирает fat JAR со всеми зависимостями внутри.
7. **Publishing** (опционально): выгрузка в Nexus/Artifactory.

### 1.3 Артефакты Java-мира

- **`.class`** — один скомпилированный класс. Хранит bytecode + constant pool + метаданные.
- **JAR** — обычный: `.class`-файлы + `META-INF/MANIFEST.MF` + resources. Как zip. Используется как **library** (упаковка библиотеки).
- **Fat JAR / Uber JAR** — JAR со всеми зависимостями внутри, самодостаточный. Запускается `java -jar app.jar`. Spring Boot делает это через `bootJar`.
- **WAR** — Web Application Archive. JAR с фиксированной структурой (`WEB-INF/classes`, `WEB-INF/lib`, `WEB-INF/web.xml`). Разворачивается в внешний Tomcat/Jetty. Устаревает — теперь fat JAR.
- **EAR** — Enterprise Archive для JavaEE серверов приложений (WebSphere, WebLogic). Практически мёртв, только legacy.
- **Native Image** (GraalVM) — прекомпилированный AOT-бинарник. Стартует за десятки мс, но без reflection/dynamic classloading без специальных подсказок.

### 1.4 Fat JAR — что внутри

Spring Boot fat JAR:
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
│   │   ├── spring-webmvc-6.1.0.jar
│   │   └── ... (150+ файлов)
│   └── classpath.idx
└── org/springframework/boot/loader/  ← Boot launcher
    └── JarLauncher.class
```

При `java -jar myapp.jar` — стандартный JVM launcher находит `Main-Class: JarLauncher` → тот кастомно загружает classpath из `BOOT-INF/lib/` (обычный JVM classloader не умеет читать JAR-внутри-JAR).

Детали: `04-jar-fatjar-detailed.md`.

### 1.5 Reproducible builds

**Проблема**: один и тот же commit собранный дважды даёт разные бинарники. Причины:
- Таймстампы в файлах (`Last-Modified` в MANIFEST).
- Порядок файлов в архиве (файловая система zip'а).
- `Build-Time` метаданные.
- Локаль/timezone build-хоста.

**Зачем нужно**: security (можно проверить: то что в проде = то что собрано из commit X). Bit-for-bit сравнение → доказательство отсутствия supply chain injection.

**Как достичь**:
- Gradle: `-Dorg.gradle.reproducible-archives=true`.
- Установить `SOURCE_DATE_EPOCH` env var.
- Убрать `Build-Time` из MANIFEST.
- Использовать jar/zip normalization.

### 1.6 Build caching

**Локальный** (Gradle): `~/.gradle/caches`. Task output кэшируется по hash inputs. Второй `gradle build` без изменений — секунды.

**Remote build cache**: команда шарит cache через HTTP. Gradle Enterprise / Bazel remote cache / self-hosted. Если один разработчик собрал task X — другие переиспользуют.

**CI cache**: `.gradle/caches` сохраняется между runs (GitLab CI `cache:` секция, GitHub Actions `actions/cache`).

**Docker layer cache**: каждая `RUN`/`COPY` — слой. Слой кэшируется по hash содержимого. Изменил один Java-файл → invalidate только слой `COPY target/*.jar` (если построен правильно).

Правило Docker cache для Spring Boot:
```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
# Сначала — зависимости (редко меняются)
COPY build/libs/dependencies/ ./
COPY build/libs/spring-boot-loader/ ./
COPY build/libs/snapshot-dependencies/ ./
# Потом — свой код (часто меняется)
COPY build/libs/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

Это делает `bootBuildImage` из Spring Boot автоматически через **Cloud Native Buildpacks**.

### 1.7 Build vs Compile vs Package

Не путай:
- **Compile** — только `.java → .class`. Часть build'а.
- **Package** — упаковка `.class` в JAR. Тоже часть build'а.
- **Build** — весь жизненный цикл: resolve deps → compile → test → package → publish.
- **Assemble** — Gradle-специфичный термин: build без tests. `gradle assemble` собирает JAR, но не гоняет тесты.

### 1.8 Собесное

**Q: Зачем `gradle assemble` вместо `gradle build`?**  
Assemble — только сборка, без тестов. Быстрее для CI-стадий где тесты в отдельном stage/job.

**Q: Что такое "fat JAR" и почему нельзя просто взять regular JAR?**  
Regular JAR содержит только классы этого модуля. Deps в classpath (`java -cp` или Maven local repo). При запуске на другой машине — нужен весь classpath отдельно. Fat JAR — самодостаточен, для деплоя проще (один файл — один сервис).

**Q: Что делает `bootJar` в отличии от `jar`?**  
Boot's `bootJar` собирает Spring Boot fat JAR с их custom loader'ом. Обычный Gradle `jar` — thin JAR (только твой код). Обычно оба enabled в Boot-проекте, но `bootJar` — тот что заливаешь.

---

## 2. Что такое DEPLOY

### 2.1 Определение

Deploy — процесс **размещения** и **активации** артефакта в целевой среде. Три подзадачи:

1. **Provisioning** — подготовить где будет работать: сервер / VM / Pod / контейнер.
2. **Distribution** — доставить артефакт туда.
3. **Activation** — запустить, направить трафик, обновить DNS/LB.

### 2.2 Уровни абстракции deployment

```
[Bare metal]  → scp jar, systemd, руками. 1990s.
     ↓
[VM] → Terraform + Ansible. Каждая VM — снежинка. Immutable через AMI (packer).
     ↓
[Container] → Docker run на VM. Артефакт = image, не JAR.
     ↓
[Orchestrator (K8s)] → declarative, самолечится, автоскейл. 2020s стандарт.
     ↓
[Serverless] → отдал функцию, платформа сама всё. AWS Lambda, Cloud Functions.
     ↓
[Managed platforms] → Heroku, Cloud Run, Vercel. Git push → running.
```

Каждый следующий уровень абстракции = меньше твоего контроля, больше автоматизации.

### 2.3 Deployable unit — что реально деплоится

- **Bare metal / VM**: JAR + systemd unit + environment variables.
- **Container**: Docker image (`myapp:1.0.42`).
- **K8s**: манифесты (Deployment YAML + ConfigMap + Service). Image — часть.
- **Helm**: chart + values.yaml. Deploy = `helm install`.
- **Kustomize**: base + overlays. Deploy = `kubectl apply -k overlay/prod`.
- **Serverless**: zip с кодом + manifest.
- **Cloud Native Buildpacks**: source → готовый OCI image без Dockerfile.

### 2.4 Immutable vs mutable deployment

**Mutable** (устарел):
- SSH на сервер → скопировал новый JAR → перезапустил systemd.
- «Обновление на месте». Стейт сервера меняется, накапливается drift.
- Хорошо работает пока хорошо. Плохо — когда 30 серверов и один непонятно чем от других отличается.

**Immutable**:
- Собрал новый image → развернул новую VM/Pod из этого image → убил старую.
- Никаких «обновлений на месте», никаких SSH.
- Стейт сервера = image. Одинаковые images → одинаковое поведение.
- Идея из cattle-vs-pets: сервера — крупный рогатый скот, не домашние животные. Не лечишь — заменяешь.

K8s — immutable by design. Не можешь «редактировать Pod», можешь только заменить.

### 2.5 Что происходит при K8s deploy — детально

Уже разбирали в `80-kubernetes-internals-interview.md`. Кратко ещё раз:

```
kubectl apply -f deployment.yaml
     ↓
apiserver: auth + admission + etcd write
     ↓
Deployment controller видит change → создаёт новый ReplicaSet
     ↓
ReplicaSet controller создаёт Pod'ы
     ↓
Scheduler назначает nodeName
     ↓
Kubelet на ноде через CRI (containerd) через OCI runtime (runc)
     ↓
Linux syscalls: clone(), setns(), unshare() → создан контейнер
     ↓
Приложение запускается, readiness становится true
     ↓
EndpointSlice controller добавляет Pod IP в endpoints
     ↓
Kube-proxy обновляет iptables на всех нодах
     ↓
Трафик идёт в новый Pod
```

Каждое звено может сломаться — отсюда все виды `kubectl describe pod` errors.

---

## 3. Три оси классификации деплоя

Виды/стратегии деплоя часто мешают в одну кучу. Раздели по осям:

### Ось 1: Как заменяем старую версию новой

- **In-place** — обновление старой версии на месте (тот же процесс/контейнер).
- **Recreate** — снести всё → поднять всё новое.
- **Rolling** — постепенно (по одному) заменяем.
- **Blue-Green** — параллельно две среды, свитч трафика.
- **Canary** — постепенно перенаправляем трафик 5% → 20% → 100%.

### Ось 2: Кто видит новую версию

- **Big Bang** — все пользователи разом.
- **Ring / Wave** — сначала внутренние, потом beta-users, потом всех.
- **A/B Testing** — часть пользователей на v1, часть на v2 (по criteria).
- **Feature flags** — фича в проде, но за флагом.
- **Dark Launch** — код в проде, но не выключен для пользователей (только internal test).

### Ось 3: Куда идёт реальный трафик

- **Live traffic** — реальные пользователи попадают в новую версию.
- **Shadow / Mirror** — реальный трафик копируется в новую версию, но ответ не показывается.
- **Synthetic** — только искусственные запросы (smoke test бота).

Один реальный deploy обычно **комбинирует** эти оси. Например: «canary + feature flag + shadow copy для 10% пользователей».

---

## 4. Виды деплоя — детальный разбор

Дальше — каждый вид развёрнуто: механика, плюсы/минусы, когда использовать, как реализовать в K8s.

### 4.1 In-Place (обновление на месте)

**Что**: тот же процесс продолжает работать, но с новой версией кода. Не в контейнерах — из мира classic-серверов.

Как: подложил новый JAR/WAR → сервер увидел (hot deploy Tomcat / OSGi bundle reload) → продолжил без рестарта.

**Плюсы**: нет даунтайма, нет пере-инициализации.  
**Минусы**: classloader hell, memory leaks (permgen/metaspace), state inconsistency. **Ненадёжно.**

Практически: **не используется в prod с контейнерной эры**. Legacy WebSphere/WebLogic для внутренних систем, где не могут остановиться. Наследие 2000-х.

В K8s — невозможно (immutable containers). Ближайший аналог — Spring Boot DevTools с auto-reload, но только для локальной разработки.

### 4.2 Recreate (пересоздание, big bang)

**Что**: убей все инстансы старой версии → подними все инстансы новой. Downtime гарантирован (секунды-минуты).

Схема:
```
Time →
  Old: [v1][v1][v1][v1]
  Off:               ↓
                    [__][__][__][__]  ← downtime
  New:                                ↑
                                     [v2][v2][v2][v2]
```

**K8s config**:
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: Recreate
```

При `kubectl apply` с изменённым template — все старые Pod'ы удаляются одновременно, потом создаются новые.

**Плюсы**:
- Простой (один state в моменте, не надо думать о совместимости версий).
- Освобождает ресурсы (не нужны `maxSurge` мощности).
- Проще с несовместимыми миграциями БД (нет момента когда старая и новая работают).

**Минусы**:
- Downtime — минимум 30-60 секунд (Boot startup).
- Нет постепенной проверки — прод падает или прод жив.
- Клиенты получают 5xx во время окна.

**Когда используется**:
- Dev / staging.
- Batch-приложения (не HTTP-запросы, всё равно нет пользователей).
- Приложения с эксклюзивными ресурсами: только один инстанс может держать распределённый lock на файл/legacy систему.
- Ranchers (см. Streamlit / Dash internal apps) где 1 replica ok.
- Внутренние админ-панели с плановым окном.

**Никогда**: user-facing prod без planned downtime.

### 4.3 Rolling Update — стандарт

**Что**: постепенная замена. Один или несколько Pod'ов за раз: сначала новый поднимается, потом старый убивается. Всегда есть работающие Pod'ы.

Схема (replicas=4, maxSurge=1, maxUnavailable=0):
```
Step 0:  [v1][v1][v1][v1]                              4 v1 ready
Step 1:  [v1][v1][v1][v1][v2]      ← поднимаем v2      4 v1 + 1 v2 starting
Step 2:  [v1][v1][v1]    [v2]      ← убили один v1     3 v1 + 1 v2 ready
Step 3:  [v1][v1][v1][v2][v2]      ← ещё один v2       3 v1 + 1 v2 + 1 v2 starting
Step 4:  [v1][v1]    [v2][v2]      ← и т.д.
...
Step N:                [v2][v2][v2][v2]                 4 v2 ready
```

**K8s config** (по умолчанию для Deployment):
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%             # можно поднять сверху нормы
      maxUnavailable: 25%       # можно недоступно
```

**Zero-downtime настройка** (prod):
```yaml
rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0             # никогда не убиваем без нового Ready
```

**Плюсы**:
- Zero downtime (если правильно настроено).
- Постепенная обкатка новой версии — если что-то ломается на первых Pod'ах, rollout stuck и старая версия ещё работает.
- Встроено в K8s. Ничего дополнительного.
- Экономно по ресурсам (максимум `replicas + maxSurge`).

**Минусы**:
- **Обе версии работают одновременно** — обязательна backward compatibility (API, БД схема, message contracts).
- Долго — при 20 replicas и 30-секундном readiness это 10+ минут.
- Rollback тоже rolling (медленный).
- Нельзя сразу переключить 100%.

**Rollback**: `kubectl rollout undo deployment/X` — тоже rolling, обратный процесс.

**Kubernetes нюансы rolling update**:

1. **maxSurge** может быть числом или процентом. `25%` = ceil(0.25 × replicas).
2. **maxUnavailable** тоже. Оба нулём быть не могут (deploy никогда не сдвинется).
3. **progressDeadlineSeconds** (default 600s) — если новый ReplicaSet не стабилизировался за это время → Deployment marked `Progressing=False`. **Автоматического rollback нет**, нужно вручную.
4. **minReadySeconds** — сколько секунд Pod должен быть Ready перед тем как считаться «прошедшим стадию». Полезно если приложение стартует, но нестабильно первые секунды.
5. Rolling update контролируется через **ReplicaSet**: старый RS скейлится вниз, новый — вверх.

**Проблема ROlling update — DB миграции**. Разберём отдельно (см. §7).

**Когда использовать**: **99% prod-случаев**. Дефолт.

### 4.4 Blue-Green

**Что**: две параллельные среды. Blue — текущий prod. Green — новая версия. Оба полностью развёрнуты. Свитчинг трафика — атомарный.

Схема:
```
  Load Balancer / Service
         │
    ┌────┴────┐
    │         │
   BLUE      GREEN
   v1.5      v1.6
   4 pods    4 pods
   (active)  (idle/testing)

  ── deploy step ──►
  
    ┌────┴────┐
    │         │
   BLUE      GREEN
   v1.5      v1.6
   4 pods    4 pods
   (idle/    (active — flip!)
   rollback)  
```

**Реализация в K8s** — через label selector Service:

```yaml
# два Deployment'а
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
# Один Service, селектит по version
apiVersion: v1
kind: Service
metadata: {name: myapp}
spec:
  selector:
    app: myapp
    version: blue     # ← меняешь на green, чтобы переключить
  ports: [{port: 80, targetPort: 8080}]
```

Deploy:
1. `myapp-blue` работает на v1.5, Service указывает на `blue`.
2. Создать `myapp-green` с v1.6, 4 replicas. Стоит рядом, трафик не идёт (Service не селектит).
3. **Smoke testing на green** — direct через Pod IP / отдельный dev Service.
4. `kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'`
5. Через 1-5 секунд kube-proxy обновляет iptables → весь трафик идёт в green.
6. Blue остаётся 15-60 минут как **rollback target**. Если проблема → флип обратно за секунду.
7. Убираем blue, следующий deploy: green → blue (либо всегда blue активный, green — новая версия; либо чередуем).

**Плюсы**:
- **Мгновенный rollback** — секунда, не десятки минут rolling.
- Полный test новой версии перед экспозицией.
- Нет «обе версии одновременно» — либо всё blue, либо всё green.
- Проще для несовместимых изменений (одномоментный switch).

**Минусы**:
- **2× ресурсов** во время deploy (два полных стека).
- **In-flight соединения**: при свитче — активные HTTP-keep-alive или long-polling соединения могут остаться на blue. Нужно правильное draining.
- **БД проблема остаётся**: обе версии подключены к той же БД. Миграция должна быть compatible.
- Сложнее реализовать (два Deployment'а, кастомный workflow).

**Когда использовать**:
- Критичные системы где 5xx на 30 секунд неприемлемы.
- Когда нужен мгновенный rollback (финтех, health, ecommerce peak).
- Когда есть бюджет на 2× ресурсов.

**Tools**: **Argo Rollouts** делает это через один CRD:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    blueGreen:
      activeService: myapp
      previewService: myapp-preview     # для тестирования
      autoPromotionEnabled: false       # ждать ручного promote
      scaleDownDelaySeconds: 3600       # держать старое 1 час
```

### 4.5 Canary (канареечный)

**Что**: **маленькая часть трафика** идёт на новую версию, наблюдение → увеличение процента постепенно. Название — от канарейки в шахте (шахтёры брали птицу под землю: если газ утёк — птица умрёт первой, шахтёры узнают).

Схема:
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

**Реализация без mesh — через replicas ratio**:

Один Service селектит по `app: myapp` (без version). Два Deployment'а:
- `myapp-stable` — 19 replicas, image v1.5.
- `myapp-canary` — 1 replica, image v1.6.

Один общий label `app: myapp` → Service селектит все 20 Pod'ов. Балансировка round-robin/random → 1 из 20 → 5% трафика на canary.

Минус: гранулярность = 1/replicas. Хочешь 1% на canary при 20 replicas — не получится, нужно 100 replicas. И трафик не reliable 5% — statistical (Pod может получить больше/меньше).

**Реализация с service mesh (Istio)**:
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

Точный процент. Плюс можно роутить по headers / cookies / geolocation:
```yaml
http:
- match:
  - headers:
      x-user-tier: {exact: beta}
  route:
  - destination: {host: myapp, subset: canary}
- route:                          # остальные
  - destination: {host: myapp, subset: stable}
    weight: 100
```

**Реализация через Argo Rollouts** (стандартный prod-выбор):
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

**AnalysisTemplate**:
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

При каждой паузе Argo Rollouts опрашивает Prometheus. Success rate ≥ 99% → следующий weight. Меньше → **автоматический rollback**.

**Плюсы**:
- **Реальный трафик** — не искусственные тесты.
- **Минимальный blast radius** — если новая версия сломана, страдают 5% пользователей, не 100%.
- Автоматическая остановка при деградации.
- Идеально для рискованных изменений.

**Минусы**:
- **Долго** — 30 минут — часы. Не подходит для срочных фиксов.
- Обе версии работают одновременно (compatibility).
- Требуется observability (без метрик canary — просто rolling с задержкой).
- Сложность (Istio setup, Argo Rollouts, analysis templates).

**Когда использовать**:
- Крупные критичные системы (миллионы пользователей).
- Экспериментальные фичи.
- ML-моделей deploy — validated on subset.

**Combined с A/B**: canary — по %, A/B — по criteria. Комбинируется: 5% на canary И только beta-tier users.

### 4.6 A/B Testing

**Что**: разные пользователи получают разные версии в течение продлённого времени. Цель — не rollout, а **сравнение бизнес-поведения** (conversion rate, time-on-site).

Отличие от canary:
- **Canary** — временный, цель — safe rollout. Все в итоге получат новую.
- **A/B** — постоянный (недели-месяцы), цель — измерить какая версия лучше. Может кончиться выбором варианта или разделением на сегменты.

Роутинг по criteria:
- User ID (hash → bucket A или B).
- Geolocation.
- Device (mobile vs desktop).
- Random assignment сохранённый в cookie.

**Реализация**: обычно через **feature flag сервис** (LaunchDarkly, Unleash) внутри приложения:
```java
if (unleash.isEnabled("new-checkout", context)) {
    return newCheckoutFlow();
} else {
    return oldCheckoutFlow();
}
```

Один deployment, один Pod, вся логика — в приложении. Метрики (conversion) — считаются по группам A/B.

**Или**: два Deployment'а + Istio routing по header'у.

**Плюсы**:
- Data-driven решения о продукте.
- Не про технику деплоя — про бизнес-эксперименты.

**Минусы**:
- Долгий цикл (2-4 недели минимум для статистической значимости).
- Сложность анализа.
- Загрязняет кодовую базу (обе ветки логики живут).

**A/B testing — это НЕ deployment strategy** в чистом виде. Это technique для рискованных фич + продуктовых экспериментов. Часто путают.

### 4.7 Shadow / Traffic Mirroring

**Что**: реальный live трафик **копируется** в новую версию, но её ответ **не возвращается клиенту**. Клиент получает ответ старой версии.

Схема:
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

**Реализация в Istio**:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata: {name: myapp}
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

50% всего трафика **дублируется** в shadow. Пользователи видят только stable, но shadow обрабатывает половину нагрузки.

**Плюсы**:
- **Zero risk** для пользователей — они не видят shadow ответы.
- Реальная нагрузка на новую версию — проверка производительности.
- Сравнение ответов: логируешь ответы обоих версий, сравниваешь offline.
- Идеально для рефакторинга: убеждаешься что новая имплементация даёт те же ответы что старая.

**Минусы**:
- **Side effects проблема**: shadow-запросы делают реальные writes → двойная запись в БД. Нужно либо:
  - Shadow работает read-only (специальный mode).
  - Отдельная БД для shadow (репликация).
  - Feature flag внутри приложения: «если shadow request — skip DB write».
- **Overhead** — двойная нагрузка на систему.
- Не подходит для API с side effects (payments, order creation) без изоляции.

**Когда использовать**:
- Read-heavy сервисы (search, recommendations).
- Рефакторинг существующего сервиса без изменения API.
- Load testing на реальном трафике перед canary.
- ML models — сравнить предсказания.

### 4.8 Dark Launch

**Что**: код **уже в проде**, но **невидим пользователю**. Активация — через feature flag, без нового деплоя.

Отличие от feature flag: dark launch — специально про то, что **вся инфраструктура готова**, но UI/фича не показана. Feature flag — общий термин.

Схема:
```
Neделя 1: deploy v1.6 с новой функциональностью за флагом OFF
         (flag=false → dead code, но всё работает)
         ─── PROD OK, никто не видит фичу ──►

Неделя 2: включаем flag для internal users (test accounts)
         ─── команда проверяет ──►

Неделя 3: включаем для 1% реальных пользователей
         ─── мониторим ──►

Неделя 4: 100%
```

**Реализация**:
```java
@Value("${feature.new-payment.enabled:false}")
private boolean newPaymentEnabled;

@Value("${feature.new-payment.beta-users:}")
private Set<String> betaUsers;

@PostMapping("/pay")
public PaymentResult pay(PaymentRequest req, @AuthPrincipal User user) {
    boolean useNew = newPaymentEnabled 
        || betaUsers.contains(user.getId());
    if (useNew) {
        return newPaymentGateway.charge(req);
    }
    return oldPaymentGateway.charge(req);
}
```

Флаг из ConfigMap / Consul / feature flag сервиса. Изменение флага — без нового деплоя.

**Плюсы**:
- Разделение **deploy** и **release** (12-factor III → но эта третья dimension — release timing).
- Deploy в пятницу можно, release в понедельник.
- Instant rollback (флаг OFF, не redeploy).
- Скрытая доработка — код в проде тестируется тихо.

**Минусы**:
- Dead code накапливается (если флаг не убирают после rollout).
- Тесты усложняются (матрица флагов).
- Логика if/else в коде.

**Anti-patterns**:
- Флаги никогда не выключаются, накапливается 200 «временных» флагов.
- Каждая фича за флагом (перебор).

**Правило**: каждый флаг имеет **expiration date**. Через 3 месяца после enable для всех — код старой ветки удаляется.

Подробно про feature flags — в `79-cicd-deploy-patterns.md §5`.

### 4.9 Ring-based / Wave deployment

**Что**: несколько «колец» пользователей, deploy идёт постепенно от кольца к кольцу. Каждое кольцо — категория аудитории.

Пример (Microsoft Windows Insider программа):
```
Ring 0: internal Microsoft (десятки инженеров)      ← первая неделя
Ring 1: Fast Ring (миллион insiders)                ← вторая неделя
Ring 2: Slow Ring (10 миллионов beta users)         ← через месяц
Ring 3: Release Preview                             ← через 2 месяца
Ring 4: General Availability (миллиард пользователей)  ← через 3 месяца
```

Каждое кольцо — время наблюдения. Если проблема на ring 1 — не докатывается до ring 2.

**Реализация**: по сути **canary с явными группами** вместо процентов. Через feature flag сервис + user attributes:
- Employee → Ring 0.
- Beta signup → Ring 1.
- Country: US → Ring 2 (сначала америка).
- Rest → Ring 3.

**Плюсы**:
- Каждое кольцо — контролируемое окно feedback'а.
- Разные SLA (внутренние толерантнее к багам).
- Регуляторная compliance (некоторые регионы обязаны получить позже).

**Минусы**:
- Долго (месяцы).
- Обе версии работают месяцами — сложно поддерживать compatibility.
- Управление сложное.

**Когда**: крупный ecosystem (Windows, Chrome, iOS). Не для внутренних сервисов.

### 4.10 Progressive Delivery — обобщение

**Progressive delivery** = canary + автоматизация + метрики + automation. Термин от Weaveworks (2018).

Не отдельная стратегия, а **зонтик над canary/blue-green с автоматизацией**:
- Автоматическое продвижение по этапам.
- Автоматический rollback по метрикам.
- Attention-based promotion (не по времени, а по достижению KPI).

Реализация — Argo Rollouts, Flagger. Уже разбирались.

---

## 5. Deployment topology — где реально бегут артефакты

### 5.1 Bare metal

Физический сервер. `scp app.jar user@server:/opt/app/`, `systemctl restart app`.

**Плюсы**: полный контроль, минимум overhead.  
**Минусы**: снежинки (каждый сервер уникален), сложный provisioning, дорогой scaling (заказать железо → недели).

Сейчас — только legacy или очень performance-sensitive (HFT, gaming).

### 5.2 VM

Виртуальная машина в облаке / on-prem гипервизоре. То же что bare metal, но virtualized. Provisioning — Terraform + Ansible / Packer.

**Immutable VM approach**: Packer build AMI/image → new VMs from image → drop old ones. Cattle не pets.

**Auto-scaling groups** (AWS ASG, GCP MIG, Azure VMSS) — управляемое масштабирование.

Всё ещё много legacy prod'ов на VM. Java-приложение работает так же, как в контейнере, но управление тяжелее.

### 5.3 Container

Docker (или другой) на VM. Артефакт — image, не JAR.

**Плюсы vs VM**:
- Легче (нет виртуализации ядра).
- Быстрый старт (секунды vs минуты).
- Reproducible (image = состояние).
- Хорошая изоляция без overhead.

**Docker daemon на VM** + `docker-compose` — простой setup для мелких проектов. Не production-grade для scale.

### 5.4 Kubernetes

Стандарт для контейнерных workloads на 2026. Даёт:
- Orchestration (кто где бежит).
- Self-healing (Pod упал → перезапуск).
- Service discovery.
- Rolling updates встроены.
- Autoscaling.

Complexity: cluster нужно поддерживать. **Managed K8s** (EKS, GKE, AKS) убирает control plane заботы.

### 5.5 Serverless / FaaS

AWS Lambda, GCP Cloud Functions, Azure Functions. Deploy — загружаешь код (zip). Провайдер сам управляет запуском.

**Модель**: pay-per-invocation. Нет running instance между запросами (cold start).

Java на Lambda — исторически медленно (cold start 2-10s). Улучшается с SnapStart (AWS, 2022) — snapshot JVM state, старт 100-500ms.

**Когда**: event-driven (Lambda triggered by S3 upload, SQS message, HTTP через API Gateway). Не для long-running.

### 5.6 Managed platforms

Heroku, Fly.io, Railway, Render, Google Cloud Run, AWS App Runner, Vercel (для web). Уровень выше K8s.

Deploy = `git push` или `platform deploy`. Никаких манифестов, dockerfile'ов чаще всего не нужен (Buildpacks делают image из source).

**Плюсы**: скорость, DX.  
**Минусы**: vendor lock-in, дороже per-request на scale.

**Cloud Run** — интересный компромисс: managed serverless container platform. Загружаешь docker image, платформа сама масштабирует 0 → N. Всё-таки контейнер (не FaaS), но без K8s complexity.

---

## 6. Артефакты и registries

### 6.1 Артефакты по слою

- **Source**: git repo, commit SHA.
- **Compiled**: `.class` files, native binary.
- **Package (Java)**: JAR, WAR, EAR.
- **Container**: OCI image.
- **Chart (K8s package)**: Helm chart tarball.
- **Deploy manifest**: raw YAML или Kustomize base+overlay.
- **Release bundle**: image + config + manifests, tagged with version.

### 6.2 Registries

**Java/JVM artifacts**:
- **Maven Central** — публичный default для Java deps.
- **Nexus** (Sonatype) — self-hosted, стандарт для enterprise. Прокси к Maven Central + private artifacts. КНП: `nexus.isna`.
- **Artifactory** (JFrog) — коммерческий Nexus.
- **GitHub Packages / GitLab Packages** — интегрированные с их SCM.

**Container images (OCI)**:
- **Docker Hub** — публичный дефолт. Rate limits для анонимных пуллов.
- **Harbor** — open source enterprise registry с scanning, replication, RBAC. Часто on-prem.
- **AWS ECR**, **Google GCR / Artifact Registry**, **Azure ACR** — облачные.
- **GitHub Container Registry** (`ghcr.io`), **GitLab Registry**.
- **quay.io** — Red Hat.

**Helm charts**:
- **OCI registries** (Harbor, ECR, GCR) — стандарт с Helm 3.8+.
- **Artifact Hub** — публичный каталог.
- **ChartMuseum** — self-hosted (устаревающий подход).

### 6.3 Image tagging strategies

- **`:latest`** — HTML `<blink>`. Не используй. Каждый Pod может подтянуть разное.
- **Semantic version** (`:1.0.42`, `:v2.3.1`) — human readable, но иммутабельность зависит от дисциплины (кто-то может перепушить).
- **Git commit SHA** (`:abc123def`) — immutable by definition, идеально для CI. Проблема: не читаемо.
- **Комбинация** (`:1.0.42-abc123def`) — best of both.
- **Digest** (`@sha256:abc...`) — content-addressable, гарантированно immutable. Prod best practice.

Реальный prod deploy манифест указывает **digest**, не tag:
```yaml
image: registry.example.com/myapp@sha256:abcd1234...
```

Так гарантируешь что этот Pod запустит именно этот bit-for-bit образ, даже если tag был перепушен.

---

## 7. БД миграции при deploy — самая сложная часть

При Rolling / Blue-Green / Canary две версии кода **одновременно** обращаются к БД. Схема должна поддерживать **обе**.

### 7.1 Expand-Contract pattern

Всегда двухэтапный релиз при breaking изменении:

**Плохо (одним релизом)**:
- v1 читает `user.name`.
- Release: `RENAME COLUMN name TO full_name`.
- v2 читает `user.full_name`.
- Rolling: v1 (ещё живой) → читает `name` → column не существует → 500.

**Хорошо (expand → transition → contract)**:

**Release 1 (expand)**:
- Migration: `ADD COLUMN full_name; UPDATE users SET full_name = name;`
- v1 читает `name`.
- v2 (пока не задеплоена) читала бы `full_name`.
- Оба поля синхронизируются через триггер:
  ```sql
  CREATE TRIGGER sync_name_full_name BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION sync_name_columns();
  ```

**Release 2 (transition)**: deploy v2.
- v2 пишет и читает `full_name`.
- v1 всё ещё пишет `name` (триггер синхронизирует).
- Обе версии совместимы.

**Release 3 (contract, через 1-4 недели)**:
- Только v2 в проде уже.
- Migration: `DROP TRIGGER; DROP COLUMN name;`

Три релиза, недели времени. Но zero downtime.

### 7.2 Правила safe migrations

**Всегда безопасно**:
- `ADD COLUMN` с nullable / DEFAULT.
- `ADD TABLE`.
- `ADD INDEX CONCURRENTLY` (PostgreSQL).
- `ADD CHECK CONSTRAINT NOT VALID` + отдельный `VALIDATE`.

**Опасно**:
- `ADD COLUMN NOT NULL DEFAULT expr` — full table rewrite в старых PG (до 11).
- `DROP COLUMN` — если код ещё использует.
- `RENAME COLUMN / TABLE` — атомарно, но код не готов.
- `ALTER COLUMN TYPE` — часто lock + rewrite.
- `ADD INDEX` без CONCURRENTLY — эксклюзивный lock всей таблицы.

### 7.3 Migration как отдельный этап

Схема:
```
1. Run migration Job (Liquibase/Flyway) → schema updated
2. Rolling deploy new version → code updated
```

В K8s — через Argo CD **PreSync hook**:
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

Argo Sync: PreSync jobs → Sync (Deployment/Service) → PostSync.

Migration должна быть **backward-compatible** (см. §7.1) — чтобы старая версия работала до и во время rolling.

### 7.4 Rollback после несовместимой миграции

**Тяжёлая ситуация.** Deploy v2 → migration → в проде проблема → нужно на v1.

Варианты:
1. **Быстрая обратная миграция** (если возможно — reverse SQL). Не всегда.
2. **Hot fix** — v2.1 с исправлением, catch-forward.
3. **PIT restore из бэкапа** — потеря данных за время между backup и restore. Последнее средство.

**Правильно — не допускать**: expand-contract, никогда breaking migration в одном релизе.

---

## 8. Deployment environments

### 8.1 Каноническая пирамида

```
                    ┌───────────┐
                    │   PROD    │  real users
                    ├───────────┤
                    │ PRE-PROD  │  final validation
                    │(staging)  │  clone of prod
                    ├───────────┤
                    │    QA     │  QA team тесты
                    ├───────────┤
                    │    DEV    │  developer sandbox
                    └───────────┘
```

Продвижение (promotion) артефакта: dev → QA → staging → prod. Тот же image идёт через все стадии, конфиг меняется.

### 8.2 KNP-style

- `dev` — разработчики.
- `test` — QA.
- `release` (staging) — препрод, зеркало прод конфига.
- `prod` — production.
- Namespace: `knp`, `fno`, `fo`, `tax-report` (плюс `21` варианты для Java 21 версий).

Каждая — отдельный K8s namespace или отдельный кластер.

### 8.3 Environments как код

Все окружения описаны в git repo. Пример структуры (Kustomize):
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

`kubectl apply -k overlays/prod` → base + prod-специфичные патчи применяются.

Аналог в Helm: один chart + разные `values.yaml` per env:
```
helm install -f values-prod.yaml myapp ./chart
```

### 8.4 Ephemeral environments

Для каждого PR — свой временный env. `PR-1234` → namespace `pr-1234` с полным стеком.

Плюсы: изолированное тестирование, реалистичный интеграционный тест.  
Минусы: ресурсы, БД / stateful нужно clone'ить.

Автоматизация: ArgoCD ApplicationSet per branch, или GitHub Actions/GitLab CI создают namespace при PR открытии, удаляют при закрытии.

---

## 9. Deployment tooling landscape

### 9.1 CI (Continuous Integration)

Собирает и тестирует.
- **Jenkins** — Java, самый старый, plugin ecosystem огромный. Гибкий, но complex.
- **GitLab CI** — интегрирован с GitLab. `.gitlab-ci.yml`. Стандарт для GitLab пользователей.
- **GitHub Actions** — интегрирован с GitHub. YAML workflows. Огромный marketplace of actions.
- **CircleCI**, **Travis CI** — SaaS исторические.
- **Buildkite** — hybrid (agents on your infra, control на SaaS).
- **Tekton** — K8s-native CI на CRD.

### 9.2 CD — continuous delivery/deployment

**Continuous Delivery** — готово к deploy в любой момент, но кто-то жмёт кнопку.  
**Continuous Deployment** — автоматически в prod при merge в main.

Инструменты:
- **Argo CD** — GitOps для K8s. Watches git repo, syncs kubectl apply. Стандарт.
- **Flux** — тоже GitOps, без UI. Weave Works.
- **Spinnaker** — Netflix, мощный, для мульти-cloud, blue-green из коробки. Сложный.
- **Argo Rollouts** — расширение Argo для progressive delivery (canary, blue-green с metrics).
- **Octopus Deploy** — коммерческий, для .NET экосистемы часто.
- **Harness** — коммерческий, ML-based deploys.

### 9.3 Package managers для K8s

- **Helm** — templating + release management. Chart = template + values. `helm install` создаёт release. Стандарт де-факто.
- **Kustomize** — overlay-based, no templates (patches). Проще Helm, но менее мощно. Встроен в kubectl.
- **jsonnet / Tanka** — программируемая конфигурация (Grafana).
- **Pulumi**, **CDK8s** — configuration в TypeScript/Python/Go (не YAML).

Helm vs Kustomize:
- Kustomize — если тебе нужен просто patch (overrides для env).
- Helm — если нужны loops, conditionals, computed values, versioned releases.

### 9.4 Helmsman (KNP)

В КНП используется **Helmsman** поверх Helm — declarative describe of releases. Один файл описывает все установленные chart'ы:
```yaml
apps:
  isnaknpuser:
    namespace: knp
    chart: nexus/isnaknpuser
    version: 1.0.42
    valuesFile: ./values/prod/isnaknpuser.yaml
```

`helmsman apply` — синхронизирует с реальным кластером.

Плюс: declarative над imperative `helm install`. Минус: устаревающий (сейчас чаще GitOps через ArgoCD).

Детали: `58-helm-helmsman.md`.

---

## 10. Rollback strategies

### 10.1 Types of rollback

1. **kubectl rollout undo** — быстро, но НЕ через git. Argo CD в GitOps setup вернёт вперёд.
   ```bash
   kubectl rollout undo deployment/myapp
   kubectl rollout undo deployment/myapp --to-revision=3
   ```

2. **Git revert** (GitOps way):
   ```bash
   git revert HEAD
   git push
   # ArgoCD синхронизирует → rollback
   ```

3. **Redeploy previous version tag**:
   ```bash
   kustomize edit set image myapp=myapp:1.0.41  # предыдущий tag
   git commit -am "Rollback to 1.0.41"
   git push
   ```

4. **Blue-Green flip**:
   ```bash
   kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
   ```
   Секунды. Мгновенно.

5. **Feature flag off** — не rollback артефакта, а rollback фичи:
   ```yaml
   # ConfigMap: feature.new-payment.enabled=false
   ```
   Не нужен redeploy, флаг перечитывается через `@RefreshScope` / Consul.

### 10.2 Roll-forward vs Roll-back

**Roll-back** — вернуться к предыдущей версии.  
**Roll-forward** — исправить в новой версии, deploy v2.1 быстро.

Когда roll-forward:
- Проблема мелкая, знаешь фикс.
- БД мигрирована несовместимо (rollback невозможен).
- v1 тоже имеет проблему (просто другую).

Когда roll-back:
- Не знаешь причину, нужно время на диагностику.
- Есть stable известный good state.
- Проблема серьёзная (finance impact).

Первое, что делают senior teams при инциденте — **rollback first, debug later**. Меньше stress, больше времени.

### 10.3 Проблема необратимого deploy

Некоторые изменения не откатываются:
- Sent emails.
- Migration которая DROP COLUMN с потерей данных.
- External API calls (записали в чужую систему).
- Log messages, audit trails.

Из этого правило: **опасные изменения — за feature flag**, дай возможность instant rollback без redeploy.

---

## 11. Semver и версионирование

### 11.1 Semantic Versioning

`MAJOR.MINOR.PATCH` (`1.4.2`):
- **MAJOR** — breaking changes (API убрал/изменил). Клиенты должны адаптироваться.
- **MINOR** — новые фичи, backward compatible.
- **PATCH** — bug fixes only.

Extensions:
- `1.4.2-rc.1` — release candidate.
- `1.4.2-alpha.3` — alpha build.
- `1.4.2+build.abc123` — build metadata.

Semver — для **libraries** и **API contracts**. Для internal-сервисов часто overkill.

### 11.2 CalVer (Calendar Versioning)

`YYYY.MM.PATCH`: `2026.01.5` — фича января 2026, пятый хотфикс.

Плюсы: сразу видно возраст. Не надо думать «minor или major».  
Минусы: не показывает breaking changes.

Использует Ubuntu (`24.04`), JetBrains IDE.

### 11.3 KNP style

Из повседневной работы КНП — тикет-based:
- `release-ISNA2-23651-...` — ветка от задачи.
- `ISNA2-23651: описание` — commit сообщение.
- Deployment описывается как «релиз ветки ISNA2-XXXXX».

Semver номер обычно не используется — фиксируется тикет.

---

## 12. Собесные вопросы с ответами

### Q1: В чём разница между Build и Deployment?

**Build** — процесс превращения source code в исполняемый артефакт (JAR, image). Reproducible: один commit → один артефакт.

**Deployment** — процесс размещения артефакта в целевой среде (Pod в K8s, VM, serverless) и активации (запуск процесса, направление трафика).

Между ними — **Release**: связывание артефакта с конфигурацией для конкретной среды. `myapp:1.0.42` (build) + `application-prod.yml` (config) = `release-prod-1.0.42` (release). Deploy — активация этого release.

12-factor разделяет три стадии: build → release → deploy. Каждая immutable. Один build → много release'ов (для разных env) → любое количество deploy'ев одного release'а (перезапуски).

### Q2: Какие есть виды деплоя?

Три оси классификации:

**По способу замены** (техническая):
- **Recreate** — снести все → поднять все. Downtime.
- **Rolling Update** — постепенно, по Pod'у. Zero-downtime default.
- **Blue-Green** — две параллельные среды, flip switch.
- **Canary** — постепенный переход трафика по %.

**По видимости для пользователей**:
- **Big Bang** — все разом.
- **Ring / Wave** — по кольцам аудитории.
- **A/B Testing** — параллельное сравнение.
- **Feature Flags / Dark Launch** — код в проде, включается флагом.

**По типу трафика**:
- **Live** — реальный.
- **Shadow / Mirror** — копия live-трафика.

Реальный production обычно комбинирует: canary + feature flags + shadow.

### Q3: Rolling vs Blue-Green — когда что?

**Rolling** — default choice. 99% случаев. Плюсы: экономно (нет 2× ресурсов), встроено в K8s, автоматический self-heal. Минусы: обе версии одновременно (compatibility), медленный rollback.

**Blue-Green** — когда:
- Нужен мгновенный rollback (секунды).
- Полный test новой версии перед экспозицией.
- Приложение не поддерживает running side-by-side с предыдущей версией.
- Бюджет на 2× ресурсов есть.

Классическое применение: финансовые системы, health, регуляторные — где даже минута deploy'я unacceptable.

### Q4: Что такое Canary Deployment?

Постепенный вывод новой версии: **маленький процент трафика** идёт на неё, наблюдение метрик, увеличение процента.

Пример: 5% → wait 10 min → check success rate → 25% → wait → check → 50% → wait → 100%.

**Плюсы**: реальный пользовательский трафик, минимальный blast radius, автоматический rollback при деградации.

**Минусы**: долго (часы), сложность (нужен mesh или Argo Rollouts + Prometheus), обе версии одновременно.

Реализация в K8s:
- **Без mesh**: 2 Deployment'а с разными replicas ratio (95%/5% = 19 + 1 Pod'ов).
- **С Istio**: VirtualService с weight-based routing.
- **С Argo Rollouts**: CRD Rollout с этапами и analysis templates (Prometheus метрики).

Название — от «канарейки в шахте»: шахтёры брали птицу под землю; если газ утёк — птица умрёт первой, шахтёры узнают.

### Q5: Как работает Rolling Update в K8s?

Deployment имеет `strategy.rollingUpdate.maxSurge` и `maxUnavailable`.

1. При изменении `spec.template` — Deployment controller создаёт новый ReplicaSet с новым template hash.
2. Скейлит новый RS вверх, старый вниз, соблюдая invariants:
   - Total pods ≤ `replicas + maxSurge`.
   - Ready pods ≥ `replicas - maxUnavailable`.
3. Ждёт readiness каждого нового Pod'а перед killing старого.
4. Продолжает до полной замены.

**Zero-downtime prod настройка**: `maxSurge: 1`, `maxUnavailable: 0`. Никогда не убиваем без нового Ready.

**Rollback**: `kubectl rollout undo` — обратный процесс, старый RS оставался с 0 replicas, скейлится вверх.

**Обязательное условие zero-downtime**: правильные readiness probes, `preStop` hook (для draining), `terminationGracePeriodSeconds` больше graceful shutdown timeout приложения.

### Q6: Как ты бы задеплоил приложение с breaking API change?

Через **expand-contract pattern**, 2-3 релиза:

**Release 1 (expand)**: добавляем новое API, старое оставляем.
- v1.1 поддерживает `/api/v1/users/{id}` (старое) И `/api/v2/users/{id}` (новое).

**Release 2 (migrate consumers)**: все клиенты переходят на v2.
- Мониторим трафик на `/api/v1/*`. Пока не 0 — не удаляем.

**Release 3 (contract)**: убираем старое API.
- v1.2 без `/api/v1/*`.

Пример с БД схемой — аналогично:
1. `ADD COLUMN full_name` + sync trigger.
2. Deploy v2 которая пишет `full_name`.
3. Backfill.
4. Deploy v3 которая читает только `full_name`.
5. `DROP COLUMN name`.

Никогда одним PR: `изменил schema + изменил код`.

### Q7: Что такое Feature Flag и как это связано с deploy?

Feature flag — механизм включать/выключать функциональность **без redeploy**.

Разделяет **deploy** (код в проде) и **release** (фича видима пользователям).

Простейшая реализация:
```java
@Value("${feature.new-checkout:false}")
boolean enabled;

if (enabled) newCheckout(); else oldCheckout();
```

Продвинутые системы (Unleash, LaunchDarkly) — динамическое включение по user/geo/percentage без рестарта.

**Практическое применение**:
- Deploy в пятницу можно, release в понедельник (когда команда на месте).
- Постепенный rollout: 1% → 10% → 100%.
- Instant kill switch без redeploy.
- A/B testing.

**Anti-patterns**:
- Флаги накапливаются, dead code (правило: expiration date на каждый флаг).
- Каждая фича за флагом (перебор, сложность).

### Q8: Что происходит с in-flight запросами при rolling update?

Проблема: Pod маркирован Terminating → удаляется из EndpointSlice → **но kube-proxy на всех нодах** не сразу обновит iptables (1-5 сек). В это время новые запросы всё ещё могут прилететь в умирающий Pod.

Решение — **preStop hook + graceful shutdown в приложении**:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
terminationGracePeriodSeconds: 45
```

Порядок:
1. Pod → Terminating, endpoints обновляются.
2. `preStop sleep 10` — окно для распространения iptables.
3. `SIGTERM` в контейнер.
4. Spring Boot (с `server.shutdown: graceful`) прекращает принимать новые запросы, ждёт in-flight до `spring.lifecycle.timeout-per-shutdown-phase: 25s`.
5. Boot exits, container done.
6. Terminated within `terminationGracePeriodSeconds`.

Если `terminationGracePeriodSeconds` меньше чем время graceful shutdown → SIGKILL в середине → in-flight запросы обрываются.

### Q9: Разница CI, Continuous Delivery, Continuous Deployment?

- **CI (Continuous Integration)** — каждый commit автоматически билдится и тестируется.
- **CD (Continuous Delivery)** — каждый успешный build **готов к deploy** в prod. Deploy — по кнопке.
- **CDeployment** — каждый успешный build **автоматически** идёт в prod.

Пирамида: CI → CDelivery → CDeployment. Каждый уровень сложнее (нужна лучше автоматизация, тесты, observability, rollback).

Continuous Deployment требует:
- Полное покрытие тестами.
- Автоматический smoke test после deploy.
- Автоматический rollback при деградации метрик.
- Feature flags для рискованных изменений.

Netflix / Amazon делают Continuous Deployment. Большинство enterprise — Continuous Delivery (кнопка в prod).

### Q10: Что такое GitOps?

GitOps = **git — единственный источник правды** для declarative infrastructure. Оператор в кластере (ArgoCD, Flux) watches git repo, синхронизирует состояние кластера с манифестами в git.

**Классика (без GitOps)**:
```
Developer → CI собирает image + делает kubectl apply → cluster
```
Проблемы: CI имеет admin credentials к кластеру, drift между git и cluster, no audit trail.

**GitOps way**:
```
Developer → CI собирает image + PR в manifests-repo → merge → ArgoCD видит → sync cluster
```

ArgoCD в кластере, у него read-only pull из git. CI никаких kubectl. Git всегда = кластер (`selfHeal: true`).

Плюсы: audit trail через git, rollback = git revert, drift detection, PR-based approvals.

Подробно — `79-cicd-deploy-patterns.md §2`.

### Q11: Immutable vs Mutable deployment?

**Mutable**: SSH на сервер → скопировал новый JAR → рестартнул. State сервера меняется, накапливается drift (снежинки).

**Immutable**: собрал новый image → развернул новую VM/Pod из этого image → убил старую. Никаких изменений на месте, всегда replace.

Kubernetes — **immutable by design**. Нельзя «редактировать Pod», только заменить.

Immutable плюсы: 
- Reproducibility (одинаковые images → одинаковое поведение).
- Rollback = redeploy предыдущий image.
- Нет configuration drift.
- Simplified debugging (state = image).

Minus: больше resource overhead (полный replace), сложнее для stateful приложений.

Cattle vs pets: серверы — крупный рогатый скот, не домашние животные. Не лечишь — заменяешь.

### Q12: Как обеспечить zero-downtime deploy Spring Boot приложения?

Полный чек-лист:

**Spring Boot config**:
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
management:
  endpoint.health.probes.enabled: true
  endpoints.web.exposure.include: health,info,prometheus
```

**K8s Deployment**:
- `strategy.rollingUpdate: {maxSurge: 1, maxUnavailable: 0}`.
- `terminationGracePeriodSeconds: 45` (больше Spring timeout).
- `preStop` hook `sleep 10` (kube-proxy update).
- `readinessProbe` на `/actuator/health/readiness`.
- `livenessProbe` на `/actuator/health/liveness` (не readiness).
- `startupProbe` для медленного Spring Boot старта (JVM warmup).
- `PodDisruptionBudget` с `minAvailable: replicas - 1`.

**Клиенты**: HTTP клиенты имеют retry на 5xx / connection reset для idempotent операций.

**Backward compatibility**: API endpoints и DB schema поддерживают старую версию во время rolling.

**Что происходит**:
1. New Pod поднимается, startup passes, readiness Ready → трафик идёт.
2. Old Pod → Terminating → удалён из EndpointSlice → kube-proxy update (1-5 сек).
3. preStop `sleep 10` — ждём распространения iptables.
4. `SIGTERM` → Boot прекращает принимать новые, ждёт in-flight.
5. Boot exits (в течение 25 сек) → container done.

### Q13: Что такое deployment topology?

Уровень абстракции где реально запускается приложение:

- **Bare metal** → физический сервер, JAR + systemd.
- **VM** → виртуалка в облаке, Terraform + Ansible.
- **Container** → Docker на VM.
- **Kubernetes** → orchestrated containers.
- **Serverless / FaaS** → Lambda, только код + trigger.
- **Managed platforms** → Heroku, Cloud Run, git push → running.

Каждый уровень выше = меньше твоего контроля, больше автоматизации, обычно дороже per-request но дешевле по dev-time.

Java-мир — обычно K8s или Cloud Run для новых проектов, VM/bare metal для legacy.

### Q14: Что такое Shadow / Traffic Mirroring?

Реальный live трафик **копируется** в новую версию, но её ответ не возвращается клиенту. Клиент видит только ответ старой.

```
User → Load Balancer → BLUE (v1.5) → response to user
                    ↓ (mirror)
                    SHADOW (v1.6) → response discarded
```

**Применение**: тестирование новой версии на реальной нагрузке без риска. Особенно полезно для рефакторинга — сравниваем ответы двух версий, убеждаемся что новая эквивалентна старой.

**Проблема**: side effects. Если shadow-запросы делают DB writes → двойные записи. Решения: shadow работает read-only, отдельная shadow DB (replica), или feature flag «shadow mode → skip writes» внутри приложения.

Istio реализует через `mirror` в VirtualService.

### Q15: Ring-based deployment — что это?

Постепенный deploy по «кольцам» аудитории:
- Ring 0: internal (employees).
- Ring 1: beta users.
- Ring 2: 10% регионов.
- Ring 3: All users.

Каждое кольцо — feedback window. Проблема на ring 1 → не докатывается до ring 3.

По сути **canary с explicit user groups** вместо percentage. Реализация через feature flag сервисы + user attributes.

Классический пример — Microsoft Windows Insider программа, Google Chrome (Canary → Beta → Stable channels).

Для внутренних сервисов обычно overkill — canary по проценту достаточно.

### Q16: Разница semver и calver?

**Semver** (`MAJOR.MINOR.PATCH`, `1.4.2`):
- Major — breaking changes.
- Minor — features, backward compat.
- Patch — bug fixes.

Для API/library contracts. Клиенты решают «можно ли обновиться безопасно?» по номеру.

**Calver** (`YYYY.MM.PATCH`, `2026.09.5`):
- Год.Месяц.Патч.
- Для time-based releases.

Использует Ubuntu (`24.04`), JetBrains, Firefox. Сразу видно возраст. Не показывает breaking changes.

Для internal сервисов часто используют git SHA как immutable identifier, semver — для публичных libraries.

### Q17: Что происходит при `kubectl apply`?

1. `kubectl` computes 3-way merge между текущим объектом в etcd и новым манифестом (учитывая `last-applied` annotation).
2. Отправляет PATCH на apiserver.
3. apiserver: authentication → authorization (RBAC) → mutating admission (webhooks, defaults) → schema validation → validating admission → write to etcd.
4. Возвращает результат клиенту.
5. Controllers видят изменение через watch:
   - Deployment controller: если `spec.template` изменился → создаёт новый ReplicaSet.
   - ReplicaSet controller: скейлит согласно strategy.
6. Scheduler: назначает `nodeName` для новых Pod'ов.
7. Kubelet на ноде через CRI (containerd) через OCI runtime (runc) → linux syscalls (clone, unshare, pivot_root) → контейнер запущен.
8. Приложение стартует, readiness passes.
9. EndpointSlice controller добавляет Pod IP.
10. Kube-proxy обновляет iptables → трафик идёт.

Всё это — async, level-triggered reconciliation. Подробно — `80-kubernetes-internals-interview.md`.

### Q18: Как быстро откатить deploy если что-то сломалось?

По скорости rollback (от быстрого к медленному):

1. **Feature flag off** (миллисекунды) — если фича за флагом.
2. **Blue-Green flip** (секунды) — если есть blue-green setup.
3. **kubectl rollout undo** (десятки секунд — минуты) — rolling reverse.
4. **Redeploy previous image** через GitOps — минуты (git revert + Argo sync).
5. **PIT restore из бэкапа** — часы, потеря данных. Последнее средство.

**Правило prod**: **rollback first, debug later**. При инциденте — сначала возвращаемся к стабильному состоянию, потом разбираемся почему. Не тратим downtime на диагностику.

### Q19: Что такое release engineering?

Release engineering — дисциплина о том, как **надёжно** доставлять изменения от commit до пользователей. Включает:
- Build automation (reproducible).
- Test automation (unit → integration → e2e).
- Environment management (dev/staging/prod parity).
- Deployment strategies (rolling/canary/blue-green).
- Rollback procedures.
- Observability (метрики, логи, tracing).
- Incident response (runbooks, postmortems).
- Compliance/audit.

В Google — отдельная роль **SRE Release Engineer**. В крупных компаниях — целые команды. В маленьких — часть DevOps culture.

Ключевая цель — **низкий Change Failure Rate** (< 15% deploy'ев вызывают инциденты) и **быстрый MTTR** (< 1 час на восстановление).

### Q20: Что такое 12-factor app в контексте deploy?

12-factor — методология для cloud-native приложений (Heroku, 2011). Ключевое для deploy:

- **III Config** — конфиг в env vars, не в коде. Тот же image в dev / staging / prod с разными env.
- **V Build, release, run** — три раздельные стадии, каждая immutable.
- **VI Processes** — приложение stateless, state в внешних БД / cache.
- **IX Disposability** — быстрый старт и graceful shutdown.
- **X Dev/prod parity** — все env максимально одинаковы.
- **XI Logs** — stdout/stderr, не в файл. Инфраструктура собирает.

Practical outcome: 12-factor app «естественно» деплоится в K8s. Один образ, разные ConfigMap/Secret per env, stateless, graceful, logs → stdout → Loki/ELK.

Anti-pattern: приложение читает конфиг из hardcoded файла, пишет log в `/var/log/app.log` внутри контейнера, requires reboot at config change — плохо на K8s.

---

## 13. Мини-шпаргалка «прочитал — знаю»

За 3-5 секунд должно всплывать по каждому пункту:

- [ ] Build vs Release vs Deploy — три отдельные стадии, каждая immutable.
- [ ] Один build → много release'ов → бесконечное количество deploy'ев одного release'а.
- [ ] Fat JAR vs regular JAR — самодостаточность.
- [ ] Reproducible build — bit-for-bit одинаковый output для одного commit.
- [ ] Docker layer cache — оптимизация через порядок COPY.
- [ ] Recreate — downtime, простой, dev only.
- [ ] Rolling — default, zero-downtime, обе версии одновременно.
- [ ] `maxSurge`, `maxUnavailable`, `progressDeadlineSeconds`.
- [ ] Blue-Green — мгновенный rollback, 2× ресурсов.
- [ ] Canary — постепенный %, автоматика через Argo Rollouts + Prometheus.
- [ ] A/B — не про deploy, про продукт experiments.
- [ ] Shadow — mirror traffic, риск side effects.
- [ ] Dark Launch — код в проде за флагом.
- [ ] Ring-based — canary с группами вместо процентов.
- [ ] Feature flags — separation of deploy and release.
- [ ] Immutable deployment — cattle not pets.
- [ ] Bare metal → VM → Container → K8s → Serverless — по уровню абстракции.
- [ ] Image tags: digest ≫ semver ≫ SHA ≫ `latest`.
- [ ] Expand-Contract pattern для БД миграций.
- [ ] Migration как отдельный Job / PreSync hook.
- [ ] Environments: dev → QA → staging → prod, promotion того же image.
- [ ] Kustomize (overlays) vs Helm (templates).
- [ ] GitOps: ArgoCD/Flux, git = truth.
- [ ] Semver: MAJOR.MINOR.PATCH.
- [ ] Rollback: git revert / rollout undo / blue-green flip / feature flag.
- [ ] preStop hook + graceful shutdown + terminationGracePeriodSeconds — zero-downtime.
- [ ] 12-factor: build/release/run separation, env vars config, disposability.
- [ ] DORA metrics: deploy frequency, lead time, change failure rate, MTTR.

Если по каждому объясняешь за 3-5 предложений — уровень middle+ в deployment engineering.

---

## Итог

**Build** = source → artifact. Одинаковый source → одинаковый artifact.
**Release** = artifact + config. Immutable release identifier.
**Deploy** = активация release в среде. Может повторяться (crash recovery).

**Виды деплоя** — не один правильный ответ, а спектр по трём осям:
- Как заменяем: Recreate → Rolling → Blue-Green → Canary.
- Кто видит: Big Bang → Ring → A/B → Feature Flags → Dark Launch.
- Куда трафик: Live → Shadow → Synthetic.

Прод-стандарт для микросервисов на K8s в 2026:
- **Rolling update** для 99% случаев с zero-downtime настройкой.
- **Blue-Green** для критичных финтеч/health систем.
- **Canary + Argo Rollouts + Prometheus** для рискованных изменений и крупных ecosystem.
- **Feature flags** обязательны для новых рисковых фич.
- **GitOps** через ArgoCD/Flux как способ deploy.
- **Immutable everything** — image, release, config.
- **Expand-Contract** для БД миграций.

Дальше в теме:
- Книга «Continuous Delivery» Jez Humble и David Farley (классика 2010).
- «Site Reliability Engineering» (Google SRE book, бесплатно).
- «Accelerate: The Science of Lean Software and DevOps» — DORA metrics.
- Practice: Argo Rollouts tutorial, Flagger docs.
