# 79. CI/CD и deploy-паттерны: GitOps, blue-green, canary, feature flags

Как правильно катать изменения в прод: паттерны деплоя, GitOps подход, feature flags. Как избежать «сломался деплой в пятницу вечером».

---

## 1. Основные CI/CD паттерны

### 1.1 Trunk-based development

Все разработчики коммитят в `main` (или `master`). Никаких долгих feature-веток. Каждый commit проходит CI, коммитится сразу.

**Плюсы**: нет merge-hell'а, все видят изменения других сразу, короткая обратная связь.

**Минусы**: требует высокой дисциплины, полного unit-test coverage, feature flags (потому что незаконченная фича = в trunk = в проде).

Используется в Google, Facebook, Netflix. Требует зрелой команды.

### 1.2 GitFlow

`main` → `develop` → feature/hotfix/release ветки. Слияния в кучу.

**Плюсы**: чёткие релиз-циклы.

**Минусы**: сложно, часто merge conflicts, замедляет доставку.

Устарел для большинства команд. Хорошо для проектов с длинными релиз-циклами (embedded, финтех с квартальными релизами).

### 1.3 GitHub Flow / GitLab Flow

Feature-branch с ПР, короткая жизнь ветки (пара часов — пара дней), merge в main → deploy. Проще GitFlow.

Хороший baseline для большинства команд.

### 1.4 KNP-style: master + release + release-* ветки

Как у КНП:
- `master` — прод.
- `release` — препрод (staging).
- `release-<ticket>` — feature-branches для перекатки в release.
- Периодические переносы master → release для инфра-фиксов (см. `feedback_cherry_pick_scope`).

**Плюсы**: контроль что идёт в прод.

**Минусы**: рассинхрон master/release (мой прошлый разбор — 193 release-only коммита в sync). Требует дисциплины периодического переноса.

---

## 2. GitOps: Git = единственный источник правды

**Идея**: желаемое состояние кластера — описано в Git. Оператор (Argo CD, Flux) следит за репозиторием, применяет изменения в кластер.

### 2.1 Классика (без GitOps)

```
Разработчик → CI собирает image → CI делает kubectl apply → кластер
                                    ↑
                          Кто? Права? Логи? Rollback?
```

Проблемы:
- Kubectl apply из CI = у CI runner'а kube-config с админскими правами. Атака на runner = kill кластера.
- Что реально задеплоено — не всегда совпадает с manifests в Git.
- Rollback = revert commit + повторный deploy.

### 2.2 GitOps way

```
Разработчик → CI собирает image + PR в manifests-repo → merge → ArgoCD видит → синхронизирует кластер
```

Оператор в кластере, у него read-only pull из Git. CI никаких kubectl. Git всегда = кластер.

### 2.3 Argo CD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: isnaknpuser
  namespace: argocd
spec:
  project: knp
  source:
    repoURL: https://gitlab.1sc.kz/infra/knp-manifests.git
    targetRevision: HEAD
    path: apps/isnaknpuser/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: knp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

Argo CD **раз в 3 минуты** (или по webhook) сравнивает Git ↔ кластер. Если drift — синхронизирует. `selfHeal: true` = если кто-то руками поменял манифест в кластере, Argo вернёт из Git.

**UI**: диаграмма всех приложений, sync status, health, история deploy'ев.

**Rollback**: `git revert` → Argo видит → откатывает кластер. Всегда через Git.

### 2.4 Flux

Аналог Argo CD, но без UI (CLI + web через Grafana). Более легковесный, GitOps «первопроходец».

Выбор: Argo — приятный UI, попроще войти. Flux — если UI не нужен, minimalism.

### 2.5 Anti-patterns

- **`kubectl apply` в CI после включения ArgoCD** — Argo вернёт назад.
- **Ручное `kubectl edit` в кластере** — Argo `selfHeal` вернёт.
- **Secret'ы в plaintext в Git** — используй **Sealed Secrets** (Bitnami) или **External Secrets Operator** (ключи в Vault/AWS SecretsManager).

---

## 3. Стратегии deploy

### 3.1 Rolling update (стандарт)

Уже разобрано в `77-kubernetes-deep-microservices.md`. Постепенная замена.

