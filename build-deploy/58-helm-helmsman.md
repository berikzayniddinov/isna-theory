# 58. Helm plus Helmsman: package management и declarative deploy для K8s

## Зачем нужен Helm

Kubernetes manifest files — статичные YAML. Deployment.yaml, Service.yaml, ConfigMap.yaml, Ingress.yaml. Один сервис — 5-10 файлов. Для одного применения normal — manually managed.

Разработчик начинающий K8s adventure обычно пишет manifests вручную. Copies и modifies для different services. Copies и modifies для different environments (dev/staging/prod). Копипаста растёт. Ошибки при sync между копиями. Изменение общего pattern — правки в десятках файлов.

Разница между разработчиком «пишущим yamls» и «понимающим Helm» очевидна при scale. Первый застревает в 30 микросервисах × 3 environments = 90 sets manifests все subtly different. Второй знает Helm as package manager — Chart parameterized через values.yaml. One chart definition, multiple deployments through different values files. Изменение общего patterns — one place. Environments — different values files. Rollback через `helm rollback`. Team convention across all services.

В этом файле разберём Helm и Helmsman глубоко. Проблема deployment automation. Helm как package manager. Chart structure и content. Go templating. Install/upgrade/rollback operations. Repositories и dependencies. Hooks для lifecycle events. Helm 2 vs Helm 3 differences. Helmsman как declarative layer. Alternatives (ArgoCD, Flux, Terraform, Kustomize). Full deployment flow ИСНА. Best practices.

## Проблема

30 микросервисов. Каждый нужен. 30 × Deployment.yaml. 30 × Service.yaml. 30 × ConfigMap.yaml. Plus Ingress, HPA (Horizontal Pod Autoscaler), etc.

Каждый со своей конфигурацией. Много копипасты. Изменение общего pattern (например добавить общий label к всем) требует правки во всех файлах.

Плюс. Разные environments (dev/prod) — разные ресурсы, replicas, images. Дублировать всё? Manageable для 5 services может, для 50 — nightmare.

Helm решает через templating plus values files. Parameterization mechanism enabling one source of truth reused with different configurations.

## Что такое Helm

Helm — package manager для K8s. Analog apt/yum/brew но для Kubernetes resources.

Единица распространения — Chart (пакет). Bundles related K8s resources plus templating logic plus default values.

Возможности. Templating YAML (Go templates). Values files для параметризации. Установка, обновление, rollback. Repositories (публичные plus private). Dependencies между charts (sub-charts). Hooks (pre-install, post-upgrade).

Emerged из Deis (acquired by Microsoft). Now CNCF project. De facto standard для K8s package management.

## Структура Chart

Standard directory layout:
```
mychart/
├── Chart.yaml                  ← метаданные
├── values.yaml                 ← default values
├── charts/                     ← sub-charts (dependencies)
├── templates/                  ← YAML шаблоны K8s объектов
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl            ← переиспользуемые фрагменты
│   ├── NOTES.txt               ← сообщение после install
│   └── tests/                  ← тесты для helm test
└── README.md
```

Chart.yaml — метаданные:
```yaml
apiVersion: v2
name: isna-knp
description: КНП микросервис
version: 1.0.42                  # версия chart
appVersion: 1.0.42               # версия приложения
maintainers:
  - name: berik
    email: berikluv@gmail.com
dependencies:
  - name: postgresql
    version: 12.5.0
    repository: https://charts.bitnami.com/bitnami
```

Version and appVersion могут отличаться. version — chart version (при изменении templates). appVersion — application version (business logic).

values.yaml — default values:
```yaml
replicaCount: 3

image:
  repository: nexus.isna/isna-knp
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: "2"
    memory: 1Gi

ingress:
  enabled: true
  host: knp.kgd.gov.kz

env:
  SPRING_PROFILES_ACTIVE: prod
  CONSUL_HOST: consul.isna

postgresql:
  enabled: false                 # используем внешний
```

Default configuration. Overridden при install через -f other-values.yaml или --set flags.

