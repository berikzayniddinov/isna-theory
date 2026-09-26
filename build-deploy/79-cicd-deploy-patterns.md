# 79. CI/CD и deploy-паттерны: GitOps, blue-green, canary, feature flags

## Зачем это знать

Как приложение попадает в прод — это половина инженерного качества системы. Первая половина — код. Вторая — как этот код доезжает до пользователей без сломанных деплоев, без даунтайма, без пятничных инцидентов, с возможностью откатиться за секунды. Это не «CI/CD автоматизация» в смысле «баш-скрипты запускают тесты». Это про модель мира: где живёт правда о том что задеплоено, кто может её менять, как выкатывается изменение, как ловится сбой и как откатывается.

Разница между командой, где деплой в прод — это «ох, страшно, только вечером» и командой, где деплой — это несколько раз в день без разговоров — не в размере команды и не в бюджете. В моделях. GitOps убирает вопрос «что реально задеплоено» — Git и есть кластер. Backward-compatible миграции убирают вопрос «а что если старая и новая версия работают параллельно». Feature flags разделяют деплой (код в проде) и релиз (пользователи видят) — это меняет саму физику раскатки. Progressive delivery через Argo Rollouts + Prometheus превращает деплой в самоуправляемый процесс: катится 10%, метрики хорошие — идёт дальше, плохие — откат без вмешательства.

Разберём: паттерны организации веток (trunk-based, GitFlow, GitLab flow) и когда какой имеет смысл. GitOps модель — что она даёт по сравнению с прямым kubectl apply из CI, как выглядит Argo CD, какие анти-паттерны стреляют в ногу. Стратегии deploy: rolling update (стандарт, но с ловушками при миграциях БД), blue-green (мгновенный откат за счёт двойного парка), canary (тонкая раскатка + метрики), shadow (тестирование на реальном трафике). Отдельно — работа с БД схемой: почему `RENAME COLUMN` в одном релизе стоит инцидента, и как правильно делать expand/contract. Feature flags: инструменты, паттерны, антипаттерны (флаги-мертвецы, флаги на всё подряд). Progressive delivery и связь с SLO/error budget. Секреты — от голых k8s Secret'ов до External Secrets Operator и Sealed Secrets. Rollback как отдельная дисциплина. DORA метрики как способ понять, где команда на кривой зрелости.

## Организация веток: как код доходит до релизной

**Trunk-based development** — все разработчики коммитят прямо в `main` (или `master`). Feature-ветки живут максимум часы, не дни. Каждый commit в trunk проходит CI и потенциально может уехать в прод. Незаконченные фичи прячутся за feature flags. Плюс — нет merge-hell, короткая обратная связь, все видят изменения друг друга сразу. Минус — требует зрелой команды, полного тестового покрытия и дисциплины по флагам. Google, Facebook, Netflix, все крупные SaaS работают именно так. Для маленьких команд это тоже часто оптимально: чем короче ветка, тем меньше шансов на конфликты и «а что там реально в этом ПР».

**GitFlow** — многолетний стандарт «правильной» работы: `main` → `develop` → множество `feature/*`, `release/*`, `hotfix/*` веток, регулярные merge'ы туда-сюда. Даёт чёткие релиз-циклы, но платит за это сложностью и постоянными конфликтами. Устарел для большинства современных команд, разумно оставаться на нём только в проектах с жёсткими релиз-циклами (embedded, регулируемые отрасли с квартальными релизами).

**GitHub Flow / GitLab Flow** — упрощённая версия: feature-branch с ПР, короткая жизнь (часы-дни), merge в main → деплой. Проще GitFlow, чуть менее строгий чем trunk-based. Хороший baseline для большинства команд без экстремального CI-покрытия.

**Модель КНП / ISNA** — гибрид с двумя постоянными ветками: `master` (прод) и `release` (препрод), плюс feature-ветки `release-<ticket>` для перекатки в release. Дополнительно — периодические cherry-pick'и `master → release` для инфра-фиксов, которые сначала сделали в master ради быстрого фикса, потом переносят на release чтобы не потерялись при следующем merge release → master. Даёт контроль что идёт в прод (не всё что в release уезжает в master), но требует дисциплины периодического переноса — иначе накапливается drift (в моём разборе как-то было 193 release-only коммита в sync). Внутри самих задач часто дублируется: то же изменение отдельно кладут и в release-ветку, и в master-ветку.