Плюсы: без downtime, встроено в Deployment.
Минусы: во время deploy'я работают ОБЕ версии — надо обеспечить backward compatibility (DB схема, API).

### 3.2 Recreate

Убей всех, потом запусти новых. Downtime. Только для dev/staging или систем с обязательным простоем.

### 3.3 Blue-Green

Два параллельных environment'а: **Blue** (текущий prod), **Green** (новая версия). Оба поднимают весь стек.

```
Router / LoadBalancer
    ↓
+------+       +-------+
| BLUE |       | GREEN |
| v1.5 |       | v1.6  |
+------+       +-------+
```

Deploy:
1. Deploy v1.6 в Green (параллельно с v1.5 в Blue).
2. Прогоняем smoke tests в Green (curl-ами, тестовые запросы).
3. Переключаем LB: 100% на Green.
4. Blue стоит как rollback-target ещё сутки.
5. Если rollback нужен: LB → Blue за 1 секунду.

**В k8s реализуется через 2 Deployment'а + 1 Service**:

```yaml
# Deployment blue
apiVersion: apps/v1
kind: Deployment
metadata: {name: isnaknpuser-blue}
spec:
  template:
    metadata:
      labels: {app: isnaknpuser, version: blue}
---
# Deployment green
apiVersion: apps/v1
kind: Deployment
metadata: {name: isnaknpuser-green}
spec:
  template:
    metadata:
      labels: {app: isnaknpuser, version: green}
---
# Service — переключается selector'ом
apiVersion: v1
kind: Service
metadata: {name: isnaknpuser}
spec:
  selector: {app: isnaknpuser, version: blue}   # ← меняешь на green для переключения
```

**Плюсы**: мгновенный rollback, полный test новой версии перед переключением.
**Минусы**: **2x ресурсов** во время deploy. Проблема с БД схемой (обе версии подключены).

### 3.4 Canary

Маленькая часть трафика на новую версию, наблюдение, увеличение процента постепенно.

```
Router
    ↓
95% → BLUE v1.5
 5% → GREEN v1.6 (canary)
```

Если метрики Green хорошие → переключаем 25/75, 50/50, 75/25, 100/0. Если метрики хуже → откатываем в 0.

**В k8s без mesh** — через 2 Deployment'а с разными replicas: `blue=19`, `green=1` → трафик 5% в среднем (случайный по подам).

**С mesh (Istio)**:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata: {name: isnaknpuser}
spec:
  http:
  - route:
    - destination: {host: isnaknpuser, subset: blue}
      weight: 95
    - destination: {host: isnaknpuser, subset: green}
      weight: 5
```

Точный процент, доступен по headers/geolocation. Мощно, но сложно (Istio требует ресурсов и погружения).

**Argo Rollouts** / **Flagger** — расширения k8s которые автоматизируют canary с автоматическим переключением по метрикам Prometheus.

**Плюсы**: real-user monitoring, реальный трафик, минимальный blast radius.
**Минусы**: медленнее (30 мин — часы вместо секунд), сложность метрик.

### 3.5 Shadow / Mirror

Копия live-трафика идёт **и в blue, и в green**. Green не отвечает клиенту (только для наблюдения). Проверяешь reads-heavy сервис без риска.

**С Istio**:
```yaml
http:
- route:
  - destination: {host: myapp, subset: blue}
    weight: 100
  mirror:
    host: myapp
    subset: green
  mirrorPercent: 50
