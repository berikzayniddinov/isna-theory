# 10. Kubernetes: оркестрация контейнеров

## Зачем нужен Kubernetes

Один Docker container хорошо для разработки и простых деплоев. Но реальная микросервисная production — 30-50 сервисов, каждый в 3+ репликах, с обновлениями без downtime, балансировкой, автоскейлингом, конфигурацией, секретами, мониторингом здоровья.

Управлять этим вручную невозможно. Ты запустил 150 контейнеров через `docker run`. Один упал — нужно поднять. Обновляешь версию — нужно постепенно заменить старые контейнеры новыми без потери трафика. Нужно балансировать трафик между репликами. Нужно прокинуть конфиги. Нужно масштабировать при нагрузке. Нужно обновлять сотни манифестов при изменении. Без оркестратора это невозможно.

**Kubernetes** — стандарт индустрии для оркестрации контейнеров. Разработан Google на базе внутренней системы Borg, open-source с 2014 года. Сейчас — де-факто стандарт для production микросервисов.

Основная идея Kubernetes — **desired state**. Ты описываешь желаемое состояние: "хочу 3 реплики сервиса isnaknpintegration версии 1.0.42 с 1 GB памяти". K8s следит чтобы это состояние поддерживалось. Реплика упала — K8s поднимает новую. Нода умерла — K8s переносит поды на другие. Ты обновил версию — K8s постепенно заменяет старые новыми.

Знание K8s для senior Java-разработчика в микросервисной архитектуре обязательно. Ты не только пишешь код — понимаешь как он деплоится, как настроен health check, как масштабируется, как диагностируется. В КНП все сервисы работают в Kubernetes, любая работа с prod требует K8s knowledge.

В этом файле разберём Kubernetes на достаточном уровне для практической работы. Архитектура кластера (control plane, worker nodes). Основные ресурсы (Pod, Deployment, Service, Namespace, ConfigMap, Secret). Networking. Probes для health checks. Rolling update и rollback. Practical диагностика через kubectl.

## Архитектура кластера

Kubernetes cluster — группа физических или виртуальных машин, работающих вместе. Разделяется на две части.

**Control Plane** (master) — мозг кластера. Обычно на отдельных нодах, 3+ реплики для HA. Не запускает пользовательские приложения — только системные компоненты K8s.

**Worker nodes** — где реально работают пользовательские поды. Обычные машины с Docker (или containerd) и K8s агентом.

Компоненты Control Plane.

**kube-apiserver** — единственный вход в кластер. Все команды (kubectl, controllers, kubelet) идут через REST API apiserver. Валидирует запросы, аутентифицирует, авторизует. Читает и пишет состояние в etcd.

**etcd** — распределённая KV-БД на основе Raft consensus. Хранит **всё состояние кластера**: манифесты объектов, статусы, секреты, ConfigMaps. Обычно 3 или 5 нод для HA. Если etcd упал целиком — кластер не управляем, но существующие поды продолжают работать (kubelet-ы имеют закэшированное состояние).

**kube-scheduler** — планировщик. Смотрит "есть Pod без назначенной ноды" и решает на какой ноде запустить. Учитывает: ресурсы (CPU/memory на нодах), taints и tolerations, node selectors, affinity/anti-affinity правила.

**kube-controller-manager** — набор контроллеров. Каждый следит за своим типом ресурсов и приводит реальное состояние к желаемому. Deployment controller держит N реплик. ReplicaSet controller следит за подами. Node controller замечает мёртвые ноды и эвакуирует с них поды. Endpoints controller обновляет endpoints Service. Job/CronJob controllers. Принцип **reconciliation loop** — постоянное сравнение spec (желаемое) с status (реальное), приведение status к spec.

**cloud-controller-manager** — интеграция с облачным API (AWS, GCP, Azure). Создание LoadBalancer, disks, node lifecycle. Для on-premise кластеров не нужен.

Компоненты Worker node.

**kubelet** — агент K8s на каждой ноде. Слушает apiserver: "какие поды должны быть на этой ноде?". Через container runtime запускает и останавливает контейнеры. Отчитывается о статусе. Запускает **probes** (liveness/readiness). Именно kubelet реально делает эквивалент `docker run` под капотом.

