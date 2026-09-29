# 125. Kubernetes визуально: всё в одном месте на примере ISNA tax-rep

## Полная схема — весь кластер K8s одной картинкой

```
                    ┌────────────────────────┐
                    │      Пользователь      │
                    │  (kubectl / браузер)   │
                    └───────────┬────────────┘
                                │
                                │  HTTPS
                                ▼
    ╔══════════════════════════════════════════════════════════════════════╗
    ║                                                                      ║
    ║                    KUBERNETES CLUSTER (preprod)                      ║
    ║                                                                      ║
    ║   ┌──────────────────────────────────────────────────────────────┐   ║
    ║   │           CONTROL PLANE (мозг кластера)                      │   ║
    ║   │                                                              │   ║
    ║   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │   ║
    ║   │   │   API    │  │Scheduler │  │Controller│  │  etcd    │     │   ║
    ║   │   │  Server  │  │          │  │ Manager  │  │  (БД)    │     │   ║
    ║   │   │          │  │          │  │          │  │          │     │   ║
    ║   │   │ принимает│  │  куда    │  │  следит  │  │ хранит   │     │   ║
    ║   │   │ kubectl  │  │ запустить│  │ что всё  │  │  всё:    │     │   ║
    ║   │   │          │  │ pod?     │  │ живо     │  │ pods,    │     │   ║
    ║   │   │          │  │          │  │          │  │ deploys, │     │   ║
    ║   │   │          │  │          │  │          │  │ secrets  │     │   ║
    ║   │   └──────────┘  └──────────┘  └──────────┘  └──────────┘     │   ║
    ║   └──────────────────────────────────────────────────────────────┘   ║
    ║                                                                      ║
    ║   ═══════════════════════════════════════════════════════════════    ║
    ║                                                                      ║
    ║   Namespace: tax-report  ← наши 54 сервиса ISNA                      ║
    ║   ┌──────────────────────────────────────────────────────────────┐   ║
    ║   │                                                              │   ║
    ║   │  ┌───────────────────────────────────────────────────────┐   │   ║
    ║   │  │            Ingress (nginx)                            │   │   ║
    ║   │  │  → https://test-arm.kgd.gov.kz                        │   │   ║
    ║   │  │  роутит запросы на нужные Services                    │   │   ║
    ║   │  └────────────────────────┬──────────────────────────────┘   │   ║
    ║   │                           │                                  │   ║
    ║   │            ┌──────────────┼──────────────┐                   │   ║
    ║   │            ▼              ▼              ▼                   │   ║
    ║   │                                                              │   ║
    ║   │   ┌───────────┐   ┌───────────┐   ┌───────────┐              │   ║
    ║   │   │  Service  │   │  Service  │   │  Service  │              │   ║
    ║   │   │isna-tax-  │   │isna-tax-  │   │isna-tax-  │              │   ║
    ║   │   │  rep      │   │  rep-328  │   │  rep-     │  ...         │   ║
    ║   │   │  service  │   │  service  │   │  report   │              │   ║
    ║   │   │           │   │           │   │  service  │              │   ║
    ║   │   │ ClusterIP │   │ ClusterIP │   │ ClusterIP │              │   ║
    ║   │   │10.100.1.10│   │10.100.1.11│   │10.100.1.12│              │   ║
    ║   │   └─────┬─────┘   └─────┬─────┘   └─────┬─────┘              │   ║
    ║   │         │               │               │                    │   ║
    ║   │         │ балансировка  │               │                    │   ║
    ║   │         │ на pods       │               │                    │   ║
    ║   │         ▼               ▼               ▼                    │   ║
    ║   │                                                              │   ║
    ║   │   ═══════════════ WORKER NODES ═══════════════════════       │   ║
    ║   │                                                              │   ║
    ║   │   ┌─────────────────┐  ┌─────────────────┐                   │   ║
    ║   │   │    Node 1       │  │    Node 2       │                   │   ║
    ║   │   │  (сервер)       │  │  (сервер)       │                   │   ║
    ║   │   │                 │  │                 │                   │   ║
    ║   │   │  ┌───────────┐  │  │  ┌───────────┐  │                   │   ║
    ║   │   │  │ kubelet   │  │  │  │ kubelet   │  │                   │   ║
    ║   │   │  └───────────┘  │  │  └───────────┘  │                   │   ║
    ║   │   │  ┌───────────┐  │  │  ┌───────────┐  │                   │   ║
    ║   │   │  │containerd │  │  │  │containerd │  │                   │   ║
    ║   │   │  └───────────┘  │  │  └───────────┘  │                   │   ║
    ║   │   │                 │  │                 │                   │   ║
    ║   │   │  Pods:          │  │  Pods:          │                   │   ║
    ║   │   │                 │  │                 │                   │   ║
    ║   │   │ ┌─────────────┐ │  │ ┌─────────────┐ │                   │   ║
    ║   │   │ │isna-tax-rep │ │  │ │isna-tax-rep │ │                   │   ║
    ║   │   │ │pod-abc      │ │  │ │pod-def      │ │                   │   ║
    ║   │   │ │             │ │  │ │             │ │                   │   ║
    ║   │   │ │Docker       │ │  │ │Docker       │ │                   │   ║
    ║   │   │ │container:   │ │  │ │container:   │ │                   │   ║
    ║   │   │ │java -jar    │ │  │ │java -jar    │ │                   │   ║
    ║   │   │ │isna-tax-    │ │  │ │isna-tax-    │ │                   │   ║
    ║   │   │ │rep.jar      │ │  │ │rep.jar      │ │                   │   ║
    ║   │   │ └─────────────┘ │  │ └─────────────┘ │                   │   ║
    ║   │   │                 │  │                 │                   │   ║
    ║   │   │ ┌─────────────┐ │  │ ┌─────────────┐ │                   │   ║
    ║   │   │ │isna-tax-rep │ │  │ │isna-tax-rep │ │                   │   ║
    ║   │   │ │-328-pod-xyz │ │  │ │-report-abc  │ │                   │   ║
    ║   │   │ │             │ │  │ │             │ │                   │   ║
    ║   │   │ │Docker       │ │  │ │Docker       │ │                   │   ║
    ║   │   │ │container:   │ │  │ │container:   │ │                   │   ║
    ║   │   │ │java -jar    │ │  │ │java 21      │ │                   │   ║
    ║   │   │ │328.jar      │ │  │ │-jar         │ │                   │   ║
    ║   │   │ │             │ │  │ │report.jar   │ │                   │   ║
    ║   │   │ └─────────────┘ │  │ └─────────────┘ │                   │   ║
    ║   │   │                 │  │                 │                   │   ║
    ║   │   │ ┌─────────────┐ │  │ ┌─────────────┐ │                   │   ║
    ║   │   │ │isna-tax-rep │ │  │ │isna-tax-rep │ │                   │   ║
    ║   │   │ │-sync-pod    │ │  │ │-nz-007-pod  │ │                   │   ║
    ║   │   │ └─────────────┘ │  │ └─────────────┘ │                   │   ║
    ║   │   │                 │  │                 │                   │   ║
    ║   │   │  ...            │  │  ...            │                   │   ║
    ║   │   │                 │  │                 │                   │   ║
    ║   │   │ Диск node:      │  │ Диск node:      │                   │   ║
    ║   │   │ /var/lib/       │  │ /var/lib/       │                   │   ║
    ║   │   │  containerd/    │  │  containerd/    │                   │   ║
    ║   │   │  ← кэш images   │  │  ← кэш images   │                   │   ║
    ║   │   │                 │  │                 │                   │   ║
    ║   │   │ /var/log/pods/  │  │ /var/log/pods/  │                   │   ║
    ║   │   │  ← логи         │  │  ← логи         │                   │   ║
    ║   │   └─────────────────┘  └─────────────────┘                   │   ║
    ║   │                                                              │   ║
    ║   │  Плюс в этом namespace:                                      │   ║
    ║   │                                                              │   ║
    ║   │  ┌────────────────┐  ┌────────────────┐                      │   ║
    ║   │  │  ConfigMap     │  │    Secrets     │                      │   ║
    ║   │  │                │  │                │                      │   ║
    ║   │  │ application.yml│  │ db-password    │                      │   ║
    ║   │  │ конфиги        │  │ nexus-creds    │                      │   ║
    ║   │  │                │  │ ЭЦП keys       │                      │   ║
    ║   │  └────────────────┘  └────────────────┘                      │   ║
    ║   │                                                              │   ║
    ║   │  ┌────────────────────────────────────────────────────────┐  │   ║
    ║   │  │  PersistentVolume (внешний диск / NFS)                 │  │   ║
    ║   │  │                                                        │  │   ║
    ║   │  │  Данные которые НЕ должны потеряться:                  │  │   ║
    ║   │  │  - файлы БД PostgreSQL (если запущена в K8s)           │  │   ║
    ║   │  │  - загруженные ФНО файлы                               │  │   ║
    ║   │  │  - логи архивные                                       │  │   ║
    ║   │  └────────────────────────────────────────────────────────┘  │   ║
    ║   │                                                              │   ║
    ║   └──────────────────────────────────────────────────────────────┘   ║
    ║                                                                      ║
    ║   ═══════════════════════════════════════════════════════════════    ║
    ║                                                                      ║
    ║   Другие namespaces (в том же кластере):                             ║
    ║                                                                      ║
    ║   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   ║
    ║   │ kube-system      │  │  monitoring      │  │ ingress-nginx    │   ║
    ║   │                  │  │                  │  │                  │   ║
    ║   │  CoreDNS         │  │  Prometheus      │  │  nginx-ingress   │   ║
    ║   │  kube-proxy      │  │  Grafana         │  │  controller      │   ║
    ║   │  metrics-server  │  │                  │  │                  │   ║
    ║   │                  │  │                  │  │                  │   ║
    ║   │  системное       │  │  метрики +       │  │  входные HTTP    │   ║
    ║   │  для работы K8s  │  │  дашборды        │  │  запросы         │   ║
    ║   └──────────────────┘  └──────────────────┘  └──────────────────┘   ║
    ║                                                                      ║
    ╚══════════════════════════════════════════════════════════════════════╝
                                    │
                                    │ K8s качает images отсюда
                                    ▼
                    ┌──────────────────────────┐
                    │        NEXUS             │
                    │  test-nexus.kgd.gov.kz   │
                    │                          │
                    │  Docker images:          │
                    │  isna-tax-rep:abc123     │
                    │  isna-tax-rep-328:abc123 │
                    │  isna-tax-rep-report:xyz │
                    │  ... (54 images)         │
                    └──────────────────────────┘
```