```

Полезно для рефакторинга: убеждаешься что новая реализация даёт те же ответы что старая.

---

## 4. Работа с БД схемой при deploy

Самая сложная часть deploy без downtime. Основные паттерны:

### 4.1 Backward-compatible миграции

Всегда добавляй, никогда не удаляй/переименовывай **в одном релизе**.

**Плохо**:
- v1 использует `user.name`.
- Deploy миграция: `RENAME COLUMN name TO full_name`.
- v2 использует `user.full_name`.
- **Rolling update**: во время deploy'я работают ОБА кода. v1 (ещё живой) читает `name` → ошибка.

**Хорошо (двухфазный deploy)**:
- **Release 1** (expand):
  - Миграция: `ADD COLUMN full_name; UPDATE ... SET full_name = name;`
  - v1 продолжает читать `name` (пишет в оба или триггер).
- **Release 2** (contract, недели/месяцы позже):
  - Все клиенты уже v1.5+, читают `full_name`.
  - Миграция: `DROP COLUMN name`.

Долго. Но безопасно.

### 4.2 Пре-миграция как отдельный Job

```yaml
apiVersion: batch/v1
kind: Job
metadata: {name: liquibase-migrate-2026-09-12}
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: liquibase
        image: liquibase:4.31
        command: [liquibase, --url=jdbc:postgresql://...., update]
```

Argo CD может делать через `PreSync` hook — миграции ДО rolling update приложения.

### 4.3 Feature toggle на новое поле/логику

Deploy v2 с новой колонкой, но чтение/использование за флагом. Мигрируешь → включаешь флаг → тестируешь → откатываешь если проблема. См. секцию feature flags.

---

## 5. Feature flags

### 5.1 Зачем

- Разделение **deploy** и **release**. Код в проде, но пользователи не видят.
- Постепенный rollout: 1% → 10% → 50% → 100%.
- A/B тестирование.
- Kill switch: если новая фича ломает — flip флаг, не rollback.
- Персональный доступ для beta-тестеров.

### 5.2 Простейшая реализация

```java
@Value("${feature.new-search:false}")
private boolean newSearchEnabled;

@GetMapping("/search")
public List<Result> search(@RequestParam String q) {
    if (newSearchEnabled) {
        return newSearchService.search(q);
    }
    return oldSearchService.search(q);
}
```

Флаг из ConfigMap → изменение → rolling restart. Минус: рестарт нужен.

### 5.3 Динамические флаги через ConfigMap + Spring Cloud

Spring Cloud Kubernetes Config: подписывается на ConfigMap, refresh контекста без рестарта.

Или Consul (у КНП уже есть) — можно хранить флаги там, `@RefreshScope`.

### 5.4 Специализированные системы

**Unleash** (open source):
- Свой сервер + Java SDK.
- Флаги по user_id, role, geo, %-rollout, dates.
- UI для управления.

**LaunchDarkly** (SaaS, платный):
- Индустриальный стандарт.
- Real-time updates (WebSocket), не polling.
- Feature dependencies, audit log.

**Togglz** (Java lib, open source):
- Легкий, встраиваемый.
- Console UI бесплатный.

### 5.5 Пример с Unleash

```java
@Autowired UnleashClient unleash;

@GetMapping("/search")
public List<Result> search(@RequestParam String q, Principal user) {
    UnleashContext ctx = UnleashContext.builder()
        .userId(user.getName())
        .build();
    if (unleash.isEnabled("new_search", ctx)) {
        return newSearchService.search(q);
    }
    return oldSearchService.search(q);
}
```

В UI Unleash: "включи `new_search` для 5% пользователей, начиная с завтра". Меняется без рестарта, без deploy'а.

### 5.6 Anti-patterns

- **Флаги никогда не выключаются, dead code накапливается**. Правило: каждый флаг имеет **дату expiration**. Через 3 месяца — удали код старой ветки.
- **Флаги в тестах**: тесты гонятся с одним значением — забываешь тестировать другую сторону. Настрой матрицу.
- **Флаги на всё подряд** — сложность растёт. Флаги для рискованных фич, не для мелочей.

---

## 6. Progressive delivery: canary + метрики + автоматика

### 6.1 Инструменты

**Argo Rollouts** (расширение к Argo CD):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: {name: isnaknpuser}
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 10        # 10% на новую
      - pause: {duration: 5m} # ждём 5 минут
      - analysis:            # проверяем метрики
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: isnaknpuser
      - setWeight: 25
      - pause: {duration: 10m}
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
```

AnalysisTemplate:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: {name: success-rate}
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.99
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_server_requests_seconds_count{
            app="{{args.service-name}}", status!~"5.."
          }[2m])) / 
          sum(rate(http_server_requests_seconds_count{
            app="{{args.service-name}}"
          }[2m]))