**kube-proxy** — сетевой прокси на каждой ноде. Реализует **Service** — раскидывает трафик по подам через iptables (по умолчанию) или IPVS. Настраивает сетевые правила для service discovery внутри кластера.

**Container Runtime** — то что реально исполняет контейнеры. Раньше был Docker, сейчас **containerd** (стандарт) или CRI-O. Kubelet общается с runtime через CRI (Container Runtime Interface).

## Pod: минимальная единица

**Pod** — минимальная единица деплоя в K8s. Обычно один pod = один контейнер (одно приложение), но может быть несколько связанных контейнеров в одном pod (**sidecar** паттерн: основной + envoy proxy + логгер).

Контейнеры внутри pod'а делят:
- **Сеть** — один IP адрес, доступ друг к другу через localhost.
- **Volumes** — общий scratch storage.

Не делят файловую систему — у каждого своя.

**Pod эфемерен**. Умер pod → появился новый **с другим IP и другим именем**. Никогда не обращайся к pod напрямую по IP. Для стабильности используется Service (см. ниже).

Простой Pod манифест (для примера — обычно Pod создаётся через Deployment):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: isnaknpintegration-abc123
  namespace: knp
spec:
  containers:
  - name: app
    image: nexus.isna/isnaknpintegration:1.0.42
    ports:
    - containerPort: 8080
    env:
    - name: SPRING_PROFILES_ACTIVE
      value: prod
    resources:
      requests:
        memory: 512Mi
        cpu: 200m
      limits:
        memory: 1Gi
        cpu: 1000m
```

**Resource requests и limits** — критичная тема.

**Requests** — гарантированные ресурсы. Scheduler использует для планирования: pod ставится на ноду где есть свободные ресурсы >= requests. Если requests не указаны — scheduler ставит куда угодно, ресурсы могут не хватить.

**Limits** — максимально разрешённые. При превышении CPU limit контейнер throttled. При превышении memory limit — **OOMKilled** (exit code 137).

Для Java приложений критично соотношение JVM heap и container memory limit. Правильно: `-Xmx = 70-75% container memory`, чтобы оставить место для metaspace, thread stacks, direct memory, JVM internal.

Пример неправильной настройки:
- `resources.limits.memory: 1Gi`
- `-Xmx900m`

JVM heap 900 MB + metaspace (100-200 MB) + threads (100-200 MB) + direct memory + native = легко превышает 1 GB → OOMKilled. Знак что настроено неправильно.

## Deployment

Прямое создание Pod через yaml — редко. Обычно через **Deployment** — объект, декларирующий желаемое состояние replicated pod'ов.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: isnaknpintegration
  namespace: knp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: isnaknpintegration
  template:
    metadata:
      labels:
        app: isnaknpintegration
    spec:
      containers:
      - name: app
        image: nexus.isna/isnaknpintegration:1.0.42
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: prod
        resources:
          requests: {memory: 512Mi, cpu: 200m}
          limits: {memory: 1Gi, cpu: 1Gi}
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 5
```

Deployment создаёт **ReplicaSet** — объект следящий за количеством pods. Тот создаёт указанное количество Pod'ов из template. Если pod умирает — ReplicaSet создаёт новый. Если ты меняешь replicas: 5 — ReplicaSet создаёт ещё 2. Если 1 — убивает 2.

Deployment добавляет **rolling update** поверх ReplicaSet. Когда обновляешь image в deployment, создаётся новый ReplicaSet с новыми pods, старый ReplicaSet постепенно scale down. По одному pod'у за раз: поднимается новый, ждёт readiness, убивается один старый. Zero-downtime deployment.

Настройки rolling update:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1              # сколько сверх желаемого может быть во время update
      maxUnavailable: 0        # сколько может быть недоступно
