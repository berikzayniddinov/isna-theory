# 124. Build и Deploy визуально: реальный pipeline ISNA tax-rep

## Про проект

**isna-tax-rep** — приложение отчётности КНП (Кабинет налогоплательщика Казахстана).

- **~54 микросервиса** (модули gradle): `isna-tax-rep`, `isna-tax-rep-100`, `isna-tax-rep-110`, `-200`, `-328`, `-report`, `-sync`, `-fo`, `-integration`, `-liquibase`, `-nz-001`, `-nz-007` и т.д.
- **Java 11** (большая часть) + **Java 21** (isna-tax-rep-report и другие).
- **Gradle 6.9.4** + плагин **jib** (собирает Docker images без Docker daemon).
- **CI**: GitLab CI.
- **Kubernetes namespace**: `tax-report`.

## 1. Главная схема — три ветки, три окружения

```
    ┌──────────────┐
    │  Developer   │
    │              │
    │  git commit  │
    │  git push    │
    └──────┬───────┘
           │
           ▼
    ┌────────────────────────────────────────┐
    │    GitLab (2 разных сервера)           │
    │                                        │
    │  ┌──────────────────┐                  │
    │  │ gitlab.1sc.kz    │  ← dev + release │
    │  │                  │                  │
    │  │  ветки:          │                  │
    │  │  - dev           │                  │
    │  │  - release       │                  │
    │  │  - release-*     │                  │
    │  └──────────────────┘                  │
    │                                        │
    │  ┌──────────────────┐                  │
    │  │ gitlab.kgd.gov.kz│  ← только prod   │
    │  │                  │                  │
    │  │  ветки:          │                  │
    │  │  - master        │                  │
    │  └──────────────────┘                  │
    └──────┬─────────────────────────────────┘
           │
           │  push триггерит CI
           ▼
    ┌────────────────────────────────────────┐
    │       .gitlab-ci.yml                   │
    │                                        │
    │  Определяет что делать:                │
    │  - какие тесты запустить               │
    │  - какие images собрать                │
    │  - в какой Nexus залить                │
    │  - в какой K8s задеплоить              │
    └──────┬─────────────────────────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────────┐
    │              14 стадий (stages)                          │
    │                                                          │
    │   ┌──────────┐  ┌──────────┐  ┌──────────┐               │
    │   │ mr_gate  │→ │ test_dev │→ │build_dev │→ deploy_dev   │
    │   └──────────┘  └──────────┘  └──────────┘               │
    │                                                          │
    │   test_preprod → build_preprod → deploy_preprod          │
    │                                                          │
    │   test_preprod_java21 → build_deploy_preprod_java21      │
    │                                                          │
    │   e2e_gate → test_prod → build_prod → deploy_prod        │
    │                                                          │
    │   → sonar_prod                                           │
    └──────┬───────────────────────────────────────────────────┘
           │
           ▼
    Три разных Nexus сервера + три разных K8s кластера
```


## 2. Три окружения — dev / preprod / prod

**Каждое окружение имеет свой Nexus и свой K8s**:

```
    ┌────────────────────────────────────────────────────────┐
    │                    DEV окружение                       │
    │                                                        │
    │   Nexus:                                               │
    │   registry.1sc.kz                                      │
    │                                                        │
    │   K8s cluster (dev)                                    │
    │   namespace: tax-report                                │
    │                                                        │
    │   Триггер: push в ветку "dev"                          │
    │   Кто пушит: разработчики свободно                     │
    └────────────────────────────────────────────────────────┘
    
    ┌────────────────────────────────────────────────────────┐
    │                  PREPROD окружение                     │
    │                                                        │
    │   Nexus:                                               │
    │   test-nexus.kgd.gov.kz                                │
    │                                                        │
    │   K8s cluster (preprod)                                │
    │   namespace: tax-report                                │
    │                                                        │
    │   Триггер: push в ветку "release"                      │
    │   Кто пушит: через MR + code review                    │
    │                                                        │
    │   ⚠ Дополнительно: e2e_gate                            │
    │   → перед prod проверяются e2e тесты                   │
    └────────────────────────────────────────────────────────┘
    
    ┌────────────────────────────────────────────────────────┐
    │                    PROD окружение                      │
    │                                                        │
    │   Nexus:                                               │
    │   nexus.kgd.gov.kz    ← у клиента внутри контура       │
    │                                                        │
    │   K8s cluster (prod)                                   │
    │   namespace: tax-report                                │
    │                                                        │
    │   Триггер: push в ветку "master"                       │
    │   (только gitlab.kgd.gov.kz!)                          │
    │                                                        │
    │   ⚠ Требует: e2e_gate прошёл                           │
    └────────────────────────────────────────────────────────┘
```


