# 09. Docker: контейнеризация Java-приложений

## Проблема, которую решает Docker

Классическая история "у меня работает, у тебя нет" — фундаментальная проблема software development. У тебя JDK 21, у CI — JDK 17. У тебя PostgreSQL 15 локально, на проде — PostgreSQL 12. Timezone на dev машине — Asia/Almaty, на серверах — UTC. Файловые системы разные, версии OS библиотек разные, глобальные env variables отличаются. Код одинаковый, работает по-разному.

Раньше проблему пытались решать через "золотые образы серверов" (Chef, Puppet, Ansible), стандартизацию OS у разработчиков, документирование "правильной" конфигурации. Работало плохо. Разработчик всё равно ставил на локальной машине свою версию Java, забывал обновлять зависимости, использовал другой compiler. Prod разрушение возможностей — DevOps kill'ер.

**Docker** предложил радикальное решение: упаковать приложение вместе со всем окружением в неизменяемый **container**. JDK, приложение, config, зависимости — всё в одном immutable image. Один и тот же image запускается идентично на dev машине, CI runner, production сервере. Проблема окружения сведена к нулю.

Docker появился в 2013 году, и за пять лет стал стандартом индустрии. Микросервисная архитектура и Kubernetes построены на предположении что приложения контейнеризированы. Знание Docker для senior Java-разработчика — обязательная компетенция.

В этом файле разберём Docker глубоко. Что такое container технически (namespaces, cgroups). Разница с VM. Image, layer, registry — концепции. Dockerfile синтаксис и best practices. Multi-stage build и layered JAR. Volume и network. Практические аспекты для Spring Boot: JVM в контейнере, signal handling, graceful shutdown, non-root user. Диагностика проблем.

## Container vs VM: техническая разница

Часто путают container с виртуальной машиной. Отличия принципиальны.

**Virtual Machine** запускается через **hypervisor** (VMware, KVM, Hyper-V), эмулирующий физическое железо. Каждая VM имеет полностью свою операционную систему — свой kernel, свои процессы, свои драйверы. Guest OS работает как будто на реальном оборудовании. Стартует минуты — нужно загрузить kernel, инициализировать драйверы, запустить init процессы. Тяжёлая: гигабайты только на копию OS.

**Container** — это обычный процесс операционной системы, работающий в изолированной среде. **Kernel общий с хостом.** Внутри контейнера — только приложение и его user-space библиотеки (glibc или musl, JDK, ваши JAR). Никакого дублирования OS. Стартует за секунды — просто запускается процесс. Лёгкий: десятки-сотни MB.

Изоляция контейнера построена на **двух Linux механизмах ядра**.

**Namespaces** — изолируют "видимость" процесса. Процесс в namespace видит своё пространство, не общее с системой. Разные типы namespaces:
- **PID namespace** — свой набор процессов. `ps aux` внутри контейнера показывает только процессы этого контейнера. Другие процессы хоста не видны.
- **Network namespace** — свой сетевой стек. Свои интерфейсы (обычно `eth0` — виртуальный), свои маршруты, свои iptables правила. Изоляция от сети хоста.
- **Mount namespace** — своя файловая система. Root в контейнере — не root хоста, а свой image.
- **UTS namespace** — своё hostname. `hostname` внутри контейнера — не хостовое.
- **IPC namespace** — свои очереди сообщений.
- **User namespace** — своё UID/GID mapping. Root в контейнере может быть не root на хосте.

**cgroups (control groups)** — ограничивают ресурсы. Каждый контейнер помещается в свою cgroup, для которой определены лимиты:
- CPU (сколько ядер или процентов).
- Memory (максимум RAM).
- I/O (пропускная способность диска).
- Network bandwidth.

Docker (и containerd, CRI-O) — это удобные обёртки над этими двумя механизмами Linux. Технически container — это `clone()` syscall с флагами создать новые namespaces плюс `setrlimit()` для помещения в cgroup. Никакой magic — просто process в изолированной среде.