```

`maxUnavailable: 0` — никогда не убиваем pod пока новый не готов. Строгий zero-downtime. Rolling update медленнее, зато без деградации.

## Service: стабильный endpoint

Pod'ы эфемерны — умирают, пересоздаются с новыми IP. Клиенты не могут обращаться напрямую. Нужна абстракция стабильного endpoint'а.

**Service** — виртуальный IP + балансировка трафика по группе pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: isnaknpintegration
  namespace: knp
spec:
  selector:
    app: isnaknpintegration
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

Service находит все Pod'ы с matching labels (`app: isnaknpintegration`), собирает их в **Endpoints**. Kube-proxy на каждой ноде настраивает iptables правила: трафик на ClusterIP:8080 распределяется между endpoints round-robin.

Клиент внутри кластера обращается по DNS имени: `isnaknpintegration.knp.svc.cluster.local:8080`. DNS резолвится в ClusterIP. Кube-proxy направляет на pod.

**Типы Service**:

**ClusterIP** (default) — доступен только внутри кластера. Для internal сервисов, к которым обращаются другие сервисы.

**NodePort** — открывает порт на всех нодах (диапазон 30000-32767). Внешний клиент обращается на любую ноду:порт. Используется редко, для тестов.

**LoadBalancer** — в облачном K8s создаёт cloud load balancer с public IP. Для external endpoints. On-premise — обычно не работает, нужны специальные решения (MetalLB).

**Headless** (`clusterIP: None`) — без ClusterIP. DNS возвращает список IP всех pods. Используется для StatefulSet или когда клиент сам делает discovery.

## Namespace

**Namespace** — логическое разделение кластера. Ресурсы разных namespaces изолированы. В КНП стандартные namespaces: `knp`, `fno`, `fo`, `tax-report`, `arm`.

Namespace изолирует ресурсы (объекты одинаковых имён могут быть в разных namespaces без конфликта). Не изолирует сеть по умолчанию (для этого NetworkPolicy).

`kubectl` работает с одним namespace через флаг `-n`:

```bash
kubectl get pods -n knp
kubectl logs isnaknpintegration-xyz -n knp
```

Или установить default через context:

```bash
kubectl config set-context --current --namespace=knp
```

## ConfigMap и Secret

Конфигурация приложения не должна быть в image (для immutability). Прокидывается через специальные объекты.

**ConfigMap** — открытая конфигурация:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: isnaknpintegration-config
  namespace: knp
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://db-knp:5432/knp
    logging:
      level:
        kz.gov.kgd.isna: INFO
  MAX_BATCH_SIZE: "100"
```

**Secret** — чувствительные данные (пароли, ключи, токены). Хранится в etcd в base64. Не шифрование само по себе (etcd можно зашифровать отдельно), но защищает от случайного просмотра в yaml.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: isnaknpintegration-secret
  namespace: knp
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0  # base64
```

Способы использования в pod'ах.

**Как env variables**:

```yaml
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: isnaknpintegration-config
    - secretRef:
        name: isnaknpintegration-secret
```

Все ключи из ConfigMap и Secret становятся env variables в контейнере. Spring Boot автоматически подхватит `MAX_BATCH_SIZE` → `max-batch-size` через relaxed binding.

**Как файлы** (volume):

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    configMap:
      name: isnaknpintegration-config
```

application.yml из ConfigMap появится как `/config/application.yml`, Spring Boot может подхватить через `spring.config.additional-location`.

## Ingress: внешняя точка входа

Service типа ClusterIP не доступен снаружи кластера. Для внешнего HTTP трафика используется **Ingress** — L7 балансировщик.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: knp-ingress
  namespace: knp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: knp.kgd.gov.kz
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: isnaknpintegration
            port:
              number: 8080