Выбор модели — про trade-off между гибкостью катки и контролем. Trunk-based максимально быстрый, но требует зрелых практик. Двухветочная модель КНП безопаснее в среде где прод и препрод должны отставать друг от друга, но накладывает на процесс постоянную работу по синхронизации.

## GitOps: Git как единственный источник правды

Модель «CI строит образ и делает kubectl apply из runner'а» кажется очевидной, но имеет серьёзные проблемы. У CI runner'а лежит kube-config с правами deploy — атака на runner превращается в атаку на кластер. Что реально задеплоено в кластер, не всегда совпадает с манифестами в Git: кто-то сделал `kubectl edit` для срочного фикса, забыл записать в git. Rollback = revert коммита + новый deploy из CI — минуты. Никакого аудита кто и когда изменил что.

**GitOps** переворачивает поток. В кластере живёт оператор (Argo CD, Flux), у него read-only pull-доступ к Git-репозиторию с манифестами. Раз в несколько минут (или по webhook) оператор сравнивает: что в Git → что в кластере. Если drift — синхронизирует. CI больше не имеет доступа к кластеру: он собирает образ, пушит в registry, коммитит новый image tag в manifests-repo. Дальше оператор сам подхватит.

```
Разработчик commit → CI собирает image + PR в manifests-repo → merge 
    → ArgoCD видит изменение → синхронизирует кластер
```

Три ключевых свойства этой модели. **Git всегда = кластер** — если что-то в кластере не совпадает с манифестами, оператор возвращает к манифестам (`selfHeal: true`). **CI не имеет прав в кластере** — атака на CI не даёт доступа к prod. **Аудит из коробки** — вся история изменений это git log, кто и когда что менял.

Пример Argo CD Application:

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
      prune: true       # удалять из кластера то что удалили из Git
      selfHeal: true    # возвращать вручную изменённое
    syncOptions:
    - CreateNamespace=true
```

Argo CD раз в 3 минуты (настраивается) или по webhook сравнивает состояние. UI показывает диаграмму всех приложений, sync status, health, историю деплоев с diff'ами. Rollback — `git revert` того коммита где меняли image tag, Argo подхватит.

**Anti-pattern'ы**, которые я много раз видел:

- **`kubectl apply` в CI после включения ArgoCD.** Argo вернёт назад в течение минут, вы будете гадать «почему deploy не применился». Правило: CI больше никогда не касается кластера напрямую.
- **Ручное `kubectl edit` в кластере как быстрый fix.** `selfHeal: true` вернёт через 3 минуты. Правило: любое изменение — через git.
- **Секреты в plaintext в Git.** Ни в коем случае, даже в «внутреннем» репозитории. Используй Sealed Secrets (шифрование публичным ключом controller'а) или External Secrets Operator (секреты живут в Vault/AWS Secrets Manager, оператор синхронизирует в k8s Secret).

**Flux** — альтернатива Argo CD от Weave, без встроенного UI (CLI + Grafana dashboards). Более легковесный, GitOps-первопроходец. Argo — приятный UI, легче войти для команды не знакомой с моделью. Flux — если UI не нужен, важна минималистичность. Оба production-ready.

## Deploy стратегии: rolling, blue-green, canary, shadow

Каждая стратегия — компромисс между скоростью деплоя, ресурсоёмкостью и blast radius при ошибке.

**Rolling update** — дефолт для Deployment в Kubernetes. Постепенная замена pod'ов новой версией: поднимаем один-два новых, ждём Ready, убиваем один-два старых, повторяем. Работает без даунтайма, встроено в Kubernetes, требует нуля дополнительных инструментов. Единственный существенный минус — во время раскатки работают **обе версии** одновременно. Значит либо API/схема БД должны быть backward-compatible между версиями (обычно да, для минорных релизов), либо ждать полного завершения rolling перед клиентскими вызовами по новому контракту (что реально невозможно в микросервисах). Разбор подробностей был в файле 77.

**Recreate** — противоположность rolling: убить все старые pod'ы, потом запустить все новые. Даунтайм гарантирован. Используется когда версии несовместимы (например, крупная миграция схемы БД, которая требует одновременного отсутствия старых клиентов) или для dev/staging окружений где даунтайм не важен.

**Blue-Green** — два параллельных полных парка. Blue — текущий прод, Green — новая версия. Оба живут параллельно, LoadBalancer/Ingress направляет 100% трафика на Blue. Deploy: раскатываем Green (v1.6) рядом с работающим Blue (v1.5), прогоняем smoke tests на Green напрямую (curl, health checks, дымовые проверки), переключаем LB на Green — секунды. Blue остаётся ещё сутки как rollback-мишень: если Green даёт проблемы, LB обратно на Blue за секунду. Через сутки Blue сносится.

```yaml
# два независимых Deployment
apiVersion: apps/v1
kind: Deployment
metadata: {name: isnaknpuser-blue}
spec:
  template:
    metadata:
      labels: {app: isnaknpuser, version: blue}