## 3. Что происходит на BUILD (один сервис)

**Пример**: сборка `isna-tax-rep-328`.

```
    Исходники модуля:
    
    isna-tax-rep-328/
    ├── src/main/java/         ← Java код
    ├── src/main/resources/    ← application.yml, etc
    ├── build.gradle           ← как собирать
    └── ...
                    │
                    │ gradle bootJar jibDockerBuild
                    │
                    ▼
    ┌───────────────────────────────────────────────┐
    │  Шаг 1: gradle bootJar                        │
    │                                               │
    │  Собирает Spring Boot fat JAR                 │
    │  включает все зависимости                     │
    │                                               │
    │  → isna-tax-rep-328/build/libs/               │
    │    isna-tax-rep-328.jar (~50-100 MB)          │
    └────────────────┬──────────────────────────────┘
                     │
                     ▼
    ┌───────────────────────────────────────────────┐
    │  Шаг 2: jibDockerBuild                        │
    │                                               │
    │  Плагин Jib собирает Docker image             │
    │  БЕЗ Docker daemon (это круто!)               │
    │                                               │
    │  Использует базу:                             │
    │  registry.1sc.kz/adoptopenjdk:11-alpine       │
    │                                               │
    │  Результат:                                   │
    │  <registry>/isna-tax-rep-328:<git_sha>        │
    │  ~250-300 MB                                  │
    └────────────────┬──────────────────────────────┘
                     │
                     ▼
    ┌───────────────────────────────────────────────┐
    │  Шаг 3: docker push                           │
    │                                               │
    │  Пушит два тега:                              │
    │  - isna-tax-rep-328:abc12345 (SHA коммита)    │
    │  - isna-tax-rep-328:latest                    │
    │                                               │
    │  В Nexus (в зависимости от окружения):        │
    │  - registry.1sc.kz          (dev)             │
    │  - test-nexus.kgd.gov.kz    (preprod)         │
    │  - nexus.kgd.gov.kz         (prod)            │
    └────────────────┬──────────────────────────────┘
                     │
                     ▼
    ┌───────────────────────────────────────────────┐
    │  Шаг 4: kubectl set image                     │
    │                                               │
    │  Обновляет deployment в K8s                   │
    │  на новый image                               │
    │                                               │
    │  kubectl set image \                          │
    │    deployment/isna-tax-rep-328 \              │
    │    isna-tax-rep-328=<registry>/               │
    │      isna-tax-rep-328:abc12345 \              │
    │    -n tax-report                              │
    └───────────────────────────────────────────────┘
```


## 4. Полный цикл build → deploy для ВСЕХ ~54 сервисов

**Что делает CI на push в `release`**:

```
    push в release
            │
            ▼
    ┌────────────────────────────────────────────────────┐
    │           GitLab CI Pipeline запускается           │
    └────────────────────────┬───────────────────────────┘
                             │
                             ▼
    ┌────────────────────────────────────────────────────┐
    │  Stage: test_preprod                               │
    │                                                    │
    │  Для каждого из 54 модулей:                        │
    │  cd isna-tax-rep-N                                 │
    │  gradle test                                       │
    │                                                    │
    │  ✗ хоть один упал → pipeline красный, стоп         │
    └────────────────────────┬───────────────────────────┘
                             │
                             ▼
    ┌────────────────────────────────────────────────────┐
    │  Stage: build_preprod                              │
    │                                                    │
    │  export docker_registry=test-nexus.kgd.gov.kz      │
    │  docker login -u $PREPROD_USER -p $PREPROD_PASS    │
    │                                                    │
    │  для каждого из 54 сервисов:                       │
    │    cd $SERVICE                                     │
    │    gradle bootJar jibDockerBuild                   │
    │      -Djib.to.image=$registry/$IMAGE:$CI_SHA       │
    │    docker tag $registry/$IMAGE:$SHA $IMAGE:latest  │
    │                                                    │
    │  для каждого:                                      │
    │    docker push $registry/$IMAGE:$SHA               │
    │    docker push $registry/$IMAGE:latest             │
    │                                                    │
    │  → 54 images залиты в Nexus                        │
    └────────────────────────┬───────────────────────────┘
                             │
                             ▼
    ┌────────────────────────────────────────────────────┐
    │  Stage: deploy_preprod                             │
    │                                                    │
    │  для каждого из 54 сервисов:                       │
    │    kubectl set image \                             │
    │      deployment/$IMAGE_NAME \                      │
    │      $IMAGE_NAME=$registry/$IMAGE:$CI_SHA \        │
    │      -n tax-report                                 │
    │                                                    │
    │  K8s делает rolling update каждого deployment      │
    └────────────────────────┬───────────────────────────┘
                             │
                             ▼
    ┌────────────────────────────────────────────────────┐
    │  Stage: e2e_gate                                   │
    │                                                    │
    │  Запускает e2e тесты из knp-e2e проекта            │
    │  ✓ прошёл → можно мержить в master                 │
    │  ✗ упал → блок мержа                               │
    └────────────────────────────────────────────────────┘
```