## Разбор — что где на схеме

**Control Plane** (сверху, "мозг"):
- **API Server** — принимает `kubectl` команды.
- **Scheduler** — решает на какой Node запустить новый Pod.
- **Controller Manager** — следит что всё живо (упал pod → создать новый).
- **etcd** — БД где хранится ВСЁ состояние (все pods, deployments, secrets).

**Namespace `tax-report`** (посередине, где наши сервисы):
- **Ingress (nginx)** — точка входа снаружи (URL → внутренние Services).
- **Services** — стабильные адреса для групп pods.
- **Pods** — реально запущенные Docker containers с нашими Java-приложениями.

**Worker Nodes** (внутри namespace, физические серверы):
- Каждая имеет **kubelet** (агент K8s) и **containerd** (запускает containers).
- На каждой node — по нескольку Pod'ов.
- На диске: `/var/lib/containerd/` (кэш images) и `/var/log/pods/` (логи).

**Дополнительно в namespace tax-report**:
- **ConfigMap** — конфиги (`application.yml`).
- **Secrets** — пароли (БД, Nexus, ЭЦП).
- **PersistentVolume** — внешний диск для важных данных.

**Другие namespaces**:
- `kube-system` — системные компоненты K8s (DNS, proxy).
- `monitoring` — Prometheus + Grafana.
- `ingress-nginx` — nginx контроллер.