**Отсюда**: container — процесс, не мини-VM. Быстрый, лёгкий, но kernel общий с хостом. Изоляция меньше чем у VM — уязвимость в kernel может позволить escape из контейнера. Для production это учитывается через дополнительные механизмы (seccomp, AppArmor, SELinux, gVisor как runtime).

**На Windows и macOS** Docker Desktop под капотом запускает Linux VM (WSL2 на Windows, LinuxKit/HyperKit на macOS). Namespaces и cgroups — это Linux фичи, их нет в Windows kernel. Внутри Linux VM уже нативные Linux контейнеры. Отсюда всякие странности с performance и networking на Windows/macOS — это через VM, не native.

## Ключевые концепции

Небольшой словарь.

**Image** — шаблон контейнера. Immutable. Содержит файловую систему плюс метаданные (какая команда запускается, какие env variables установлены). Хранится в **registry**.

**Container** — запущенный экземпляр image. Изолированный процесс со своими namespaces и cgroups. От одного image можно запустить сколько угодно контейнеров.

**Dockerfile** — текстовый файл с инструкциями для сборки image.

**Layer** — часть image, соответствующая одной инструкции Dockerfile. Immutable, кэшируется.

**Registry** — хранилище images. Docker Hub — публичный. Nexus, Harbor, ECR, GCR — enterprise/cloud registries. В КНП — внутренний Nexus.

**Repository** — набор images с одним именем и разными tags. `isna-knp:1.0.42`, `isna-knp:1.0.43`, `isna-knp:latest` — все в одном repository.

**Volume** — постоянное хранилище для данных, живёт независимо от контейнеров. Умер контейнер — volume остался.

**Network** — сеть для коммуникации между контейнерами. Docker создаёт виртуальные сети.

## Dockerfile: рецепт сборки

Dockerfile — текстовый файл с последовательностью инструкций. Каждая инструкция создаёт слой в результирующем image.

Простейший Dockerfile для Spring Boot приложения:

```dockerfile
FROM amazoncorretto:21-alpine
WORKDIR /app
COPY build/libs/isna-knp-integration.jar app.jar
ENV JAVA_OPTS="-Xms512m -Xmx1024m -XX:+UseG1GC"
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

Разберём инструкции подробно.

**FROM** — базовый image. С чего начинается твой. `amazoncorretto:21-alpine` — Amazon Corretto 21 на базе Alpine Linux. Варианты:
- `amazoncorretto:21-alpine` — Alpine Linux base (5 MB), musl libc.
- `eclipse-temurin:21-jre-alpine` — альтернатива от Eclipse Foundation.
- `openjdk:21-slim` — Debian slim base (glibc).
- `gcr.io/distroless/java21` — Google distroless, минимум, без shell.
- `scratch` — вообще пустой.

Alpine — 5 MB, тонкий, использует musl. Иногда конфликты с native библиотеками, скомпилированными под glibc. Slim/Debian — больше (30-50 MB), но glibc стандартный, меньше проблем.

В КНП — Corretto/OpenJDK на Alpine для стандартных сервисов. Distroless — для критичных с точки зрения security.

**WORKDIR** — рабочая директория для последующих инструкций. Аналог `cd`. Если директории нет — создаётся.

**COPY vs ADD**. Обе копируют файлы с хоста в image. COPY — простое копирование. ADD дополнительно может распаковывать tar-архивы, скачивать по URL. Обычно **предпочитается COPY** — предсказуемое поведение, без магии. ADD только для распаковки локальных tar-архивов если нужно.

**RUN** — выполняет команду при сборке image. Результат фиксируется в слое.

```dockerfile
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

Правило — объединять связанные команды в один RUN. Иначе получаются лишние слои с промежуточным состоянием. Пример: если сделать три отдельных `RUN apt-get update`, `RUN apt-get install curl`, `RUN rm -rf /var/lib/apt/lists/*` — получишь три слоя, в первом обновлённый apt cache, во втором установленный curl со всё ещё большим cache, только в третьем cache очищен. Общий размер image больше.

