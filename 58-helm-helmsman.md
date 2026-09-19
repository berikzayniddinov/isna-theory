# 58. Helm + Helmsman

Package manager для K8s + декларативный orchestrator.

---

## 1. Проблема

K8s манифесты — статичные YAML. Для 30 микросервисов:
- 30 × Deployment.yaml
- 30 × Service.yaml
- 30 × ConfigMap.yaml
- + Ingress, HPA, etc.

Каждый со своей конфигурацией. Много копипасты. Изменение общего паттерна → правки во всех.

Плюс: разные env (dev/prod) — разные ресурсы, replicas, images. Дублировать всё?

**Helm решает** — templating + values.

---

## 2. Что такое Helm

**Helm** — package manager для K8s (как apt/yum/brew).

Единица распространения — **Chart** (пакет).

Возможности:
- Templating YAML (Go templates).
- Values files для параметризации.
- Установка / обновление / rollback.
- Repositories (публичные + private).
- Dependencies между charts.
- Hooks (pre-install, post-upgrade).

---

## 3. Структура Chart

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

### 3.1 Chart.yaml

```yaml
apiVersion: v2
name: isna-knp
description: КНП микросервис
version: 1.0.42                  # версия chart'а
appVersion: 1.0.42               # версия приложения
maintainers:
  - name: berik
    email: berikluv@gmail.com
dependencies:
  - name: postgresql
    version: 12.5.0
    repository: https://charts.bitnami.com/bitnami
```

### 3.2 values.yaml

Default values:
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

### 3.3 templates/deployment.yaml

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

### 3.4 templates/_helpers.tpl

Переиспользуемые фрагменты:
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

---

## 4. Go templating

Helm использует **Go templates**.

### 4.1 Значения

- `{{ .Values.replicaCount }}` — из values.yaml.
- `{{ .Chart.Name }}` — из Chart.yaml.
- `{{ .Release.Name }}` — имя release.
- `{{ .Release.Namespace }}`.

### 4.2 Функции

```
{{ .Values.name | upper }}
{{ .Values.name | quote }}
{{ .Values.name | default "unknown" }}
{{ toYaml .Values.resources | indent 4 }}
{{ include "chart.helper" . }}
```

Много встроенных: `upper`, `lower`, `quote`, `trim`, `replace`, `split`, `join`.

### 4.3 Условия

```
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
{{- end }}
```

### 4.4 Циклы

```
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}
```

### 4.5 Whitespace control

- `{{-` — обрезать whitespace слева.
- `-}}` — справа.

Иначе — много пустых строк в результате.

---

## 5. Install / upgrade / rollback

### 5.1 Install

```bash
helm install my-release ./mychart
helm install my-release ./mychart -f values-prod.yaml
helm install my-release ./mychart --set replicaCount=5,image.tag=1.0.43
```

Каждая install = **новый release**.

### 5.2 Upgrade

```bash
helm upgrade my-release ./mychart --set image.tag=1.0.44
```

Обновляет existing release.

### 5.3 Rollback

```bash
helm history my-release            # список revisions
helm rollback my-release 3         # к revision 3
```

### 5.4 Uninstall

```bash
helm uninstall my-release
```

Удаляет все K8s объекты релиза.

### 5.5 Debug / dry-run

```bash
helm template my-release ./mychart -f values.yaml    # только генерация, без apply
helm install my-release ./mychart --dry-run --debug
helm lint ./mychart                                   # валидация
```

---