## 5. Nexus — где что хранится

**Три отдельных Nexus сервера**:

```
    ┌────────────────────────────────────────┐
    │  registry.1sc.kz (dev)                 │
    │                                        │
    │  Docker Registry:                      │
    │    isna-tax-rep:abc123                 │
    │    isna-tax-rep-100:abc123             │
    │    isna-tax-rep-328:abc123             │
    │    ... (54 сервиса × много SHA)        │
    │                                        │
    │  Base images:                          │
    │    adoptopenjdk:11-alpine              │
    │    gradleimage:6.9.4                   │
    │                                        │
    │  Gradle репозиторий:                   │
    │    (кэш Maven Central + свои)          │
    └────────────────────────────────────────┘
    
    ┌────────────────────────────────────────┐
    │  test-nexus.kgd.gov.kz (preprod)       │
    │                                        │
    │  Точно такая же структура,             │
    │  но для preprod окружения              │
    │                                        │
    │  ⚠ Отдельный сервер, отдельная         │
    │    сеть                                │
    └────────────────────────────────────────┘
    
    ┌────────────────────────────────────────┐
    │  nexus.kgd.gov.kz (prod)               │
    │                                        │
    │  У клиента внутри защищённого          │
    │  контура                               │
    │                                        │
    │  Доступ только с prod сервера          │
    └────────────────────────────────────────┘
```


## 6. Deploy в Kubernetes — как реально идёт

**После `kubectl set image` в K8s**:

```
    K8s cluster (preprod)
    namespace: tax-report
    
    ┌────────────────────────────────────────────────────┐
    │           Deployment: isna-tax-rep-328             │
    │           replicas: 2                              │
    │           image: test-nexus.../328:abc123          │
    │                                                    │
    │  До update (старая версия xyz789):                 │
    │  ┌──────────┐  ┌──────────┐                        │
    │  │ Pod 1    │  │ Pod 2    │                        │
    │  │  xyz789  │  │  xyz789  │                        │
    │  └──────────┘  └──────────┘                        │
    │                                                    │
    └────────────────────────────────────────────────────┘
                          │
                          │ kubectl set image ... abc123
                          ▼
    ┌────────────────────────────────────────────────────┐
    │  Rolling update:                                   │
    │                                                    │
    │  1. Создать новый Pod с abc123                     │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
    │  │ Pod 1    │  │ Pod 2    │  │ Pod 3    │          │
    │  │  xyz789  │  │  xyz789  │  │  abc123  │          │
    │  └──────────┘  └──────────┘  └──────────┘          │
    │                                ↑ starting          │
    │                                                    │
    │  2. Ждём health check                              │
    │                                                    │
    │  3. Убить старый Pod 1                             │
    │                ┌──────────┐  ┌──────────┐          │
    │                │ Pod 2    │  │ Pod 3    │          │
    │                │  xyz789  │  │  abc123  │          │
    │                └──────────┘  └──────────┘          │
    │                                                    │
    │  4. Создать новый Pod с abc123                     │
    │  ┌──────────┐  ┌──────────┐                        │
    │  │ Pod 4    │  │ Pod 3    │                        │
    │  │  abc123  │  │  abc123  │                        │
    │  └──────────┘  └──────────┘                        │
    │           ↑ starting                               │
    │                                                    │
    │  5. Убить Pod 2                                    │
    │                                                    │
    │  Итог: 2 pod'а с abc123                            │
    │  ┌──────────┐  ┌──────────┐                        │
    │  │ Pod 3    │  │ Pod 4    │                        │
    │  │  abc123  │  │  abc123  │                        │
    │  └──────────┘  └──────────┘                        │
    │                                                    │
    │  Downtime: 0 (всегда работал минимум 1 pod)        │
    └────────────────────────────────────────────────────┘
```


## 7. Как pod физически берёт image из Nexus