**CMD vs ENTRYPOINT**. Оба определяют что выполнить при `docker run`. Тонкая разница.

**ENTRYPOINT** — фиксированная команда, которая запускается всегда.

**CMD** — аргументы по умолчанию, могут быть переопределены при `docker run`.

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=prod"]
```

`docker run isna-knp` → выполняется `java -jar app.jar --spring.profiles.active=prod`.
`docker run isna-knp --spring.profiles.active=dev` → `java -jar app.jar --spring.profiles.active=dev` (CMD переопределён).

Есть **две формы** записи ENTRYPOINT и CMD:

**Exec form** (предпочтительная): `["java", "-jar", "app.jar"]` — прямой exec, аргументы как массив.

**Shell form**: `java -jar app.jar` — оборачивается в `sh -c`.

Разница критична для сигналов. При shell form реально запускается `sh -c "java -jar app.jar"` — PID 1 в контейнере это shell, не Java. Когда Docker/K8s отправляет SIGTERM (для graceful shutdown), сигнал идёт в PID 1 = shell. Shell не forwardit сигнал в java подпроцесс. Java не получает SIGTERM, не делает graceful shutdown, падает по timeout когда прилетает SIGKILL.

При exec form Java запускается напрямую как PID 1. SIGTERM идёт в неё, Spring Boot делает graceful shutdown (закрывает Consul registration, дорабатывает in-flight запросы, закрывает connection pools). Правильно.

**Правило**: используй exec form. Если нужны env variables в команде — правильнее `ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]` (иногда нужно), но ещё лучше — использовать `ENV JAVA_TOOL_OPTIONS="..."`. JVM автоматически подхватывает JAVA_TOOL_OPTIONS без явной передачи в command line.

**EXPOSE** — декоративная инструкция. "Этот image слушает на порту 8080". Не открывает порт наружу! Просто документация плюс метаданные для инструментов. Реальное открытие порта — при `docker run -p 8080:8080`.

**ENV** — устанавливает переменную окружения. Доступна и в RUN на этапе сборки, и в контейнере в runtime.

```dockerfile
ENV JAVA_OPTS="-Xmx1g"
ENV SPRING_PROFILES_ACTIVE=prod
```

**ARG** — переменная только для этапа сборки. Не сохраняется в image.

```dockerfile
ARG APP_VERSION=1.0.0
LABEL version=$APP_VERSION
```

При сборке — `docker build --build-arg APP_VERSION=1.0.42 .`.

**USER** — меняет пользователя для последующих инструкций и для runtime. **Best practice**: не запускать root в контейнере. Создать non-root пользователя:

```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

Уменьшает blast radius при уязвимости в приложении.