## 6. Repositories

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo postgres
helm install pg bitnami/postgresql
```

Публичные:
- **Bitnami** — apps.
- **artifacthub.io** — search.

Private: nginx с basic auth, chartmuseum, harbor.

---

## 7. Dependencies

Chart может зависеть от других:

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

Позволяет собирать сложные приложения.

---

## 8. Hooks

Выполнить job в определённой фазе:

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

Полезно для:
- DB migrations перед install.
- Backup перед upgrade.
- Cleanup после uninstall.

Фазы: pre-install, post-install, pre-upgrade, post-upgrade, pre-delete, post-delete.

---

## 9. Helm 2 vs 3

**Helm 2** — с Tiller (server-side агент в K8s). Deprecated с 2020.

**Helm 3** — client-only, использует K8s API напрямую. RBAC как обычно.

Всегда Helm 3.

---

## 10. Helmsman

**Helmsman** — Layer над Helm. Декларативное описание desired state releases в K8s.

### 10.1 Проблема

Helm сам — imperative: `helm install`, `helm upgrade`. Хочется **GitOps**: описал в git → синхронизировалось в K8s.

### 10.2 Что даёт

Один YAML описывает **все releases** кластера:

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

Helmsman:
1. Читает current state (что установлено в K8s).
2. Сравнивает с desired state (yaml).
3. Применяет разницу: install / upgrade / uninstall.

### 10.3 Плюсы

- **GitOps-friendly**: yaml в git → CI applies.
- Один файл — весь кластер.
- Diff и dry-run.
- Priority (порядок install).

### 10.4 В ИСНА

Из memory `knp-form-hz5-actuator-cache-nosuchmethod` — упоминается `helmsman-knp-form` — конфиг для isna-form.

`helmsman-knp-form!135` (MR).

Т.е. в ИСНА для управления deploy используется helmsman.

---

## 11. Альтернативы Helmsman

### 11.1 ArgoCD / Flux

**GitOps operators**. Живут в кластере, следят за git → синхронизируют.

Плюсы: continuous sync (drift detection), UI, hooks.

Более enterprise.

### 11.2 Terraform + Helm Provider

Terraform умеет управлять Helm releases.

Плюс: если у тебя уже Terraform для infrastructure — унифицированно.

### 11.3 Custom scripts

Bash скрипты вокруг `helm install`. Просто, но не декларативно.

---

## 12. Helm vs Kustomize

**Kustomize** — альтернативный подход. Не templating, а **overlays**.

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

`kustomization.yaml` в overlay:
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

Плюсы:
- Нет template engine (чище).
- Встроен в kubectl (`kubectl apply -k`).

Минусы:
- Меньше выразительности.
- Не так удобно для чужих apps.

**Правило**: Helm для чужих apps (postgres, kafka); Kustomize для своих (простая параметризация).

Часто **вместе**: Helm для install, Kustomize для env-specific overlays.

---

## 13. Best practices

1. **Один chart на microservice**.
2. **values.yaml** для defaults + values-{env}.yaml для env-specific.
3. **_helpers.tpl** для переиспользуемого.
4. **Version chart independently** от app version.
5. **Chart.yaml appVersion = git tag** приложения.
6. **helm lint** в CI.
7. **helm template** dry-run — smoke check.
8. **Hooks** для migrations.
9. **Не hardcode namespace** — использовать `.Release.Namespace`.
10. **Immutable secrets** — использовать SealedSecrets или ExternalSecrets.
11. **Test releases** через `helm test`.

---

## 14. Пример полного deploy flow ИСНА

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

---

## 15. Собесные вопросы

1. **Что такое Helm?** — Package manager для K8s.
2. **Что такое chart?** — Пакет (templates + values + Chart.yaml).
3. **Разница values.yaml и Chart.yaml?** — Values — data (replicaCount, image); Chart — metadata (name, version).
4. **Как передать env-specific values?** — `helm install ... -f values-prod.yaml`.
5. **Что делает `helm upgrade`?** — Обновляет existing release новыми values / chart version.
6. **helm rollback — как?** — `helm history` + `helm rollback <release> <revision>`.
7. **Что такое hooks?** — Jobs в определённой фазе (pre-install, post-upgrade).
8. **Helm 2 vs 3?** — Helm 2 с Tiller (server); Helm 3 client-only.
9. **Что такое Helmsman?** — Declarative orchestrator над Helm; описываешь все releases в yaml.
10. **Helm vs Kustomize?** — Helm: templating + package management; Kustomize: overlays без templating.
11. **GitOps — зачем?** — Git = source of truth для infrastructure; ArgoCD/Flux sync.
12. **helm template — что делает?** — Генерирует YAML без apply (dry-run).
13. **_helpers.tpl — что?** — Переиспользуемые фрагменты (labels, names).
14. **Sub-charts?** — Dependencies других charts; composition сложных apps.
15. **`.Release.Name` — что?** — Имя release, задаётся при `helm install <name>`.

---

## Итог

- **Helm** = package manager K8s; chart = пакет.
- **Templates + values** для параметризации.
- **helm install/upgrade/rollback** — базовые операции.
- **Helmsman** — declarative layer над Helm (GitOps).
- **Alternatives**: ArgoCD, Flux, Kustomize.
- В ИСНА: **helmsman** для управления releases.

Следующий — `59-consul-deep-dive.md`.