```
    K8s Node                        Nexus
    ┌──────────────┐              ┌──────────────┐
    │              │              │              │
    │  kubelet     │              │  registry.   │
    │              │              │  1sc.kz      │
    │  создаёт Pod │              │              │
    │              │              │              │
    │  сначала     │  1. GET image manifest      │
    │  нужен image ├─────────────▶│              │
    │              │              │              │
    │              │  2. проверяет: есть layers? │
    │              │              │              │
    │              │  3. GET каждый layer        │
    │              ├─────────────▶│              │
    │              │◀─────────────│  качает      │
    │              │              │  ~50-100 MB  │
    │              │              │              │
    │              │  4. сохраняет в             │
    │              │     /var/lib/containerd/    │
    │              │              │              │
    │              │  5. запускает container     │
    │              │     из скачанного image     │
    │              │                             │
    │              │  6. Pod становится Running  │
    └──────────────┘              └──────────────┘
    
    ⚠ Первый раз качается долго (~30 сек - 2 минуты).
    Дальше layers кэшируются на node — быстро.
```


## 8. Полный путь от commit до prod

**Реальный сценарий разработки в КНП**:

```
    Разработчик работает над ISNA2-23803 (пример)
    
    День 1: разработка
    ────────────────────
    
    git checkout -b release-ISNA2-23803
    (пишет код в isna-tax-rep-328)
    git commit -m "ISNA2-23803: ФО 000.10 ..."
    git push
    
                        │
                        ▼
    
    ┌──────────────────────────┐
    │  gitlab.1sc.kz           │
    │  ветка: release-ISNA2-*  │
    └────────┬─────────────────┘
             │
             │ создаёт Merge Request → release
             │
             ▼
    ┌──────────────────────────┐
    │  Merge Request           │
    │                          │
    │  - code review           │
    │  - CI прогоняет тесты    │
    │  - e2e_gate check        │
    └────────┬─────────────────┘
             │
             │ merge в release
             ▼
    ┌──────────────────────────────────┐
    │  CI автоматически:               │
    │  test_preprod → build_preprod →  │
    │  deploy_preprod                  │
    │                                  │
    │  все 54 сервиса обновлены        │
    │  в preprod                       │
    └────────┬─────────────────────────┘
             │
             │ проходит e2e_gate
             ▼
    ┌──────────────────────────────────┐
    │  QA тестирует на preprod         │
    │  (test-arm.kgd.gov.kz)           │
    └────────┬─────────────────────────┘
             │
             │ если всё ок
             ▼
    ┌──────────────────────────────────┐
    │  Мерж release → master           │
    │  (в gitlab.kgd.gov.kz)           │
    │                                  │
    │  ⚠ Отдельный GitLab у клиента!   │
    └────────┬─────────────────────────┘
             │
             │ CI на master:
             │ test_prod → build_prod → deploy_prod
             │
             ▼
    ┌──────────────────────────────────┐
    │  PRODUCTION                      │
    │                                  │
    │  nexus.kgd.gov.kz                │
    │  K8s cluster (prod)              │
    │  namespace: tax-report           │
    │                                  │
    │  Пользователи через ЭЦП          │
    │  видят новую фичу                │
    └──────────────────────────────────┘
```


## Что запомнить

**Проект isna-tax-rep = ~54 микросервиса** (модули gradle) в одном git-репо.

**Три окружения = три разных Nexus + три K8s**:
- dev → `registry.1sc.kz`
- preprod → `test-nexus.kgd.gov.kz`
- prod → `nexus.kgd.gov.kz`

**Два разных GitLab сервера**:
- `gitlab.1sc.kz` — dev/release (разработка).
- `gitlab.kgd.gov.kz` — только master (у клиента).

**14 стадий CI pipeline**:
- mr_gate → test → build → deploy для каждого окружения
- e2e_gate между preprod и prod (обязательный)

**Build через Jib**:
- `gradle bootJar jibDockerBuild` — собирает Docker image БЕЗ Docker daemon.
- Каждый image имеет 2 тега: `:<git_sha>` и `:latest`.

**Deploy через `kubectl set image`**:
- Не через Helm или манифесты.
- Просто меняет image в существующем Deployment.
- K8s делает rolling update автоматически.

**Namespace = `tax-report`** — все 54 сервиса в одном namespace K8s.

**Путь: dev → release → master**:
- Обычный flow: разработка в feature ветке → MR в release → тестирование → мерж в master → prod.

---

Дальше:
- **03** — Gradle детально.
- **04** — JAR / fat JAR.
- **09** — Docker.
- **10, 77** — Kubernetes.
- **79** — CI/CD deploy patterns.
- **82** — git push to pod full pipeline.
- **121** — Maven vs Gradle.