**HEALTHCHECK** — команда проверки здоровья контейнера. Docker периодически вызывает, статус (healthy/unhealthy) виден в `docker ps`.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1
```

В Kubernetes редко используется — там свои probes более гибкие.

## Слои и кэш

Каждая инструкция Dockerfile создаёт **layer** — diff файловой системы после выполнения инструкции. Image — стопка слоёв.

```
Dockerfile:                    Image слои:
FROM alpine:3.19        →  Layer 1: alpine база (5 MB)
RUN apk add curl        →  Layer 2: +curl (2 MB)
COPY app.jar /app/      →  Layer 3: +app.jar (80 MB)
CMD [...]               →  metadata (не слой)
```

**Docker кэширует слои**. При пересборке — если инструкция не изменилась И входные данные не изменились, слой берётся из кэша, не пересобирается. Это огромный ускоритель build.

Правило порядка: **менее часто меняющиеся инструкции — выше**. Часто меняющиеся — вниз.

Плохо:

```dockerfile
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY . .              # весь проект — часто меняется
RUN ./gradlew build
CMD ["java", "-jar", "build/libs/app.jar"]
```

Любое изменение в src → COPY меняется → все последующие слои пересобираются, включая долгий `RUN ./gradlew build`.

Хорошо:

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY gradle gradle
COPY gradlew build.gradle settings.gradle ./
RUN ./gradlew --no-daemon dependencies       # cached если deps не менялись
COPY src src
RUN ./gradlew --no-daemon bootJar

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/build/libs/app.jar app.jar
USER app
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Разделены копирование конфигов сборки (редко меняется), dependencies (редко меняется), source code (часто), сборка. Кэш работает эффективно.

## Multi-stage build

`FROM ... AS build` создаёт промежуточную стадию сборки. Второй `FROM` начинает новую стадию — финальный image. `COPY --from=build` берёт файлы из первой стадии.

**Multi-stage** решает проблему размера. Ты используешь полноценный JDK image (сотни MB) для компиляции. Финальный image — тонкий JRE (~200 MB) с только скомпилированным JAR. JDK, gradle cache, исходники не попадают в финальный.

Разница ощутима. Без multi-stage — image 800 MB+ (JDK, source, gradle cache, deps, jar). С multi-stage — 250 MB (JRE + jar). Меньше в 3-4 раза. Быстрее скачивать в K8s, экономия storage в registry.

## Layered JAR (Spring Boot)

Ещё более продвинутая оптимизация. Spring Boot 2.3+ поддерживает разбиение fat JAR на слои. Изнутри JAR:

```
BOOT-INF/lib/                    — зависимости (редко меняются)
BOOT-INF/classes/                — твой код (часто меняется)
org/springframework/boot/loader/ — Spring Boot loader (стабильно)
```

Можно извлечь через `layertools`:

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS layers
WORKDIR /app
COPY app.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=layers /app/dependencies/ ./
COPY --from=layers /app/spring-boot-loader/ ./
COPY --from=layers /app/snapshot-dependencies/ ./
COPY --from=layers /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

Каждый `COPY` — свой Docker layer. dependencies — редко меняются, layer cached. Application — часто меняется, только он пересобирается. Registry pull быстрый, потому что скачивает только изменённый application layer (~1 MB) вместо всего JAR (~80 MB).

Активация в build.gradle:

```gradle
bootJar {
    layered {
        enabled = true
    }
}
```

## Работа с images

Стандартные команды.

Сборка:

```bash
docker build -t isna-knp-integration:1.0.42 .
docker build -t isna-knp-integration:1.0.42 -f Dockerfile.prod .
docker build --no-cache -t isna-knp .        # игнорировать кэш
docker build --build-arg APP_VERSION=1.0.42 -t isna-knp .
```

Просмотр:

```bash
docker images                        # список локальных images
docker history isna-knp:1.0.42       # слои image с размерами
docker inspect isna-knp:1.0.42       # полная информация (JSON)
```

Тэгирование и push в registry:

```bash
docker tag isna-knp:1.0.42 nexus.isna.internal/isna-knp:1.0.42
docker push nexus.isna.internal/isna-knp:1.0.42
```

Registry требует аутентификации:

```bash
docker login nexus.isna.internal
# логин/пароль
```

## Запуск контейнера

Простой запуск:

```bash
docker run isna-knp-integration:1.0.42
```

Контейнер стартует, привязывается к терминалу. Ctrl-C убивает.

Полный запуск с production параметрами:

```bash
docker run \
    -d \                                      # detached (фон)
    --name knp \                              # имя контейнера
    -p 8080:8080 \                            # host_port:container_port
    -e DB_PASSWORD=secret \                   # env variable
    -e SPRING_PROFILES_ACTIVE=prod \
    --env-file .env \                          # env из файла
    -v /host/logs:/app/logs \                  # volume
    --memory 1g \                              # cgroup memory limit
    --cpus 2 \                                 # cgroup cpu limit
    --restart unless-stopped \                 # policy рестарта
    --network isna-net \                       # сеть
    isna-knp-integration:1.0.42