```

Ingress — только правила. Реально работает **Ingress Controller** — специальный pod с nginx (или Traefik, HAProxy) внутри. Он читает Ingress ресурсы, конфигурирует свой nginx, обслуживает external HTTP трафик.

В КНП стандартно nginx-ingress. Перед ним обычно ещё cloud load balancer или отдельный DNS/CDN уровень.

## Probes: проверки здоровья

Kubernetes мониторит здоровье pod'ов через три типа probes.

**Startup probe** — для медленных стартов. Проверяется первой. Пока не пройдёт, liveness и readiness не проверяются. Защищает медленные приложения от преждевременного kill.

**Liveness probe** — "жив ли контейнер". Fail несколько раз подряд → kubelet убивает контейнер, поднимает новый. Не должна зависеть от внешних систем — если DB упала, liveness всё равно UP.

**Readiness probe** — "готов ли принимать трафик". Fail → pod остаётся, но исключается из Endpoints (Service не направляет трафик на него). Может зависеть от внешних систем.

Типы probes:

**httpGet** — HTTP запрос на endpoint. Ответ 2xx или 3xx = success. Стандарт для Spring Boot приложений через Actuator.

**tcpSocket** — простое TCP connection на порт. Success если может подключиться. Легковесный, но не показывает готовность приложения.

**exec** — команда внутри контейнера. Exit code 0 = success.

Пример полной конфигурации:

```yaml
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
  failureThreshold: 3
```

Реальный кейс из КНП: `knp-form-arm-hazelcast3-not-migrated-flap` — Hazelcast 3 с несовместимостью → readiness периодически fail → K8s исключает pod из Service. Rolling restart пересоздал под уже с патчем.

## Основные команды kubectl

kubectl — CLI для управления K8s кластером. Основные команды.

**Просмотр объектов**:

```bash
kubectl get pods -n knp                    # список pods в namespace
kubectl get pods -n knp -o wide            # с IP и нодой
kubectl get deployments -n knp
kubectl get services -n knp
kubectl get all -n knp                     # всё сразу
```

**Детали**:

```bash
kubectl describe pod isnaknpintegration-xyz -n knp
kubectl describe deployment isnaknpintegration -n knp
```

`describe` показывает events, статус, ошибки. Первое что смотрят при проблемах.

**Логи**:

```bash
kubectl logs isnaknpintegration-xyz -n knp
kubectl logs -f isnaknpintegration-xyz -n knp        # follow
kubectl logs --tail 100 isnaknpintegration-xyz -n knp
kubectl logs -p isnaknpintegration-xyz -n knp        # предыдущий инстанс (после рестарта)
```

Реальный кавет из КНП (`knp-prod-historical-logs-elk`): `kubectl logs` показывает только текущий инстанс с последней ротации. Логи старше суток — только в ELK.

**Exec в контейнер**:

```bash
kubectl exec -it isnaknpintegration-xyz -n knp -- sh
kubectl exec isnaknpintegration-xyz -n knp -- ls /app
```

Полезно для отладки: посмотреть файлы, запустить jstack, диагностировать сеть.

**Rolling restart** deployment без изменения конфигурации:

```bash
kubectl rollout restart deployment/isnaknpintegration -n knp
```

Стандартный fix для многих проблем (Consul deregister, stale connection pool, memory leak).

**Rollout status**:

```bash
kubectl rollout status deployment/isnaknpintegration -n knp
```

Показывает progress rolling update.

**Rollback**:

```bash
kubectl rollout undo deployment/isnaknpintegration -n knp
kubectl rollout history deployment/isnaknpintegration -n knp
```

**Scale**:

```bash
kubectl scale deployment/isnaknpintegration --replicas=5 -n knp
```

**Apply YAML**:

```bash
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
```

## Helm

Управлять десятками YAML-манифестов для многих сервисов вручную неудобно. **Helm** — package manager для K8s.

Chart — набор шаблонов манифестов + values.yaml с настройками. Templates используют Go templating для параметризации.

Пример deployment template:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.name }}
spec:
  replicas: {{ .Values.replicas }}
  template:
    spec:
      containers:
      - name: app
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

values.yaml:

```yaml
name: isnaknpintegration
replicas: 3
image:
  repository: nexus.isna/isnaknpintegration
  tag: 1.0.42
resources:
  requests: {memory: 512Mi, cpu: 200m}
  limits: {memory: 1Gi, cpu: 1Gi}