```

Argo Rollouts автоматически:
- Поднимает canary с 10%.
- Ждёт 5 минут.
- Опрашивает Prometheus: success_rate ≥ 99%?
- Если да — 25%. Если нет — abort, rollback.

Это magic — deploy без вмешательства человека, safe.

**Flagger** — от Weave (Flux ecosystem), делает то же самое.

### 6.2 SLO / SLI / error budget

**SLI** (Service Level Indicator) — что мерим. `success_rate`, `p99_latency`.
**SLO** (Service Level Objective) — целевое значение. `success_rate ≥ 99.9%` в квартал.
**Error budget** — 100% - 99.9% = 0.1% времени можно быть down. За квартал = ~43 минуты.

Если error budget сгорел (много инцидентов) — freeze deploy'ев до конца периода. Проверенная стратегия Google SRE.

---

## 7. Ротация секретов

Хранение секретов в проде:

### 7.1 k8s Secrets (default)

```bash
kubectl create secret generic db-password --from-literal=password=xxx
```

Хранится в etcd. По умолчанию — base64, НЕ шифрование. `kubectl get secret ... -o yaml` покажет "xxx" в base64.

**Плюсы**: встроено.
**Минусы**: доступно любому с `get secrets` в namespace. Нельзя ротировать без изменения ConfigMap ссылки.

Включи **etcd encryption at rest** (администратор кластера настраивает `EncryptionConfiguration`).

### 7.2 External Secrets Operator (ESO)

Секреты хранятся в **Vault / AWS Secrets Manager / Azure Key Vault / GCP Secret Manager**. Оператор в k8s синхронизирует их в k8s Secret'ы.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: {name: db-password}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target: {name: db-password}
  data:
  - secretKey: password
    remoteRef:
      key: knp/db
      property: password
```

Ротация:
1. В Vault новое значение.
2. Через 1 час ESO обновляет k8s Secret.
3. Deployment ссылается на Secret через `env.valueFrom.secretKeyRef` → k8s **не** авто-перезапускает под, но новый под получит новое значение.
4. Для авто-перезапуска: annotation с hash секрета в deployment.

### 7.3 Sealed Secrets (Bitnami)

Шифруешь секрет публичным ключом controller'а. Зашифрованное валится в Git. В кластере controller расшифровывает и создаёт обычный Secret.

```bash
kubeseal < secret.yaml > sealed-secret.yaml
git add sealed-secret.yaml
```

Плюс: Secrets в Git безопасно (шифрование). Работает с GitOps.
Минус: приватный ключ — критичный (потеря = потеря секретов, компромисс = раскрытие).

---

## 8. Rollback strategy

### 8.1 Автоматический (Argo Rollouts / Flagger)

Метрики плохие → откат автоматически.

### 8.2 Ручной через Git (GitOps)

```bash
git revert HEAD
git push
# ArgoCD видит → откатывает
```

### 8.3 kubectl rollout undo

```bash
kubectl rollout undo deployment/X -n knp
```

Быстро, но НЕ в Git. Argo CD (если есть) откатит обратно. Для emergency.

### 8.4 Что делать если БД мигрирована и rollback не совместим

Тяжелая ситуация. Варианты:
1. Быстро мигрировать назад (если возможно — новая колонка → drop).
2. Hotfix новой версии с игнорированием проблемы (feature flag off).
3. Downtime + restore из бэкапа (последнее средство).

**Правильно — не допускать**. Backward-compatible миграции + expand/contract pattern.

---

## 9. Пример CI/CD пайплайна для isna-knp-подобного сервиса

`.gitlab-ci.yml` (стилизованный):