```

Управление запущенными контейнерами:

```bash
docker ps                       # запущенные контейнеры
docker ps -a                    # + остановленные
docker logs knp                 # логи контейнера
docker logs -f knp              # follow (tail)
docker logs --tail 100 knp
docker exec -it knp sh          # войти внутрь для отладки
docker stop knp                 # graceful (SIGTERM, потом SIGKILL через 10 сек)
docker kill knp                 # SIGKILL сразу
docker rm knp                   # удалить остановленный
docker rm -f knp                # force
docker stats                    # использование ресурсов live
```

## Volumes: постоянные данные

Данные внутри контейнера эфемерны. Умер контейнер (удалён) — данные внутри его файловой системы теряются. Для персистентности используются **volumes**.

**Named volume** — Docker управляет хранилищем:

```bash
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres:15
```

Volume лежит в `/var/lib/docker/volumes/pgdata/_data`, живёт независимо от контейнеров.

**Bind mount** — папка хоста:

```bash
docker run -v /home/user/data:/app/data isna-knp
```

Полезно для development (код с хоста внутри контейнера).

## Networks

По умолчанию Docker создаёт мост `bridge`. Контейнеры на нём видят друг друга по IP.

**Named network** предпочтительнее для управления:

```bash
docker network create isna-net
docker run --network isna-net --name db postgres:15
docker run --network isna-net --name app isna-knp
```

`app` может достучаться до БД по имени `db` через Docker DNS (`jdbc:postgresql://db:5432/knp`).

Типы сетей:
- **bridge** (default) — виртуальный switch.
- **host** — использует сеть хоста напрямую (быстро, но нет изоляции).
- **none** — без сети.
- **overlay** — для Docker Swarm или multi-host сценариев.

## Docker Compose

Для локальной разработки — оркестрация нескольких контейнеров одним файлом. `docker-compose.yml`:

```yaml
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: knp
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
  
  app:
    build: .
    depends_on:
      - db
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/knp
      SPRING_DATASOURCE_PASSWORD: secret

volumes:
  pgdata:
```

Команды:

```bash
docker compose up -d       # запустить всё в фоне
docker compose logs -f
docker compose down
```

Незаменимо для локального dev. В production обычно Kubernetes.

## Практические трюки для Spring Boot

**JVM в контейнере**. Раньше JVM не видела cgroup limits, брала память как процент от host memory. В контейнере с ограничением 1 GB на хосте с 32 GB — JVM брала heap 24 GB → OOMKilled кернелом. Классическая проблема.

С JDK 10+ JVM видит cgroup memory limit. Всё равно принято фиксировать явно:

```dockerfile
ENV JAVA_OPTS="-Xms512m -Xmx1024m -XX:MaxRAMPercentage=75.0"
```

Или через `JAVA_TOOL_OPTIONS` — JVM подхватит автоматически без явной передачи:

```dockerfile
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
```

**Signal handling**. Spring Boot 2.3+ поддерживает graceful shutdown при SIGTERM. Условия: приложение должно быть PID 1 в контейнере (exec form ENTRYPOINT), плюс настройка:

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

При SIGTERM Spring останавливает приём новых запросов, дорабатывает in-flight, закрывает connection pools, deregister из Consul. Только потом exit.

Shell form ENTRYPOINT ломает это — сигнал идёт в sh, не в Java. Приложение убивается SIGKILL через grace period (обычно 30 сек), без graceful shutdown.

**`.dockerignore`** — аналог `.gitignore` для контекста сборки. Docker при `docker build .` копирует всё из директории в builder daemon. Без `.dockerignore` в контекст попадёт `.git/`, `.gradle/`, `build/`, `node_modules/` — гигабайты ненужного. Пример:

```
.git
.gradle
build
node_modules
*.log
Dockerfile
```

Ускоряет `docker build` в разы.

**Non-root user**. Требование безопасности во многих корпоративных стандартах:

```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

**Distroless / scratch base images**. Ещё более минимальные:

```dockerfile
FROM gcr.io/distroless/java21-debian12
COPY app.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

Плюсы: меньше (около 100-150 MB), безопаснее (нет shell → меньше атак).
Минусы: `docker exec` не даёт sh — сложнее отладка внутри контейнера.