```

Установка/обновление:

```bash
helm install knp-integration ./chart --values values-prod.yaml
helm upgrade knp-integration ./chart --values values-prod.yaml
helm rollback knp-integration 1
```

В КНП используется **helmsman** — надстройка над Helm для декларативного управления множеством releases в кластере.

## Namespace структура в КНП

Стандартные namespaces:
- **knp** — Kabinet Nalogoplatelshika, кабинет налогоплательщика для граждан.
- **fno** — сервисы Formy Nalogovoy Otchetnosti.
- **fno21** — то же на Java 21.
- **fo** — Formy Zayavleniy Otchetnosti.
- **fo21** — то же на Java 21.
- **tax-report** — сервисы налоговой отчётности.
- **tax-report21** — на Java 21.
- **arm** — АРМ налогового органа.

Namespace изолирует ресурсы, но не сеть — для network isolation используется NetworkPolicy.

## Диагностика проблем

Типичные ситуации и подходы.

**Pod не стартует, CrashLoopBackOff**:

```bash
kubectl describe pod <name> -n <ns>       # events, exit code
kubectl logs <name> -n <ns>               # свежие логи
kubectl logs -p <name> -n <ns>            # логи предыдущего запуска (уже упавшего)
```

Exit code 137 = OOMKilled. Проверить memory limits и Xmx настройку.

Exit code 143 = SIGTERM. Обычно graceful shutdown, не ошибка.

**Pod Running, но не отвечает**:

```bash
kubectl exec -it <pod> -n <ns> -- sh
# внутри проверить:
netstat -tlnp                             # что слушает
curl http://localhost:8080/actuator/health
```

Проверить readiness probe:

```bash
kubectl describe pod <name> -n <ns>
# смотреть Events, статус probes
```

**Изменение не применилось**:

```bash
kubectl rollout status deployment/<name> -n <ns>
kubectl get pods -n <ns> -w                # watch
```

Проверить, что pod действительно пересоздался с новым image.

**Проблемы Service networking**:

```bash
kubectl get endpoints <service> -n <ns>   # видит ли Service pods
kubectl get svc <service> -n <ns>
kubectl exec -it <other-pod> -n <ns> -- curl http://<service>.<ns>.svc.cluster.local:8080
```

Пустые endpoints — Service не нашёл pods (проблема с labels селектора).

**Проблемы с ресурсами**:

```bash
kubectl top pods -n <ns>                  # CPU/memory usage
kubectl top nodes                         # node utilization
```

Если top не работает — не установлен metrics-server.

## Заключение

Kubernetes — стандарт индустрии для оркестрации контейнеров. Знание для senior разработчика в микросервисной архитектуре обязательно.

Архитектура: control plane (apiserver, etcd, scheduler, controller-manager) плюс worker nodes (kubelet, kube-proxy, container runtime). Reconciliation loop — постоянное приведение реального состояния к желаемому.

Основные ресурсы: Pod (минимальная единица), Deployment (декларация replicated pods с rolling update), Service (стабильный endpoint + балансировка), Namespace (логическая изоляция), ConfigMap/Secret (конфигурация), Ingress (внешний HTTP entry point).

Probes для health checks — startup для медленных стартов, liveness для "жив", readiness для "готов принимать трафик". Правильная настройка probes на Actuator endpoints Spring Boot приложения.

Resource requests/limits критично. Для Java: `-Xmx = 70-75% container memory limit`, чтобы оставить место для metaspace, threads, direct memory. OOMKilled (exit 137) — типичный сигнал неправильной конфигурации.

kubectl — основной инструмент для работы. get, describe, logs, exec, rollout — базовый набор для повседневной работы. Rolling restart как fix для многих проблем.

Helm для управления сложными deployments через templates и values. В КНП helmsman как надстройка для декларативного управления множеством releases.

Для КНП контекста: понимание namespaces (knp, fno, fo, tax-report), стандартная конфигурация deployments с Actuator probes, использование helmsman для деплоя. Diagnostic workflow через kubectl describe и logs. Знание типичных проблем (CrashLoopBackOff, OOMKilled, Consul deregistration, image pull) и способов их решения.

Знание Kubernetes открывает возможность работать с production системой самостоятельно: диагностировать проблемы, применять fixes, понимать как деплоится код. Это ключевой навык для senior разработчика.

Дальше — Consul для service discovery. Хотя K8s имеет свой Service mechanism, в КНП дополнительно используется Consul для client-side load balancing и специфичных health checks. Понимание обоих механизмов необходимо.