templates/deployment.yaml — templated K8s manifest:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "isna-knp.fullname" . }}
  labels:
    {{- include "isna-knp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "isna-knp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "isna-knp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: {{ .Values.service.port }}
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: {{ .Values.service.port }}
```

Template mixes static YAML с Go template directives. {{ }} для value substitution. include for reusable fragments. range для iteration. toYaml для inline complex values.

templates/_helpers.tpl — переиспользуемые фрагменты:
```yaml
{{/*
Expand chart name.
*/}}
{{- define "isna-knp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{/*
Full name.
*/}}
{{- define "isna-knp.fullname" -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{/*
Common labels.
*/}}
{{- define "isna-knp.labels" -}}
app.kubernetes.io/name: {{ include "isna-knp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

Named templates. Reusable across multiple manifests. DRY принцип. Labels consistent across all objects created by chart.

## Go templating detailed

Helm uses Go templates syntax. Standard Go feature.

Значения. {{ .Values.replicaCount }} — из values.yaml. {{ .Chart.Name }} — из Chart.yaml. {{ .Release.Name }} — имя release. {{ .Release.Namespace }}.

Functions. Many built-in для value manipulation:
```
{{ .Values.name | upper }}
{{ .Values.name | quote }}
{{ .Values.name | default "unknown" }}
{{ toYaml .Values.resources | indent 4 }}
{{ include "chart.helper" . }}
```

Много встроенных. upper, lower, quote, trim, replace, split, join, contains, hasPrefix, hasSuffix. Sprig library adds even more.

Conditions:
```
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
{{- end }}
```

Conditional resource generation. Ingress created только if enabled.

Loops:
```
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}
```

Iteration через maps или slices. Dynamic content generation.

Whitespace control. {{- обрезать whitespace слева. -}} справа. Иначе — много пустых строк в результате. Templates могут output valid YAML plus readable structure требует attention к whitespace.

## Install, upgrade, rollback

Install:
```bash
helm install my-release ./mychart
helm install my-release ./mychart -f values-prod.yaml
helm install my-release ./mychart --set replicaCount=5,image.tag=1.0.43
```

Каждый install = новый release с unique name. Multiple releases того же chart возможны (different names).

Upgrade existing release:
```bash
helm upgrade my-release ./mychart --set image.tag=1.0.44
```

Updates existing release с new values или chart version. Creates new revision.

Rollback к предыдущей revision:
```bash
helm history my-release            # список revisions
helm rollback my-release 3         # к revision 3
```

Helm maintains history all revisions. Enables quick recovery from bad deployments.

Uninstall:
```bash
helm uninstall my-release
```

Удаляет все K8s объекты релиза. Clean removal.

Debug и dry-run:
```bash
helm template my-release ./mychart -f values.yaml    # только генерация, без apply
helm install my-release ./mychart --dry-run --debug
helm lint ./mychart                                   # валидация
```

helm template outputs rendered YAML без applying к cluster. Useful для inspecting what will be created. Dry-run applies same logic но не commits changes.

lint validates chart structure и common issues.

## Repositories

Repositories для sharing charts:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo postgres
helm install pg bitnami/postgresql
```

Public repositories доступны:
- Bitnami — comprehensive apps catalog.
- artifacthub.io — searchable index across many repos.

Private repositories. Nexus с basic auth. ChartMuseum. Harbor. For internal charts организации.

## Dependencies

Chart может зависеть от других charts:
```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: 12.5.0
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

```bash
helm dependency update ./mychart    # скачивает subcharts
```

Позволяет собирать сложные приложения. Chart может include databases, caches, other required services как sub-charts.

Condition — conditional inclusion based on values. postgresql.enabled=true — include PostgreSQL sub-chart. false — skip.

## Hooks

Execute jobs в определённой фазе:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp/migrate:{{ .Chart.AppVersion }}
```

Полезно для. DB migrations перед install. Backup перед upgrade. Cleanup после uninstall.

Фазы. pre-install, post-install. pre-upgrade, post-upgrade. pre-delete, post-delete.

hook-weight для ordering multiple hooks. hook-delete-policy для cleanup lifecycle.

Common pattern — database migrations как pre-upgrade hook. Ensures schema up-to-date before new application version deploys.

## Helm 2 vs 3

Helm 2 — с Tiller (server-side агент в K8s). Central component processing helm commands. Security concerns — Tiller had cluster-wide access. Deprecated с 2020.

Helm 3 — client-only. Uses K8s API напрямую. RBAC как обычно — helm command uses кубернетовские credentials. Simpler security model.

Всегда Helm 3. No reason для Helm 2 в new projects. Migration путь documented для existing Helm 2 users.

## Helmsman

Helmsman — layer над Helm. Декларативное описание desired state releases в K8s.

Проблема. Helm сам — imperative. helm install, helm upgrade commands. Хочется GitOps — описал в git, синхронизировалось в K8s.

Что даёт. Один YAML описывает все releases кластера:
```yaml
# helmsman.yaml
context: prod

namespaces:
  knp:
    protected: true
  fno:

helmRepos:
  bitnami: https://charts.bitnami.com/bitnami
  isna: https://nexus.isna/repository/helm

apps:
  isna-knp:
    namespace: knp
    enabled: true
    chart: isna/isna-knp
    version: 1.0.42
    valuesFile: values/knp-prod.yaml
    priority: -10

  isna-fno:
    namespace: fno
    enabled: true
    chart: isna/isna-fno
    version: 2.1.0
    valuesFile: values/fno-prod.yaml
```

Запуск:
```bash
helmsman --apply -f helmsman.yaml
```

Helmsman. Читает current state (что установлено в K8s). Сравнивает с desired state (yaml). Применяет разницу — install / upgrade / uninstall.

Плюсы. GitOps-friendly — yaml в git, CI applies. Один файл — весь кластер. Diff и dry-run capabilities. Priority (порядок install) для dependencies между releases.

В КНП. Из memory knp-form-hz5-actuator-cache-nosuchmethod — упоминается helmsman-knp-form — конфиг для isna-form. helmsman-knp-form!135 (MR). В ИСНА для управления deploy используется helmsman.

## Альтернативы Helmsman

ArgoCD / Flux. GitOps operators. Живут в кластере, следят за git — синхронизируют.

Плюсы. Continuous sync (drift detection). UI (ArgoCD особенно good UI). Hooks. More enterprise-y features.

Более enterprise. Complex setup. Better для sophisticated GitOps workflows.

Terraform plus Helm Provider. Terraform умеет управлять Helm releases.

Плюс — если у тебя уже Terraform для infrastructure — унифицированно. One tool для всей infrastructure plus applications.

Custom scripts. Bash скрипты вокруг helm install. Просто, но не декларативно. Sometimes appropriate для simple setups но not scalable.

## Helm vs Kustomize

Kustomize — альтернативный подход. Не templating, а overlays.

Structure:
```
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml

overlays/
├── dev/
│   └── kustomization.yaml
└── prod/
    └── kustomization.yaml
```

kustomization.yaml в overlay:
```yaml
bases:
  - ../../base
patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |
      - op: replace
        path: /spec/replicas
        value: 5
```

Плюсы. Нет template engine (чище). Встроен в kubectl (kubectl apply -k). YAML остаётся valid YAML — не intermixed с template directives.

Минусы. Меньше выразительности. Не так удобно для чужих apps. Complex customization может стать messy с multiple patches.

Правило. Helm для чужих apps (postgres, kafka). Kustomize для своих (простая параметризация).

Часто вместе. Helm для install. Kustomize для env-specific overlays. Hybrid approach leveraging strengths обоих.

## Best practices

Один chart на microservice. Standard convention. Easier maintain independently.

values.yaml для defaults plus values-{env}.yaml для env-specific. Clean separation defaults и environment overrides.

_helpers.tpl для переиспользуемого. DRY principle. Consistent labels, names across manifests.

Version chart independently от app version. Chart может update без app version change (например adding new template).

Chart.yaml appVersion equal git tag приложения. Traceability между chart и application source.

helm lint в CI. Catches structural issues early.

helm template dry-run — smoke check. Verify template produces valid YAML.

Hooks для migrations. Standard pattern.

Не hardcode namespace. Использовать .Release.Namespace. Chart deployable к любому namespace.

Immutable secrets. Использовать SealedSecrets или ExternalSecrets. Not committing secrets to git даже encrypted formats sometimes leak.

Test releases через helm test. Chart tests validate deployment succeeded functionally.

## Full deploy flow ИСНА

Реальный workflow:
```
1. Developer commits в isna-knp repo
2. CI собирает Docker image → nexus:isna-knp:1.0.42
3. CI обновляет Chart.yaml: appVersion: 1.0.42
4. Chart публикуется в chart repository nexus:isna
5. helmsman.yaml обновляется: isna-knp.version: 1.0.42
6. `helmsman --apply -f helmsman-prod.yaml`
7. Helm upgrade → K8s rollout
8. Pre-upgrade hook: Liquibase migration
9. Rolling update pods
10. Health checks → old pods убиваются
```

End-to-end automation. Developer commits code, everything else automatic. GitOps principles applied.

Multiple stages. Image build. Chart update. Helmsman configuration. Deployment. Migration. Rollout. Health verification. Each stage может fail и trigger rollback.

## Итоги

Helm — package manager для K8s. Charts bundle related resources plus templating logic plus default values.

Chart structure — Chart.yaml (metadata), values.yaml (defaults), templates/ (K8s manifests с Go templates), charts/ (dependencies).

Go templates для parameterization. Values substitution через {{ .Values }}. Functions, conditions, loops для dynamic content. Whitespace control critical для valid YAML output.

install / upgrade / rollback operations. Revisions maintained для history. Easy recovery from bad deployments.

Repositories для sharing. Public (Bitnami, artifacthub.io) plus private (Nexus, Harbor).

Dependencies через sub-charts. Complex apps composable из pieces.

Hooks для lifecycle events. Pre/post install/upgrade/delete. Common для migrations, backups.

Helm 3 — client-only, uses K8s API directly. Helm 2 (Tiller-based) deprecated.

Helmsman — declarative layer over Helm. GitOps-friendly. Один yaml описывает весь cluster state.

Alternatives Helmsman. ArgoCD, Flux (K8s-native GitOps operators). Terraform (unified infra tool). Custom scripts.

Helm vs Kustomize. Templating vs overlays. Different philosophies. Often used together.

Best practices. One chart per service. Separate defaults и env-specific values. Helper templates. Version independence. Lint и template в CI. Migrations как hooks. No hardcoded namespaces.

Real ИСНА workflow demonstrates full GitOps deployment automation.

Дальше — Consul deep dive с Raft internals, gossip mechanics, ACL, multi-DC.