**Nexus** (снаружи K8s):
- K8s качает Docker images отсюда когда создаёт pod'ы.
- Три Nexus для трёх окружений (dev/preprod/prod).


## Как это выглядит в реальности — команды kubectl

```
    # Посмотреть все namespace:
    $ kubectl get namespaces
    NAME              STATUS
    kube-system       Active
    tax-report        Active   ← наш
    monitoring        Active
    ingress-nginx     Active
    
    # Посмотреть все pods в tax-report:
    $ kubectl get pods -n tax-report
    NAME                                READY  STATUS   NODE
    isna-tax-rep-abc123-xyz             1/1    Running  node-1
    isna-tax-rep-abc123-def             1/1    Running  node-2
    isna-tax-rep-328-abc789-qwe         1/1    Running  node-1
    isna-tax-rep-report-def456-poi      1/1    Running  node-2
    isna-tax-rep-sync-abc-yui           1/1    Running  node-1
    ... (много pods)
    
    # Обновить image:
    $ kubectl set image \
        deployment/isna-tax-rep-328 \
        isna-tax-rep-328=test-nexus/isna-tax-rep-328:v2 \
        -n tax-report
    
    # Посмотреть логи pod'а:
    $ kubectl logs -n tax-report isna-tax-rep-328-abc-xyz
    2026-09-28 10:15:00 INFO ...
    2026-09-28 10:15:01 INFO Started at ...
    
    # Посмотреть детали pod:
    $ kubectl describe pod -n tax-report isna-tax-rep-328-abc-xyz
    Node: node-1/10.0.0.5
    IP: 10.100.5.34
    Image: test-nexus/isna-tax-rep-328:abc123
    Status: Running
```

## Что запомнить

**Один K8s кластер** содержит:
- 1 Control Plane (обычно 3 сервера для HA).
- N Worker Nodes (сколько нужно).

**Всё разбито на namespaces**:
- `tax-report` — все 54 сервиса ISNA.
- Отдельные namespaces для мониторинга, ingress, kube-system.

**Каждый сервис ISNA** = 1 Deployment + 1 Service + N Pods (обычно 2-3 pod'а).

**Pods запускаются на разных Worker Nodes** — если одна упала, работает на других.

**Ingress (nginx)** — единая точка входа снаружи. Роутит по URL на нужный Service.

**Service** — стабильный внутренний адрес, балансирует между pods.

**Nexus снаружи K8s** — оттуда качаются Docker images для pods.

**Данные**:
- Runtime (кэш images, логи) — на диске каждой Node.
- Постоянные (файлы БД, uploads) — в PersistentVolume (внешний диск).
- Состояние кластера (какие есть pods/services/etc) — в etcd на Control Plane.

---

Дальше:
- **10** — K8s детально.
- **77** — K8s deep для микросервисов.
- **80** — K8s internals.
- **124** — Build и deploy pipeline ISNA.