## Docker в CI/CD

Типовой pipeline в КНП (GitLab CI):

```yaml
stages:
  - build
  - image
  - deploy

build:
  stage: build
  image: eclipse-temurin:21-jdk
  script:
    - ./gradlew bootJar
  artifacts:
    paths:
      - build/libs/*.jar

image:
  stage: image
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $NEXUS_USER -p $NEXUS_PASS $NEXUS_REGISTRY
    - docker build -t $NEXUS_REGISTRY/isna-knp:$CI_COMMIT_SHA .
    - docker push $NEXUS_REGISTRY/isna-knp:$CI_COMMIT_SHA
    - docker tag $NEXUS_REGISTRY/isna-knp:$CI_COMMIT_SHA $NEXUS_REGISTRY/isna-knp:latest
    - docker push $NEXUS_REGISTRY/isna-knp:latest

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/isnaknpintegration \
        app=$NEXUS_REGISTRY/isna-knp:$CI_COMMIT_SHA -n knp
```

Реальный процесс сложнее (Helm charts, helmsman), но идея та же: build JAR → build image → push to registry → K8s deploy.

## Диагностика проблем

**Контейнер сразу умирает**:

```bash
docker run -it isna-knp:1.0.42 sh    # войти вместо приложения
# посмотреть что там
```

Или проверить логи:

```bash
docker logs <container-id>
```

Exit code даёт подсказку:
- **137** — SIGKILL (обычно OOMKilled). Проверить memory limits.
- **143** — SIGTERM (graceful shutdown). Нормально при docker stop.
- **1** — приложение упало по своей причине. Смотри логи.

**Slow build**. Стратегии оптимизации:
- Multi-stage.
- Сортировать Dockerfile (редко меняющееся вверх).
- `.dockerignore`.
- Layered JAR.
- BuildKit: `DOCKER_BUILDKIT=1 docker build ...` — быстрее, параллельно кэширует.

**Image слишком большой**:

```bash
docker history isna-knp:1.0.42
# смотрим что раздувает
```

Обычные причины: JDK вместо JRE в финальном image, не удалённые apt cache, копирование всей папки с временными файлами.

**Приложение не отвечает**:

```bash
docker inspect <container>            # смотрим Ports, Networks
docker port <container>               # какие порты замаплены
docker exec <container> netstat -tlnp # что слушает внутри
```

## Заключение

Docker — фундамент современной микросервисной архитектуры. Container — это процесс в изолированной среде через namespaces и cgroups Linux, не мини-VM. Быстрый, лёгкий, portable.

Image — immutable шаблон со слоями. Dockerfile описывает сборку. Правильный Dockerfile: multi-stage build (JDK для сборки, JRE для runtime), правильный порядок инструкций для кэша, `.dockerignore`, non-root user, exec form ENTRYPOINT.

Для Spring Boot специфично: layered JAR для оптимального кэша, JVM heap через `MaxRAMPercentage` или явные Xmx, JAVA_TOOL_OPTIONS для передачи в JVM без правки ENTRYPOINT, graceful shutdown через `server.shutdown: graceful` и правильный signal handling.

Docker Compose для локальной разработки, для production — Kubernetes. Registry (в КНП — Nexus) как хранилище images. CI/CD автоматизирует build → image → push → deploy.

Диагностика: exit codes (137 OOMKilled, 143 SIGTERM), `docker logs`, `docker exec` для входа в running container. Понимание OOMKilled особенно важно — правильные Java memory settings в container критичны.

Для КНП контекста — все микросервисы Docker-контейнеризированы. Base image amazoncorretto:21-alpine (или jre-alpine для рантайма), multi-stage build с layered JAR, non-root user, graceful shutdown. Знание Docker — обязательный навык.

Дальше — Kubernetes, оркестратор контейнеров. Все Docker knowledge применяется в K8s deployment: контейнеры запускаются как поды, images pull из registry, resource limits через cgroups.