---
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
  selector: {app: isnaknpuser, version: blue}   # меняешь на green для переключения
```

Плюсы blue-green — мгновенный rollback (секунды), полноценный smoke test на реальной инфраструктуре до переключения, чистое переключение (все клиенты одновременно с v1.5 на v1.6, а не размазанное окно как в rolling). Минусы — **удвоение ресурсов** во время параллельного существования (что дорого для тяжёлых сервисов), и типичная головная боль с БД: обе версии одновременно подключены к БД, значит схема должна быть совместима с обеими.

**Canary** — деплой малой доли трафика на новую версию, наблюдение, постепенное увеличение процента. Классика: 5% → наблюдаем 30 минут → 25% → 50% → 100%. Если на любом шаге метрики ухудшились — откатываем в 0%. Разница с blue-green: canary тестирует новую версию на реальном трафике реальных пользователей, но с ограниченным blast radius.

В голом Kubernetes без service mesh — реализуется через два Deployment'а с разными replicas: `blue: 19, green: 1` даст примерно 5% трафика на green (в среднем, зависит от балансировки Service). Грубый инструмент, точный процент зависит от количества pod'ов и того как kube-proxy раскидывает соединения.

С Istio VirtualService — точная раскатка по процентам, доступ по headers/geolocation, sticky sessions:

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

Мощно, но Istio требует немалых ресурсов и погружения в архитектуру sidecar-mesh. Обычно оправдан только если mesh уже стоит для других целей (mTLS, distributed tracing).

**Argo Rollouts** и **Flagger** — специализированные расширения Kubernetes, автоматизирующие canary с автоматическим переключением по метрикам Prometheus. Отдельная тема ниже.

**Shadow / Mirror** — копия live-трафика идёт **и в blue, и в green**, но green **не отвечает клиенту**, только логирует и обрабатывает для наблюдения. Проверяешь read-heavy сервис (например, новую версию поисковика) на реальном трафике без риска для пользователей.

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

Полезно для рефакторинга: убеждаешься что новая реализация даёт **те же ответы** что старая на реальных запросах. Ключевая ловушка — mirror создаёт двойную нагрузку на downstream системы (БД, внешние API), надо это осознавать.

## Работа с БД схемой: expand/contract

Самая сложная часть zero-downtime deploy. Классическая ошибка: разработчик хочет переименовать колонку `name → full_name`.

Простой (и опасный) путь: миграция `RENAME COLUMN name TO full_name`, приложение переписывается на `full_name`. Что происходит в rolling update: старая версия v1 (ещё живая на нескольких pod'ах) обращается к колонке `name`. Миграция уже применена — `name` не существует. Все запросы v1 падают с ошибкой. Пять минут пока rolling не завершится — половина трафика падает.

**Expand/contract** — двухфазный (иногда трёхфазный) подход. Никогда не удаляй и не переименовывай в одном релизе. Разбить на:

**Release 1 (expand).** Миграция добавляет новую колонку и заполняет её:

```sql
ALTER TABLE users ADD COLUMN full_name text;
UPDATE users SET full_name = name;
```

Приложение v1.5 продолжает писать в `name` (по возможности — и в `name` и в `full_name` через триггер или dual-write в коде). Читает всё ещё `name` — старая схема работает.

**Release 2 (transition, недели-месяцы позже).** Приложение v1.6 переключается на чтение и запись `full_name`. Триггер или dual-write остаются на случай отката. Все клиенты уже v1.6+, никто не использует `name` в реальности.

**Release 3 (contract, ещё недели позже).** Миграция удаляет старую колонку:

```sql
ALTER TABLE users DROP COLUMN name;
```

Долго — да. Скучно — да. Безопасно — да. За эти недели все клиенты успели раскатиться на новую версию, никакого шанса что старая версия сломается.

Технически миграции лучше выделять в отдельный **Kubernetes Job**, не в initContainer основного Deployment'а. Причина: при `replicas > 1` все init'ы запустятся параллельно, Liquibase возьмёт advisory lock и остальные будут ждать — старт растянется, а при падении миграции все pod'ы уходят в CrashLoop одновременно.

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

Argo CD может делать миграции через `PreSync` hook — Job запускается до раскатки основного Deployment, следующий шаг ждёт его успеха. То же можно сделать через argo Workflows или Helm hooks.

Отдельный класс миграций — ALTER TABLE на большой таблице (разбор в файле 88). Даже безобидный `ADD COLUMN NOT NULL DEFAULT ...` может лечь на несколько минут под ACCESS EXCLUSIVE lock. Правило: всегда `SET lock_timeout` перед миграцией и всегда `CREATE INDEX CONCURRENTLY` в проде.

## Feature flags: отделить deploy от release

Feature flag (feature toggle) — конструкция вида «если флаг включен → выполнить новый код, иначе — старый». Ключевая идея — **отделить факт того что код в проде от факта что пользователи его видят**. Что это даёт:

- Постепенный rollout: 1% → 10% → 50% → 100% пользователей без пересборки/переката.
- A/B тестирование двух вариантов.
- Kill switch: если новая фича начала ломать — flip флаг, эффект мгновенный, откатка приложения не нужна.
- Персональный доступ для beta-тестеров, внутренних пользователей, конкретных клиентов.
- Дедлайны: код мержится задолго до релиза, живёт в проде за флагом, включается когда бизнес готов.

Простейшая реализация — обычный `@Value`:

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

Флаг из ConfigMap, изменение требует rolling restart pod'ов. Работает, но плата — минуты на переключение и рестарт.

Динамические флаги без рестарта — через **Spring Cloud Kubernetes Config** (подписывается на ConfigMap, refresh контекста без рестарта) или через **Consul** (у КНП уже развёрнут; можно хранить флаги там с `@RefreshScope`).

Специализированные системы:

- **Unleash** — open source. Сервер + Java SDK. Флаги по user_id, role, geo, %-rollout, датам. UI для управления. Real-time обновления через SSE.
- **LaunchDarkly** — SaaS, платный, индустриальный стандарт. WebSocket-обновления, feature dependencies, audit log, много интеграций.
- **Togglz** — легкая Java-библиотека, консольный UI бесплатный. Хорошо для маленьких проектов.

Пример с Unleash:

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

В UI Unleash: «включи `new_search` для 5% пользователей, начиная с завтра». Меняется без рестарта, без деплоя, без commit'а в git.

**Анти-паттерны**, за которые платят команды:

- **Флаги-мертвецы.** Флаг включили год назад на 100%, никто не удалил старый код. Через два года — в кодовой базе десятки веток if/else вокруг флагов, никто не знает какие ещё в работе. Правило: у каждого флага **дата expiration**, через 3 месяца — обязательный тикет удалить старую ветку.
- **Флаги в тестах.** Тесты гонятся при одном значении флага, забываете тестировать другую ветку. Значение переключаете в прод — падает то что не тестировалось. Настраивать матрицу: тесты бегут при `flag=true` и `flag=false`, обе версии проверяются.
- **Флаги на всё подряд.** Сложность растёт квадратично: 10 флагов = 1024 комбинации. Правило: флаги — для рискованных фич (новая обработка платежей, миграция кэша), не для косметики.

## Progressive delivery: canary + метрики + автоматика

Argo Rollouts и Flagger — это canary на автомате. Ты описываешь шаги («10% → пауза 5 минут → проверка метрик → 25% → пауза 10 минут → …») и critical метрики. Инструмент сам катит по шагам, при плохих метриках откатывает.

Пример Argo Rollout:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: {name: isnaknpuser}
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - analysis:
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

AnalysisTemplate — Prometheus-запрос с условием:

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

Что происходит: Rollout поднимает canary с 10% трафика, ждёт 5 минут, дальше Argo Analysis каждую минуту стучит в Prometheus, считает success_rate. Если ≥ 99% — шагаем дальше. Если 3 раза подряд ниже — abort и rollback автоматически. Deploy без вмешательства человека, но с безопасной проверкой на реальном трафике.

**Flagger** от Weave делает то же самое, часть Flux ecosystem. Выбор — по тому что уже используется в кластере (Argo CD → Argo Rollouts, Flux → Flagger).

Progressive delivery работает только при наличии **достаточно репрезентативных метрик**. Если сервис получает 10 запросов в минуту, 2-минутного окна не хватит для статистически значимого измерения — либо увеличивать интервалы, либо использовать более тонкие метрики.

## SLO, SLI, error budget

Отдельно от инструментов — концепция, которая связывает деплои с бизнесом.

- **SLI (Service Level Indicator)** — что измеряем. Success rate, p99 latency, availability.
- **SLO (Service Level Objective)** — целевое значение SLI за период. «success_rate ≥ 99.9% в квартал», «p99 latency < 300ms в месяц».
- **Error budget** — 100% минус SLO. Если SLO 99.9%, error budget = 0.1% времени можно быть down. За квартал (~90 дней) это ~1.3 часа даунтайма.

Как это работает в деплое: если error budget за период сгорел (много инцидентов) — **freeze deploy'ев** на оставшееся время. Все силы на стабилизацию. Google SRE использует эту практику с 2004 года: она объективна (метрика говорит, не мнение менеджера) и балансирует «быстро катить фичи» и «не разваливать прод».

Инструмент нашего масштаба — обычные Prometheus recording rules + alerting rules. Считаешь свою SLI, сравниваешь с SLO, alert когда error budget истощён.

## Секреты: где и как хранить

**Kubernetes Secrets** — базовый механизм. `kubectl create secret generic db-password --from-literal=password=xxx`. Хранится в etcd. Дефолтно — base64 (не шифрование, декодируется тривиально). `kubectl get secret ... -o yaml` покажет строку в base64, любой с правами `get secrets` в namespace получает секрет открытым текстом.

Первая мера — **etcd encryption at rest** (администратор кластера настраивает `EncryptionConfiguration`). Секреты шифруются при записи в etcd. RBAC ограничивает доступ. Это минимум для production.

Вторая мера — не хранить секреты в самом кластере. **External Secrets Operator** синхронизирует секреты из внешних систем (Vault, AWS Secrets Manager, GCP Secret Manager) в k8s Secret'ы. Разработчик описывает ExternalSecret:

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

Ротация: администратор меняет значение в Vault, ESO через час подтягивает обновление в k8s Secret. Deployment ссылается на Secret через `env.valueFrom.secretKeyRef` — сам под не перезапустится (K8s не отслеживает изменения содержимого Secret'а для монтированных env), но новый под получит новое значение. Для автоматического рестарта — annotation с hash содержимого секрета в deployment (Reloader controller это автоматизирует).

**Sealed Secrets** от Bitnami — для GitOps. Секрет шифруется публичным ключом controller'а, зашифрованный YAML лежит в Git. В кластере controller расшифровывает и создаёт обычный Secret.

```bash
kubeseal < secret.yaml > sealed-secret.yaml
git add sealed-secret.yaml
```

Плюс — секреты в Git безопасно (шифрование стойкое), работает с GitOps без внешних систем. Минус — приватный ключ controller'а критичен: потеря = все секреты не расшифровать, компромисс = раскрытие всех секретов. Резервировать ключ отдельно от Git.

## Rollback как отдельная дисциплина

Хороший CI/CD — это не только «катим быстро», но и «откатимся ещё быстрее если что». Три сценария:

**Автоматический через Argo Rollouts / Flagger.** Метрики плохие — откат за секунды, без человека. Идеал для тонких canary шагов.

**Через Git при GitOps.** `git revert HEAD` того коммита где меняли image tag, push. ArgoCD видит через минуту, синхронизирует. Всё в git-истории, аудит есть.

**`kubectl rollout undo`.** Быстро, но не в Git. При GitOps ArgoCD откатит обратно к манифесту, значит эффект временный. Использовать только как emergency workaround, следом обязательно git revert.

Отдельно тяжёлый случай — когда БД мигрирована и rollback приложения несовместим с новой схемой. Варианты действий:

1. **Быстрая обратная миграция.** Если добавили колонку — drop. Если изменили тип — обратно. Работает только для простых, обратимых миграций.
2. **Hotfix новой версии.** Патч приложения, который умеет работать с новой схемой, но безопасен. Часто через feature flag: отключить проблемную функциональность через флаг, деплой быстрого хотфикса.
3. **Downtime + restore из бэкапа.** Последнее средство. Всегда доступно, всегда болезненно.

**Правильный подход — не допускать этой ситуации.** Backward-compatible миграции + expand/contract pattern (см. выше). Правило: любой rollback приложения должен работать с текущей схемой БД. Если это нарушается — миграция должна быть не сейчас, а после того как код обеих версий закоммичен.

## Пример CI/CD пайплайна для микросервиса

Упрощённый `.gitlab-ci.yml` для сервиса типа isnaknpuser:

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
    - gradle dependencyCheckAnalyze
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
    - git clone https://gitlab.1sc.kz/infra/knp-manifests.git
    - cd knp-manifests
    - kustomize edit set image registry.1sc.kz/isnaknpuser=registry.1sc.kz/isnaknpuser:$CI_COMMIT_SHORT_SHA
    - git commit -am "Update isnaknpuser to $CI_COMMIT_SHORT_SHA"
    - git push
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
  when: manual
  only: [master]
```