```yaml
stages:
  - build
  - test
  - security
  - image
  - deploy-preprod
  - integration-test
  - deploy-prod

variables:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false"

build:
  stage: build
  image: gradle:8.14.5-jdk21
  script:
    - gradle assemble --no-daemon
  artifacts:
    paths: [build/libs/*.jar]

unit-test:
  stage: test
  image: gradle:8.14.5-jdk21
  script:
    - gradle test --no-daemon
  artifacts:
    reports:
      junit: build/test-results/test/*.xml

security-scan:
  stage: security
  script:
    - trivy fs --exit-code 1 --severity HIGH,CRITICAL .
    - gradle dependencyCheckAnalyze  # OWASP
  allow_failure: false

build-image:
  stage: image
  image: docker:24
  services:
  - docker:24-dind
  script:
    - docker build -t registry.1sc.kz/isnaknpuser:$CI_COMMIT_SHORT_SHA .
    - trivy image --exit-code 1 --severity HIGH,CRITICAL registry.1sc.kz/isnaknpuser:$CI_COMMIT_SHORT_SHA
    - docker push registry.1sc.kz/isnaknpuser:$CI_COMMIT_SHORT_SHA

deploy-preprod:
  stage: deploy-preprod
  script:
    # Update image tag in Git manifests repo
    - git clone https://gitlab.1sc.kz/infra/knp-manifests.git
    - cd knp-manifests
    - kustomize edit set image registry.1sc.kz/isnaknpuser=registry.1sc.kz/isnaknpuser:$CI_COMMIT_SHORT_SHA
    - git commit -am "Update isnaknpuser to $CI_COMMIT_SHORT_SHA"
    - git push
    # ArgoCD автоматически подхватит
  only: [main, release]

integration-test:
  stage: integration-test
  script:
    - ./scripts/wait-for-deploy.sh isnaknpuser preprod
    - gradle integrationTest -Ptarget=preprod
  only: [main, release]

deploy-prod:
  stage: deploy-prod
  script:
    - git clone https://gitlab.1sc.kz/infra/knp-manifests.git
    - cd knp-manifests/apps/isnaknpuser/prod
    - kustomize edit set image registry.1sc.kz/isnaknpuser=registry.1sc.kz/isnaknpuser:$CI_COMMIT_SHORT_SHA
    - git commit -am "Prod: isnaknpuser $CI_COMMIT_SHORT_SHA"
    - git push
  when: manual  # ← в прод только по кнопке
  only: [master]
```

Ключевые моменты:
- CI **не пушит в кластер напрямую** — только PR в manifests-repo.
- ArgoCD подхватывает и деплоит.
- Prod deploy — `when: manual`, кто-то жмёт кнопку осознанно.

---

## 10. Метрики зрелости CI/CD (DORA)

DevOps Research and Assessment — 4 ключевые метрики:

1. **Deployment Frequency** — как часто деплоите. Elite: несколько раз в день. Low: раз в месяц.
2. **Lead Time for Changes** — от commit до prod. Elite: < 1 час. Low: > 1 месяц.
3. **Change Failure Rate** — % деплоев вызывающих инциденты. Elite: 0-15%. Low: 46-60%.
4. **Time to Restore Service** — MTTR. Elite: < 1 час. Low: > 1 неделя.

Как поднять:
- **Frequency** и **Lead Time**: автоматизация, trunk-based, small PRs.
- **Change Failure Rate**: тесты, canary, feature flags.
- **MTTR**: observability, автоматический rollback, runbook'и.

---

## 11. Правила которые я вывел (опыт КНП)

- **Не мержь в пятницу** после 17:00. Если что-то сломается — некому чинить.
- **Не деплой прод без препрода**. Всегда staging environment зеркальный проду.
- **Feature flags для риска**. Новая обработка платежей? — обязательно за флагом.
- **Postmortem без blame**. После инцидента — что было, почему, как избежать. Без «кто виноват».
- **Runbook для каждого сервиса**. "Что делать если сервис X упал". 10 строк — как рестартить, как проверить health, куда смотреть.
- **Проверяй preprod после deploy**. `kubectl get pods` — все Ready? Smoke test через curl?
- **Alerting на бизнес-метрики, не на технические**. "Rate заказов упал на 30%" > "CPU 80% на pod X".
- **Идемпотентные миграции**. Если Job упал в середине — можно перезапустить.

---

## 12. Кратко

- **GitOps**: Git = единственный источник правды. Argo/Flux синхронизирует.
- **Deploy стратегии**: Rolling (стандарт), Blue-green (мгновенный rollback), Canary (постепенно с метриками), Shadow (тестирование на реальном трафике без риска).
- **БД**: только backward-compatible миграции. Expand → wait → contract.
- **Feature flags**: разделение deploy и release. Unleash / LaunchDarkly / Togglz.
- **Progressive delivery**: Argo Rollouts + Prometheus metrics = автоматический canary с rollback.
- **Secrets**: k8s Secrets → External Secrets (Vault) → Sealed Secrets (Git).
- **Rollback**: `git revert` в manifests → GitOps подхватит. Emergency — `kubectl rollout undo`.
- **DORA metrics**: deploy frequency, lead time, failure rate, MTTR.
- **Прод: не пятница, feature flags, canary, автоматический rollback, runbook'и**.