Ключевые моменты. CI **не пушит в кластер напрямую**, никаких `kubectl` — только PR в manifests-repo, ArgoCD подхватывает. Security scan (Trivy для образа и dependency-check для jar'а) в pipeline с `allow_failure: false` — критические уязвимости блокируют раскатку. Prod deploy — `when: manual`, кто-то жмёт кнопку осознанно после проверки на preprod.

## DORA метрики: где команда на кривой зрелости

DevOps Research and Assessment — четыре ключевые метрики, которые Google DORA team исследовала на тысячах команд. Показывают уровень зрелости CI/CD:

**Deployment Frequency** — как часто деплоите. Elite: несколько раз в день на каждый сервис. High: раз в день - неделю. Medium: раз в неделю - месяц. Low: раз в месяц-полгода.

**Lead Time for Changes** — от commit до prod. Elite: < 1 час. High: 1 день - 1 неделя. Medium: 1 неделя - месяц. Low: > 1 месяца.

**Change Failure Rate** — % деплоев вызывающих инциденты (rollback, hotfix). Elite: 0-15%. High-Medium: 16-30%. Low: > 46%.

**Time to Restore Service** — MTTR после инцидента. Elite: < 1 час. High: < 1 день. Medium: < 1 неделя. Low: > 1 месяца.

Как поднимать каждую:

- Frequency и Lead Time поднимаются автоматизацией, trunk-based development'ом, small PR'ами, коротким CI (< 15 минут).
- Change Failure Rate — тестами, feature flags, canary, smoke tests в pipeline.
- MTTR — observability (Prometheus + Grafana + Loki), автоматический rollback, runbook'и (короткие мануалы «что делать если сервис X упал»).

Elite команды деплоят в прод несколько раз в день, но при этом имеют более низкий Change Failure Rate чем Low команды — потому что маленькие изменения проще проверять, меньший blast radius при ошибке.

## Правила прод-практики (опыт)

- **Не мержь в пятницу после 17:00.** Если что-то сломается — некому чинить. У многих команд правило «Read Only Friday» — только критичные хотфиксы.
- **Не деплой прод без препрода.** Всегда staging environment зеркальный проду. Deploy сначала туда, интеграционные тесты, только потом прод.
- **Feature flags для риска.** Новая обработка платежей, новая миграция кэша, эксперимент с производительностью — обязательно за флагом.
- **Postmortem без blame.** После инцидента — что было, почему, как избежать повторения. Без «кто виноват». Виноват процесс, который позволил случиться.
- **Runbook для каждого сервиса.** Короткий мануал 10-20 строк: «что делать если сервис X упал». Как рестартить, как проверить health, куда смотреть в первую очередь. Часть репозитория сервиса.
- **Проверяй preprod после deploy автоматически.** Smoke tests в pipeline: `kubectl get pods` — все Ready? `curl /health` возвращает 200? Не полагаться на «наверное всё ок раз rollout прошёл».
- **Alerting на бизнес-метрики, не на технические.** «Rate заказов упал на 30%» важнее чем «CPU 80% на pod X» — CPU 80% может быть нормой, а падение заказов — реальный инцидент.
- **Идемпотентные миграции.** Если Job упал в середине — должен уметь перезапуститься с того же места. Liquibase из коробки идемпотентен (по changelog history), Flyway тоже.
- **Все изменения через git.** Даже emergency workaround — сначала edit, потом сразу commit в git.

## Заключение

CI/CD — это не «баш-скрипты для тестов», а модель того как код живёт в проде. Ветки — про баланс скорости и контроля: trunk-based максимально быстрый, GitFlow тяжёл, гибридные модели (типа master/release КНП) требуют дисциплины синхронизации.

GitOps убирает целый класс проблем: Git = кластер, CI не имеет доступа к кластеру, оператор синхронизирует. Argo CD или Flux — оба production-ready, выбор по вкусу UI. Правило: после включения GitOps никаких `kubectl apply` из CI, никаких `kubectl edit` руками. Все изменения через git.

Deploy стратегии — компромиссы. Rolling — стандарт, требует backward compatibility. Blue-green — мгновенный rollback ценой удвоения ресурсов, головная боль с БД. Canary — тонкая раскатка на реальный трафик, минимальный blast radius, требует Istio или Argo Rollouts. Shadow — тест на реальном трафике без риска для пользователей, но двойная нагрузка на downstream.

БД миграции — самая сложная часть zero-downtime. Правило: expand/contract, никогда rename в одном релизе, миграции отдельным Job'ом (не initContainer), большие ALTER TABLE с `lock_timeout`.

Feature flags разделяют deploy и release. Инструменты — от `@Value` до Unleash/LaunchDarkly. Анти-паттерны — флаги-мертвецы (без даты expiration), флаги на всё подряд (взрыв сложности).

Progressive delivery через Argo Rollouts + Prometheus — deploy на автопилоте с безопасным откатом по метрикам. SLO / SLI / error budget связывают деплои с бизнесом: сгорел budget — freeze раскаток.

Секреты — не в plaintext в git, минимум etcd encryption at rest, лучше External Secrets Operator с Vault или Sealed Secrets для GitOps.

Rollback — отдельная дисциплина. Автоматический (Rollouts/Flagger), через git revert при GitOps, `kubectl rollout undo` как emergency. Никогда не деплоить миграцию, которая не позволит откатить приложение.

DORA метрики (Deployment Frequency, Lead Time, Change Failure Rate, MTTR) — объективный способ понять уровень команды. Elite команды деплоят чаще и падают реже одновременно — маленькие изменения проще проверять.

Правила прода не про технологии, про дисциплину: не мержить в пятницу, feature flags для риска, postmortem без blame, runbook для каждого сервиса, alerting на бизнес-метрики. Технологии решают половину; вторая — практики команды.
