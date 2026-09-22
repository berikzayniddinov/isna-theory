# 101. Точечная схема: сборка, запуск и деплой Spring Boot микросервиса КНП

Полная визуальная карта пути от исходников до работающего Pod в K8s кластере с трафиком. Каждая точка — отдельная фаза, отдельный процесс, отдельный класс failure. Все три сценария: локальный dev-run, CI/CD pipeline, GitOps deploy.

---

## 0. Общая карта — от Java-файла до трафика

```
┌───────────────────────────────────────────────────────────────────────┐
│                     ИСХОДНЫЙ КОД в git                                │
│           gitlab.1sc.kz/isna/isnaknpuser.git                          │
└───────────────────────────────┬───────────────────────────────────────┘
                                │ git push
                                ▼
        ┌───────────────────────────────────────────────┐
        │            GITLAB SERVER                      │
        │  post-receive hook → создание Pipeline        │
        └───────────────┬───────────────────────────────┘
                        │ long polling / webhook
                        ▼
        ┌───────────────────────────────────────────────┐
        │            GITLAB RUNNER                      │
        │  K8s executor: Pod per job                    │
        └───────────────┬───────────────────────────────┘
                        │
        ┌───────────────┴───────────────┐
        │                               │
        ▼                               ▼
┌───────────────┐              ┌───────────────┐
│   BUILD       │              │  TEST         │
│ Gradle:       │              │ - Unit        │
│  compile      │              │ - Integration │
│  bootJar      │              │   (Testcont.) │
└───────┬───────┘              └───────────────┘
        │
        ▼
┌───────────────┐
│  DOCKER BUILD │
│  multi-stage  │
│  layered JAR  │
└───────┬───────┘
        │
        ▼
┌───────────────────────────┐
│   NEXUS REGISTRY          │
│  registry.1sc.kz/         │
│  isnaknpuser:abc123       │
└───────┬───────────────────┘
        │
        ▼
┌───────────────────────────┐
│  MANIFESTS REPO           │
│  kustomize edit set image │
│  git commit + push        │
└───────┬───────────────────┘
        │
        ▼
┌───────────────────────────┐
│      ArgoCD               │
│  sees git change,         │
│  kubectl apply            │
└───────┬───────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────┐
│           K8s CONTROL PLANE                            │
│  apiserver → Deployment ctrl → ReplicaSet → Pod        │
│           → Scheduler assigns node                     │
└───────┬────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────┐
│          WORKER NODE                                   │
│  Kubelet → CRI → containerd → runc → container        │
│         → Spring Boot стартует                         │
└───────┬────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────┐
│         READINESS → трафик                             │
│  EndpointSlice → kube-proxy iptables → new Pod         │
└────────────────────────────────────────────────────────┘
```

---

## 1. СБОРКА (BUILD) — точечная схема Gradle

Что происходит при `gradle bootJar` (или CI stage `build`):

```
┌─────────────────────────────────────────────────────────────────────┐
│  ИСХОДНОЕ ДЕРЕВО                                                    │
│                                                                     │
│  isnaknpuser/                                                       │
│  ├── build.gradle              ← план сборки, зависимости           │
│  ├── settings.gradle           ← multi-module setup                 │
│  ├── gradle.properties         ← flags: parallel=true, caching=true │
│  ├── gradlew                   ← wrapper skрипт                     │
│  ├── src/                                                           │
│  │   ├── main/                                                      │
│  │   │   ├── java/kz/isna/knp/user/                                 │
│  │   │   │   ├── UserApplication.java   ← @SpringBootApplication    │
│  │   │   │   ├── controller/                                        │
│  │   │   │   ├── service/                                           │
│  │   │   │   ├── repository/                                        │
│  │   │   │   └── entity/                                            │
│  │   │   └── resources/                                             │
│  │   │       ├── application.yml       ← config (env vars оверайд)  │
│  │   │       ├── application-dev.yml                                │
│  │   │       ├── application-prod.yml                               │
│  │   │       ├── logback-spring.xml                                 │
│  │   │       └── db/changelog/          ← Liquibase migrations      │
│  │   └── test/                                                      │
│  │       └── java/kz/isna/knp/user/                                 │
│  └── Dockerfile                                                     │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ФАЗА 1: DEPENDENCY RESOLUTION                                      │
│                                                                     │
│  Gradle читает build.gradle:                                        │
│    implementation 'org.springframework.boot:spring-boot-starter-web'│
│    implementation 'org.springframework.cloud:spring-cloud-consul... │
│    implementation 'org.postgresql:postgresql'                       │
│    implementation 'org.liquibase:liquibase-core'                    │
│    implementation 'org.springframework.amqp:spring-rabbit'          │
│    testImplementation 'org.testcontainers:postgresql'               │
│                                                                     │
│  → HTTP GET https://nexus.1sc.kz/repository/maven-public/           │
│  → Скачиваются JAR-файлы (кэш в ~/.gradle/caches/)                  │
│  → Разрешаются транзитивные конфликты (Gradle: highest-wins)        │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ФАЗА 2: COMPILE                                                    │
│                                                                     │
│  Task: compileJava                                                  │
│  javac src/main/java/**/*.java → build/classes/java/main/**/*.class │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  UserApplication.java                                    │       │
│  │  @SpringBootApplication                                  │       │
│  │  public class UserApplication {                          │       │
│  │      public static void main(String[] args) {            │       │
│  │          SpringApplication.run(...);                     │       │
│  │      }                                                   │       │
│  │  }                                                       │       │
│  └──────────────────────────────────────────────────────────┘       │
│                          ↓ javac                                    │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  UserApplication.class (bytecode)                        │       │
│  │  Constant pool: строки, ссылки на классы                 │       │
│  │  Bytecode instructions: iload, invokestatic, ...         │       │
│  │  Annotations: @SpringBootApplication                     │       │
│  └──────────────────────────────────────────────────────────┘       │
│                                                                     │
│  Annotation processors:                                             │
│  - Lombok: @Getter, @Setter → generates getters/setters             │
│  - MapStruct: @Mapper → generates DTO conversion code               │
│  - Spring: @ConfigurationProperties processing                      │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ФАЗА 3: PROCESS RESOURCES                                          │
│                                                                     │
│  src/main/resources/*.yml → build/resources/main/*.yml              │
│  Placeholder replacement: @version@ → 1.0.42                        │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ФАЗА 4: RUN TESTS                                                  │
│                                                                     │
│  Task: test (unit)                                                  │
│    - собирает test classpath                                        │
│    - запускает JUnit 5 engine                                       │
│    - Mockito для mock'ов                                            │
│    - отчёт: build/test-results/test/*.xml                           │
│                                                                     │
│  Task: integrationTest                                              │
│    - Testcontainers поднимает Postgres в Docker                     │
│    - @SpringBootTest поднимает полный context                       │
│    - Liquibase применяет migrations                                 │
│    - реальные HTTP endpoints через MockMvc                          │
│    - отчёт: build/test-results/integrationTest/*.xml                │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ФАЗА 5: STATIC ANALYSIS (опционально)                              │
│                                                                     │
│  - checkstyle: код style                                            │
│  - spotbugs: potential bugs (NPE, thread safety)                    │
│  - jacoco: test coverage → coverage.xml                             │
│  - sonarqube: комплексный анализ                                    │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ФАЗА 6: PACKAGING (bootJar)                                        │
│                                                                     │
│  Spring Boot Gradle plugin task: bootJar                            │
│  → build/libs/isnaknpuser-1.0.42.jar (~80-150 MB)                   │
│                                                                     │
│  Структура fat JAR:                                                 │
│  isnaknpuser-1.0.42.jar                                             │
│  ├── META-INF/                                                      │
│  │   └── MANIFEST.MF                                                │
│  │       Main-Class: org.springframework.boot.loader.JarLauncher    │
│  │       Start-Class: kz.isna.knp.user.UserApplication              │
│  ├── BOOT-INF/                                                      │
│  │   ├── classes/                                                   │
│  │   │   ├── kz/isna/knp/user/...      ← твой код (.class)          │
│  │   │   ├── application.yml                                        │
│  │   │   ├── application-prod.yml                                   │
│  │   │   ├── logback-spring.xml                                     │
│  │   │   └── db/changelog/                                          │
│  │   ├── lib/                          ← все transitive JARs        │
│  │   │   ├── spring-boot-3.2.0.jar                                  │
│  │   │   ├── spring-webmvc-6.1.0.jar                                │
│  │   │   ├── spring-cloud-consul-3.1.0.jar                          │
│  │   │   ├── postgresql-42.6.0.jar                                  │
│  │   │   ├── liquibase-core-4.24.0.jar                              │
│  │   │   ├── spring-amqp-3.1.0.jar                                  │
│  │   │   └── ... (~200 JAR-файлов)                                  │
│  │   └── classpath.idx                                              │
│  └── org/springframework/boot/loader/  ← Spring Boot custom loader  │
│      ├── JarLauncher.class                                          │
│      ├── LaunchedURLClassLoader.class                               │
│      └── ...                                                        │
│                                                                     │
│  Layered JAR (для Docker cache):                                    │
│  build/libs/dependencies/              ← редко меняется             │
│  build/libs/spring-boot-loader/                                     │
│  build/libs/snapshot-dependencies/                                  │
│  build/libs/application/               ← ты меняешь на каждом commit│
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │  build/libs/isnaknpuser.jar  │
        │  готов к запуску             │
        └──────────────────────────────┘
```

**Что можно ускорить**:
- Gradle build cache (`org.gradle.caching=true`) — кэш outputs по hash inputs.
- Configuration cache (`--configuration-cache`) — быстрый старт.
- Parallel execution (`org.gradle.parallel=true`).
- Incremental compilation (default).
- CI cache для `.gradle/caches/` между pipeline runs.

---

## 2. DOCKER BUILD — упаковка JAR в OCI image

```
┌────────────────────────────────────────────────────────────────────┐
│  Dockerfile (multi-stage, layered JAR)                             │
│                                                                    │
│  # Stage 1: extract layers                                         │
│  FROM eclipse-temurin:21-jdk AS builder                            │
│  WORKDIR /workspace                                                │
│  COPY build/libs/*.jar app.jar                                     │
│  RUN java -Djarmode=layertools -jar app.jar extract                │
│                                                                    │
│  # Stage 2: runtime                                                │
│  FROM eclipse-temurin:21-jre                                       │
│  WORKDIR /app                                                      │
│  RUN groupadd -r spring && useradd -r -g spring spring             │
│  COPY --from=builder /workspace/dependencies/ ./                   │
│  COPY --from=builder /workspace/spring-boot-loader/ ./             │
│  COPY --from=builder /workspace/snapshot-dependencies/ ./          │
│  COPY --from=builder /workspace/application/ ./                    │
│  USER spring                                                       │
│  EXPOSE 8080                                                       │
│  ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]│
└──────────────────────┬─────────────────────────────────────────────┘
                       │ docker build
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  BUILDKIT execution                                                │
│                                                                    │
│  Layer 0: FROM eclipse-temurin:21-jre                              │
│    → base image (~180 MB) — cache hit почти всегда                 │
│                                                                    │
│  Layer 1: WORKDIR /app                                             │
│    → metadata only, ~0 bytes                                       │
│                                                                    │
│  Layer 2: RUN groupadd + useradd                                   │
│    → filesystem diff, ~1 KB                                        │
│                                                                    │
│  Layer 3: COPY dependencies/                                       │
│    → ~150 MB (Spring, Postgres, RabbitMQ deps)                     │
│    → cache hit если deps не менялись                               │
│                                                                    │
│  Layer 4: COPY spring-boot-loader/                                 │
│    → ~50 KB                                                        │
│                                                                    │
│  Layer 5: COPY snapshot-dependencies/                              │
│    → внутренние SNAPSHOT версии                                    │
│                                                                    │
│  Layer 6: COPY application/                                        │
│    → ~5-15 MB (твой код + resources)                               │
│    → cache miss на каждом commit — только этот слой пересобирается │
│                                                                    │
│  Layer 7: USER spring, EXPOSE, ENTRYPOINT                          │
│    → metadata                                                      │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  ФИНАЛЬНЫЙ IMAGE                                                   │
│                                                                    │
│  registry.1sc.kz/isnaknpuser:1.0.42-abc123                         │
│  registry.1sc.kz/isnaknpuser:latest                                │
│                                                                    │
│  Manifest (JSON):                                                  │
│  {                                                                 │
│    "schemaVersion": 2,                                             │
│    "mediaType": "application/vnd.oci.image.manifest.v1+json",      │
│    "config": {"digest": "sha256:abc..."},                          │
│    "layers": [                                                     │
│      {"digest": "sha256:def...", "size": 180000000},   ← base      │
│      {"digest": "sha256:ghi...", "size": 150000000},   ← deps      │
│      {"digest": "sha256:jkl...", "size": 15000000},    ← app       │
│      ...                                                           │
│    ]                                                               │
│  }                                                                 │
│                                                                    │
│  Total: ~350 MB.                                                   │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ docker push
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  NEXUS REGISTRY                                                    │
│                                                                    │
│  API OCI Distribution Spec:                                        │
│  1. HEAD /v2/isnaknpuser/blobs/<digest>                            │
│     → проверить какие layers уже есть                              │
│  2. POST /v2/isnaknpuser/blobs/uploads/                            │
│     → начать upload только отсутствующих layers                    │
│  3. PATCH → чанки → PUT (финализация)                              │
│  4. PUT /v2/isnaknpuser/manifests/1.0.42-abc123                    │
│     → сохранить manifest, поставить tag                            │
│                                                                    │
│  Реально пересылается: только application layer (~15 MB).          │
│  Base image (180 MB) и deps (150 MB) уже в registry.               │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. ЛОКАЛЬНЫЙ ЗАПУСК (dev-run) — три способа

### 3.1 Способ 1: IDE через IntelliJ

```
┌────────────────────────────────────────────────────────────────────┐
│  IntelliJ IDEA                                                     │
│  ├── Run Configuration: UserApplication                            │
│  ├── VM options: -Xmx2g -Dspring.profiles.active=local             │
│  ├── Program arguments: (empty)                                    │
│  └── Environment variables:                                        │
│      SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/knp    │
│      SPRING_RABBITMQ_HOST=localhost                                │
│      CONSUL_HOST=localhost                                         │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ Play button
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  IDEA запускает:                                                   │
│  java -Xmx2g -Dspring.profiles.active=local ^                      │
│       -cp build/classes:build/resources:...:*.jar ^                │
│       kz.isna.knp.user.UserApplication                             │
│                                                                    │
│  → JVM starts                                                      │
│  → Spring Boot loads                                               │
│  → application-local.yml активен                                   │
│  → Порты 8080 (HTTP), 8081 (management/actuator)                   │
└────────────────────────────────────────────────────────────────────┘

Зависимости для локального dev:
  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
  │  Postgres     │   │  RabbitMQ     │   │  Consul       │
  │  :5432        │   │  :5672        │   │  :8500        │
  └───────────────┘   └───────────────┘   └───────────────┘
      docker-compose up (см. 3.3)
```

### 3.2 Способ 2: gradle bootRun

```
$ ./gradlew bootRun --args='--spring.profiles.active=local'

┌────────────────────────────────────────────────────────────────────┐
│  Gradle task: bootRun                                              │
│                                                                    │
│  1. Проверить dependencies (up-to-date checks)                     │
│  2. compile (если source изменился)                                │
│  3. processResources                                               │
│  4. Launch:                                                        │
│     java -cp <computed classpath> kz.isna.knp.user.UserApplication │
│                                                                    │
│  Плюс: не нужен IDE, работает в CLI.                               │
│  Минус: hot reload только через Spring Boot DevTools.              │
└────────────────────────────────────────────────────────────────────┘
```

### 3.3 Способ 3: docker-compose (полный стек)

```
docker-compose.yml в корне проекта:

┌────────────────────────────────────────────────────────────────────┐
│  services:                                                         │
│    postgres:                                                       │
│      image: postgres:16                                            │
│      environment:                                                  │
│        POSTGRES_DB: knp                                            │
│        POSTGRES_USER: knp                                          │
│        POSTGRES_PASSWORD: knp                                      │
│      ports: ["5432:5432"]                                          │
│      volumes: [pgdata:/var/lib/postgresql/data]                    │
│                                                                    │
│    rabbitmq:                                                       │
│      image: rabbitmq:3.13-management                               │
│      ports: ["5672:5672", "15672:15672"]                           │
│                                                                    │
│    consul:                                                         │
│      image: hashicorp/consul:1.17                                  │
│      command: agent -dev -client=0.0.0.0                           │
│      ports: ["8500:8500"]                                          │
│                                                                    │
│    keycloak:                                                       │
│      image: quay.io/keycloak/keycloak:22                           │
│      command: start-dev                                            │
│      environment:                                                  │
│        KEYCLOAK_ADMIN: admin                                       │
│        KEYCLOAK_ADMIN_PASSWORD: admin                              │
│      ports: ["8180:8080"]                                          │
│                                                                    │
│    isnaknpuser:                                                    │
│      build: .                                                      │
│      environment:                                                  │
│        SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/knp  │
│        SPRING_RABBITMQ_HOST: rabbitmq                              │
│        CONSUL_HOST: consul                                         │
│      depends_on: [postgres, rabbitmq, consul]                      │
│      ports: ["8080:8080"]                                          │
└────────────────────────────────────────────────────────────────────┘

$ docker-compose up -d           → все зависимости поднимаются
$ docker-compose logs -f isnaknpuser
```

### 3.4 Что происходит внутри JVM при старте Spring Boot

```
Time  Event
────  ────────────────────────────────────────────────
0.0s  java -jar isnaknpuser.jar
0.0s  JVM launcher: mmap heap, init GC (G1 default)
0.1s  JarLauncher.main() — custom classloader setup
0.5s  SpringApplication.run()
      │
      ├─ 0.5s Banner printed
      ├─ 1.0s Application starting
      ├─ 2.0s Loading application.yml + application-{profile}.yml
      ├─ 3.0s @ComponentScan → находит @Service, @Component, @Controller
      ├─ 5.0s @EnableAutoConfiguration → Spring Boot auto-config
      │       - JPA/Hibernate detected → DataSource bean
      │       - Consul detected → discovery client
      │       - RabbitMQ detected → ConnectionFactory
      │       - Actuator → health endpoints
      ├─ 8.0s BeanFactory: создание всех singleton beans
      │       - @Bean methods выполняются
      │       - @Autowired инъекции
      │       - @PostConstruct вызовы
      ├─ 10.0s DataSource bean:
      │        - HikariCP создаёт connection pool
      │        - Первое соединение к Postgres
      │        - Liquibase проверяет migrations
      │        - Если есть новые → применяет
      ├─ 15.0s Consul registration:
      │        - PUT /v1/agent/service/register
      │        - Health check registered
      ├─ 16.0s RabbitMQ connection:
      │        - AMQP handshake
      │        - Exchange/queue declaration
      ├─ 17.0s Tomcat/Undertow starts
      │        - Bind :8080
      │        - Servlet dispatcher готов
      ├─ 18.0s Actuator management port :8081
      ├─ 20.0s ApplicationReadyEvent published
      │        - @EventListener хендлеры
      │        - Warmup tasks
      └─ 20.0s READY — принимает HTTP запросы
                    /actuator/health/readiness → 200 UP
```

---

## 4. CI/CD PIPELINE — точечная схема .gitlab-ci.yml

```
git push origin release-ISNA2-23651
        │
        ▼ (webhook / long polling)
        │
┌───────────────────────────────────────────────────────────────────┐
│  GITLAB SERVER — post-receive hook                                │
│                                                                   │
│  1. Обновление refs в БД                                          │
│  2. Проверка protected branches                                   │
│  3. Чтение .gitlab-ci.yml из push HEAD                            │
│  4. Создание Pipeline object                                      │
│  5. Создание Jobs по stages                                       │
│  6. Первые jobs (validate) → status pending                       │
└──────────────────────┬────────────────────────────────────────────┘
                       │
                       ▼
    ┌──────────────────────────────────────────┐
    │       .gitlab-ci.yml — stages            │
    ├──────────────────────────────────────────┤
    │  stages:                                 │
    │    - validate                            │
    │    - build                               │
    │    - test                                │
    │    - security                            │
    │    - package                             │
    │    - deploy-staging                      │
    │    - integration-test                    │
    │    - deploy-prod                         │
    └──────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
  STAGE 1: VALIDATE  ────────────────────────────────  ~15 sec
═══════════════════════════════════════════════════════════════════
  ┌──────────────────────────────────────────────────────┐
  │  Runner: k8s-executor Pod поднимается                │
  │  Image: build-tools:jdk21-gradle8                    │
  │  Script:                                             │
  │    gradle checkstyleMain spotbugsMain --no-daemon    │
  │                                                      │
  │  Проверки:                                           │
  │    - checkstyle: naming, formatting                  │
  │    - spotbugs: NPE, thread safety                    │
  │                                                      │
  │  Fail → pipeline stops, автор получает уведомление   │
  └──────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
  STAGE 2: BUILD  ───────────────────────────────────  ~45 sec
═══════════════════════════════════════════════════════════════════
  ┌──────────────────────────────────────────────────────┐
  │  gradle assemble --no-daemon                         │
  │  Cache: .gradle/caches (per-branch)                  │
  │                                                      │
  │  Артефакты (uploaded в GitLab):                      │
  │    - build/libs/*.jar        (для следующих jobs)    │
  │    - expire_in: 1 day                                │
  └──────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
  STAGE 3: TEST  ────────────────────────────────────  ~2-4 min
═══════════════════════════════════════════════════════════════════
  ┌──────────────────────────────────────┬───────────────────────┐
  │  Job: unit-test                      │  Job: integration-test│
  │  parallel в stage                    │  parallel в stage     │
  ├──────────────────────────────────────┼───────────────────────┤
  │  gradle test jacocoTestReport        │  gradle integrationTest│
  │                                      │  services:            │
  │  Артефакты:                          │    - postgres:16      │
  │    junit: test-results/test/*.xml    │    - rabbitmq:3.13    │
  │    coverage: jacoco/coverage.xml     │  variables:           │
  │  when: always (report даже при fail) │    SPRING_DS_URL:     │
  │                                      │      jdbc:postgresql: │
  │                                      │      //db:5432/test   │
  └──────────────────────────────────────┴───────────────────────┘

═══════════════════════════════════════════════════════════════════
  STAGE 4: SECURITY  ────────────────────────────────  ~45 sec
═══════════════════════════════════════════════════════════════════
  ┌──────────────────────────────────────┬───────────────────────┐
  │  Job: trivy-scan (dependencies)      │  Job: sonarqube       │
  ├──────────────────────────────────────┼───────────────────────┤
  │  trivy fs --severity HIGH,CRITICAL . │  gradle sonarqube     │
  │  gradle dependencyCheckAnalyze       │    -Dsonar.host.url=  │
  │                                      │      $SONAR_URL       │
  │  Fail → pipeline stops               │  → отчёт в SonarQube  │
  └──────────────────────────────────────┴───────────────────────┘

═══════════════════════════════════════════════════════════════════
  STAGE 5: PACKAGE  ─────────────────────────────────  ~1-2 min
═══════════════════════════════════════════════════════════════════
  ┌──────────────────────────────────────────────────────┐
  │  Job: build-image                                    │
  │  image: docker:24                                    │
  │  services:                                           │
  │    - docker:24-dind                                  │
  │                                                      │
  │  Script:                                             │
  │  1. docker login -u $CI_USER -p $CI_TOKEN registry   │
  │  2. docker build \                                   │
  │       --cache-from registry.1sc.kz/user:latest \     │
  │       -t registry.1sc.kz/user:$CI_COMMIT_SHORT_SHA \ │
  │       .                                              │
  │  3. trivy image --severity HIGH,CRITICAL             │
  │       registry.1sc.kz/user:$CI_COMMIT_SHORT_SHA      │
  │  4. docker push registry.1sc.kz/user:...             │
  │                                                      │
  │  Push шлёт только application layer (~15 MB).        │
  └──────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
  STAGE 6: DEPLOY-STAGING  ──────────────────────────  ~30 sec
═══════════════════════════════════════════════════════════════════
  ┌──────────────────────────────────────────────────────┐
  │  Только для веток [main, release/*]                  │
  │                                                      │
  │  1. git clone $MANIFESTS_REPO                        │
  │  2. cd manifests/apps/isnaknpuser/staging            │
  │  3. kustomize edit set image \                       │
  │       myapp=registry.1sc.kz/user:$CI_COMMIT_SHORT_SHA│
  │  4. git commit -am "Staging: user $CI_COMMIT_SHA"    │
  │  5. git push                                         │
  │                                                      │
  │  → ArgoCD получит webhook или увидит через polling   │
  └──────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
  STAGE 7: INTEGRATION-TEST  ────────────────────────  ~2-5 min
═══════════════════════════════════════════════════════════════════
  ┌──────────────────────────────────────────────────────┐
  │  Job: smoke-test                                     │
  │                                                      │
  │  1. ./wait-for-deploy.sh isnaknpuser staging 300     │
  │     → ждёт пока ArgoCD засинкает + Pods Ready        │
  │  2. ./smoke.sh https://staging.knp.gov.kz/api        │
  │     → критичные endpoints работают                   │
  └──────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
  STAGE 8: DEPLOY-PROD  ─────────────────────────────  manual
═══════════════════════════════════════════════════════════════════
  ┌──────────────────────────────────────────────────────┐
  │  when: manual                                        │
  │  only: [master]                                      │
  │                                                      │
  │  → Кнопка в GitLab UI                                │
  │  → Тот же flow: kustomize edit + git push            │
  │  → в manifests/apps/isnaknpuser/prod/                │
  │  → ArgoCD deploys в prod cluster                     │
  └──────────────────────────────────────────────────────┘
```

---

## 5. GITOPS DEPLOY — как ArgoCD доставляет в кластер

```
CI сделал git push в manifests-repo:
        │
        ▼
┌────────────────────────────────────────────────────────────────────┐
│  MANIFESTS REPO                                                    │
│  gitlab.1sc.kz/infra/manifests.git                                 │
│                                                                    │
│  manifests/                                                        │
│  ├── base/                          ← общая конфигурация           │
│  │   ├── deployment.yaml                                           │
│  │   ├── service.yaml                                              │
│  │   ├── configmap.yaml                                            │
│  │   └── kustomization.yaml                                        │
│  └── apps/isnaknpuser/                                             │
│      ├── staging/                                                  │
│      │   ├── kustomization.yaml   ← image: user:abc123 (updated)   │
│      │   └── patches/                                              │
│      │       ├── replicas.yaml    (replicas: 2)                    │
│      │       └── resources.yaml   (memory: 1Gi)                    │
│      └── prod/                                                     │
│          ├── kustomization.yaml   ← image: user:def456 (last prod) │
│          └── patches/                                              │
│              ├── replicas.yaml    (replicas: 4)                    │
│              └── resources.yaml   (memory: 2Gi)                    │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼ (webhook → ArgoCD или polling каждые 3 мин)
┌────────────────────────────────────────────────────────────────────┐
│  ArgoCD                                                            │
│                                                                    │
│  Application myapp-staging:                                        │
│    spec:                                                           │
│      source:                                                       │
│        repoURL: gitlab.1sc.kz/infra/manifests.git                  │
│        path: apps/isnaknpuser/staging                              │
│        targetRevision: HEAD                                        │
│      destination:                                                  │
│        server: https://kubernetes.default.svc                      │
│        namespace: knp                                              │
│      syncPolicy:                                                   │
│        automated:                                                  │
│          prune: true                                               │
│          selfHeal: true                                            │
│                                                                    │
│  Reconcile loop:                                                   │
│  1. git fetch                                                      │
│  2. kustomize build apps/isnaknpuser/staging → финальные манифесты │
│  3. compare с текущим состоянием в etcd cluster                    │
│  4. если diff:                                                     │
│     - PreSync hooks (миграции БД через Job)                        │
│     - Sync: kubectl apply манифестов                               │
│     - PostSync hooks (smoke tests, notifications)                  │
│  5. wait для health probes                                         │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  APISERVER                                                         │
│                                                                    │
│  POST /apis/apps/v1/namespaces/knp/deployments/isnaknpuser         │
│                                                                    │
│  1. Authentication (ArgoCD ServiceAccount → JWT токен)             │
│  2. Authorization (RBAC: может ли SA обновлять deployments в knp?) │
│  3. Mutating admission webhooks:                                   │
│     - Istio sidecar injection (если включена mesh)                 │
│     - Default resources injection                                  │
│  4. Schema validation                                              │
│  5. Validating admission webhooks:                                 │
│     - PodSecurityPolicy / OPA Gatekeeper policies                  │
│  6. etcd write (Raft consensus)                                    │
│  7. resourceVersion++ на объекте                                   │
│  8. Return 200 to ArgoCD                                           │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
```

---

## 6. K8s POD LIFECYCLE — от apiserver до Ready

```
apiserver обновил Deployment.spec.template (image changed)
        │
        │ watch event
        ▼
┌────────────────────────────────────────────────────────────────────┐
│  DEPLOYMENT CONTROLLER                                             │
│                                                                    │
│  1. Читает spec.template                                           │
│  2. Считает hash pod template                                      │
│  3. Ищет ReplicaSets с этим hash:                                  │
│     - Нет → создаёт новый RS (RS-new, replicas: 0)                 │
│     - Есть → использует (для rollback)                             │
│  4. Scale up RS-new до 1 (maxSurge: 1)                             │
│  5. Deployment.status.newReplicaSet = "RS-new"                     │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ watch event: RS updated
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  REPLICASET CONTROLLER                                             │
│                                                                    │
│  1. Считает currentPods (0), desired (1)                           │
│  2. Создаёт Pod object в apiserver:                                │
│     - name: isnaknpuser-abc-xyz                                    │
│     - ownerRef: RS-new                                             │
│     - spec.nodeName: "" (unscheduled)                              │
│     - labels: app=isnaknpuser, pod-template-hash=...               │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ watch event: Pod created (unscheduled)
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  SCHEDULER                                                         │
│                                                                    │
│  Filter phase — какие ноды feasible:                               │
│    - resource requests (memory, cpu)                               │
│    - taints/tolerations                                            │
│    - node affinity / anti-affinity                                 │
│    - PVC access                                                    │
│                                                                    │
│  Score phase — 0-100 для каждой feasible node:                     │
│    - LeastRequestedPriority (менее загруженная = выше)             │
│    - BalancedResourceAllocation                                    │
│    - NodeAffinityPriority                                          │
│                                                                    │
│  Выбор: worker-3 (score 87)                                        │
│                                                                    │
│  Bind: PATCH pod, spec.nodeName = "worker-3"                       │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ watch event на worker-3
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  KUBELET на worker-3                                               │
│                                                                    │
│  syncLoop:                                                         │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ 1. IMAGE PULL                                        │          │
│  │    CRI: PullImage("registry.1sc.kz/user:abc123")     │          │
│  │    containerd:                                       │          │
│  │      - HTTP GET registry manifest                    │          │
│  │      - HEAD blobs — какие layers уже локально?       │          │
│  │      - GET отсутствующие layers                      │          │
│  │      - Распаковка (gzip → filesystem)                │          │
│  │    Cache hit: 0.5-1 сек                              │          │
│  │    Cold pull: 10-60 сек                              │          │
│  └──────────────────────────────────────────────────────┘          │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ 2. POD SANDBOX (pause container)                     │          │
│  │    CRI: RunPodSandbox()                              │          │
│  │      - containerd создаёт pause container            │          │
│  │      - Настраивает Linux namespaces:                 │          │
│  │        * network (свой сетевой стек)                 │          │
│  │        * PID (свой PID 1)                            │          │
│  │        * IPC (свои mq, sem)                          │          │
│  │      - CNI plugin (Calico/Flannel):                  │          │
│  │        * veth pair                                   │          │
│  │        * IP assignment (10.244.1.5)                  │          │
│  │        * Routes                                      │          │
│  │        * NetworkPolicy iptables                      │          │
│  └──────────────────────────────────────────────────────┘          │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ 3. VOLUMES MOUNT                                     │          │
│  │    ConfigMap → files в emptyDir                      │          │
│  │    Secret → tmpfs (in-memory)                        │          │
│  │    ServiceAccount → projected volume с JWT           │          │
│  │    (PVC если есть → CSI attach + mount, 30-120 сек)  │          │
│  └──────────────────────────────────────────────────────┘          │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ 4. CONTAINER CREATE                                  │          │
│  │    CRI: CreateContainer()                            │          │
│  │      containerd → runc:                              │          │
│  │        - создать OCI spec (config.json)              │          │
│  │        - clone() syscall — новые namespaces          │          │
│  │        - mount rootfs (union of layers)              │          │
│  │        - pivot_root                                  │          │
│  │        - setresuid (non-root: spring user)           │          │
│  │        - установка cgroups (memory, cpu limits)      │          │
│  └──────────────────────────────────────────────────────┘          │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ 5. CONTAINER START                                   │          │
│  │    CRI: StartContainer()                             │          │
│  │      runc start:                                     │          │
│  │        - execve() main процесса                      │          │
│  │        - ENTRYPOINT: java org.springframework...     │          │
│  │      → PID 1 в контейнере запущен                    │          │
│  └──────────────────────────────────────────────────────┘          │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  ВНУТРИ КОНТЕЙНЕРА                                                 │
│                                                                    │
│  T=0.0s:  JVM launcher (java binary)                               │
│  T=0.5s:  Spring Boot JarLauncher.main()                           │
│  T=1.0s:  application.yml + application-prod.yml загружены         │
│           (SPRING_PROFILES_ACTIVE=prod из env)                     │
│  T=3.0s:  @ComponentScan                                           │
│  T=5.0s:  @EnableAutoConfiguration                                 │
│                                                                    │
│  T=8.0s:  Bean creation:                                           │
│           - DataSource (HikariCP):                                 │
│             SPRING_DATASOURCE_URL=jdbc:postgresql://               │
│               postgres-svc.knp.svc.cluster.local:5432/knp          │
│             → HikariCP creates 10-20 connections                   │
│           - Liquibase migrations check                             │
│                                                                    │
│  T=12.0s: Consul registration:                                     │
│           CONSUL_HOST=consul.consul.svc.cluster.local              │
│           → PUT /v1/agent/service/register                         │
│                                                                    │
│  T=14.0s: RabbitMQ connection:                                     │
│           SPRING_RABBITMQ_HOST=rabbitmq.rabbit.svc                 │
│           → AMQP handshake                                         │
│           → Queues/exchanges declared                              │
│                                                                    │
│  T=16.0s: Keycloak OIDC discovery:                                 │
│           → GET /realms/knp/.well-known/openid-configuration       │
│           → JWKS keys cached                                       │
│                                                                    │
│  T=18.0s: Tomcat starts, bind :8080                                │
│  T=20.0s: ApplicationReadyEvent                                    │
│  T=20.0s: /actuator/health/readiness → {"status":"UP"}             │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
```

---

## 7. READINESS → TRAFFIC — последняя миля

```
Kubelet опрашивает readiness probe каждые 10 сек:
        │
        │  GET http://10.244.1.5:8080/actuator/health/readiness
        │  Response: 200 {"status": "UP"}
        ▼
┌────────────────────────────────────────────────────────────────────┐
│  KUBELET                                                           │
│  Обновляет Pod.status:                                             │
│    conditions:                                                     │
│      - type: Ready                                                 │
│        status: "True"                                              │
│    containerStatuses[0].ready: true                                │
│                                                                    │
│  → apiserver update                                                │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ watch event: Pod became Ready
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  ENDPOINTSLICE CONTROLLER                                          │
│                                                                    │
│  Видит: Pod isnaknpuser-abc-xyz labels совпадают с Service         │
│  selector (app=isnaknpuser)                                        │
│                                                                    │
│  Update endpointslices/isnaknpuser-<hash>:                         │
│    endpoints:                                                      │
│      - addresses: [10.244.1.5]                                     │
│        conditions: {ready: true}                                   │
│        targetRef: pod/isnaknpuser-abc-xyz                          │
│                                                                    │
│  → etcd write                                                      │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ watch event на всех нодах
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  KUBE-PROXY (DaemonSet, на каждой ноде)                            │
│                                                                    │
│  Sync loop rebuilds iptables (или ipvs) rules:                     │
│                                                                    │
│  -A KUBE-SERVICES -d 10.96.5.10/32 -p tcp --dport 80 \             │
│     -j KUBE-SVC-USER                                               │
│  -A KUBE-SVC-USER -m statistic --mode random \                     │
│     --probability 0.25 -j KUBE-SEP-P1                              │
│  -A KUBE-SVC-USER -m statistic --mode random \                     │
│     --probability 0.33 -j KUBE-SEP-P2                              │
│  -A KUBE-SVC-USER -m statistic --mode random \                     │
│     --probability 0.50 -j KUBE-SEP-P3                              │
│  -A KUBE-SVC-USER -j KUBE-SEP-P4                                   │
│  -A KUBE-SEP-P1 -j DNAT --to-destination 10.244.1.5:8080           │
│  -A KUBE-SEP-P2 -j DNAT --to-destination 10.244.2.7:8080           │
│  ...                                                               │
│                                                                    │
│  → iptables-restore --wait (atomic swap)                           │
└──────────────────────┬─────────────────────────────────────────────┘
                       │ propagation ко всем нодам (1-5 сек)
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  ТРАФИК ИДЁТ В НОВЫЙ POD                                           │
│                                                                    │
│  Client (другой сервис или Ingress):                               │
│    GET http://isnaknpuser.knp.svc.cluster.local/api/users/1        │
│                                                                    │
│  DNS resolution: CoreDNS →                                         │
│    isnaknpuser.knp.svc.cluster.local → 10.96.5.10 (ClusterIP)      │
│                                                                    │
│  TCP connect to 10.96.5.10:80                                      │
│    → kube-proxy iptables DNAT                                      │
│    → routed случайно на один из Pod'ов (10.244.1.5:8080)           │
│    → HTTP GET /api/users/1                                         │
│    → Spring Boot @RestController обрабатывает                      │
│    → JPA → Hikari → Postgres SELECT                                │
│    → response JSON                                                 │
└────────────────────────────────────────────────────────────────────┘
```

---

## 8. ROLLING UPDATE — как заменяется старая версия

```
Deployment controller видит: newRS теперь имеет 1 Ready Pod
Стратегия: maxSurge=1, maxUnavailable=0, replicas=4

Состояние сейчас: 4 v1 + 1 v2 = 5 Pod'ов
Условие соблюдено: (5 - 0) ≥ replicas
Можно scale down старый RS

Step 1 (T=30s):
┌────────────────────────────────────────────────────────────────────┐
│  Scale RS-old с 4 до 3                                             │
│  → ReplicaSet controller выбирает Pod для удаления                 │
│  → Delete pod/isnaknpuser-old-1 (soft: deletionTimestamp)          │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│  Kubelet видит Pod Terminating:                                    │
│                                                                    │
│  1. Pod status: Terminating                                        │
│  2. EndpointSlice controller REMOVES 10.244.1.6 из endpoints       │
│     → kube-proxy на всех нодах обновляет iptables                  │
│     → propagation 1-5 сек                                          │
│  3. Kubelet execute preStop hook:                                  │
│     lifecycle:                                                     │
│       preStop:                                                     │
│         exec: {command: ["sh", "-c", "sleep 10"]}                  │
│     → 10 сек pod ещё живой, но новые запросы не идут               │
│     → in-flight запросы завершаются                                │
│  4. SIGTERM → Spring Boot graceful shutdown:                       │
│     spring.lifecycle.timeout-per-shutdown-phase: 25s               │
│     - Прекращает принимать новые connections                       │
│     - Ждёт in-flight requests                                      │
│     - Закрывает datasource connections                             │
│     - Consul deregistration                                        │
│  5. Process exits 0                                                │
│  6. CRI: StopContainer, RemovePodSandbox                           │
│  7. Pod object removed                                             │
└────────────────────────────────────────────────────────────────────┘

Step 2 (T=45s): Scale RS-new с 1 до 2, поднимается ещё один v2
Step 3 (T=75s): Scale RS-old с 3 до 2
Step 4 (T=90s): Scale RS-new с 2 до 3
...
Step 8 (T=180s): Все 4 v2 Ready, RS-old с 0 replicas
                                     
Rolling complete: 4 × v2 Pod'ов принимают трафик.
Zero downtime: минимум 4 Ready Pod'а всё это время.
```

---

## 9. TIMELINE — от git push до трафика

```
 Time    Event
────    ──────────────────────────────────────────────────────
 0:00   git push origin release-ISNA2-XXXXX
 0:01   GitLab receives, post-receive hook fires
 0:02   Pipeline created, validate job pending
 0:03   Runner picks up validate
 0:15   validate done (checkstyle + spotbugs)
 0:16   Stage build starts
 0:45   Gradle compile done, jar в artifacts
 0:46   Stage test starts
 2:30   Unit tests done
 3:45   Integration tests done (Testcontainers Postgres)
 3:46   Stage security (Trivy + OWASP)
 4:30   Security done
 4:31   Stage package (docker build)
 4:35   Docker build done (layer cache hit)
 4:40   Trivy image scan done
 4:50   Docker push (только application layer, ~15MB)
 4:51   Stage deploy-staging
 4:53   Manifest repo updated, git push
 4:53   ArgoCD webhook triggered
 4:54   ArgoCD sync starts
 4:54   apiserver applies Deployment
 4:54   Deployment controller creates new ReplicaSet
 4:54   RS controller creates Pod
 4:54   Scheduler binds to worker-3
 4:55   Kubelet pulls image (cache hit)
 4:56   Container starts, JVM launch
 5:20   Spring Boot fully started, readiness Ready
 5:20   EndpointSlice updated
 5:21   kube-proxy iptables propagated
 5:22   Rolling continues: old Pod terminated
 5:45   2nd new Pod Ready
 6:10   3rd new Pod Ready
 6:35   4th new Pod Ready, rolling complete
 6:36   Smoke test job
 6:45   Smoke tests pass
 ────
 End-to-end: ~7 минут для staging.
 Prod — плюс manual approve (минуты-часы).
```

---

## 10. КАРТА ЗАВИСИМОСТЕЙ ВНУТРИ КЛАСТЕРА

```
                          ┌─────────────────┐
                          │  Ingress /      │
                          │  API Gateway    │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │  isnaknpuser    │  ← ваш сервис
                          │  4 pods         │
                          └────────┬────────┘
                                   │
             ┌──────────┬──────────┼──────────┬──────────┐
             │          │          │          │          │
             ▼          ▼          ▼          ▼          ▼
      ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
      │Postgres  │ │RabbitMQ  │ │Consul    │ │Keycloak  │ │MinIO     │
      │(StatefulS│ │(Cluster) │ │(3 nodes) │ │(SSO)     │ │(S3-compat│
      │ 3 replicas│ │           │ │          │ │          │ │ storage) │
      └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
      SPRING_DS   SPRING_RMQ  CONSUL_HOST   Keycloak      MinIO client
      _URL        _HOST         _URL        через OIDC    для файлов


  Namespace структура:
  ┌────────────────────────────────────────────────────────────────┐
  │  namespace: knp                                                │
  │  ├── isnaknpuser (Deployment, 4 replicas)                      │
  │  ├── isnaknpgateway (Deployment, 2 replicas)                   │
  │  ├── isnaknpsupport (Deployment)                               │
  │  ├── postgres-svc (ExternalName → RDS)                         │
  │  ├── rabbit-svc (ExternalName → RabbitMQ cluster)              │
  │  └── ConfigMaps + Secrets                                      │
  │                                                                │
  │  namespace: consul                                             │
  │  └── Consul StatefulSet                                        │
  │                                                                │
  │  namespace: keycloak                                           │
  │  └── Keycloak StatefulSet                                      │
  │                                                                │
  │  namespace: argocd                                             │
  │  └── ArgoCD Controllers                                        │
  └────────────────────────────────────────────────────────────────┘
```

---

## 11. ЧТО ГДЕ ХРАНИТСЯ

```
┌─────────────────────────────────────────────────────────────────────┐
│  Слой                          Где                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Source code                   gitlab.1sc.kz/isna/isnaknpuser       │
│  Manifests                     gitlab.1sc.kz/infra/manifests        │
│  Maven dependencies            nexus.1sc.kz/maven-public            │
│  Docker images                 registry.1sc.kz (Nexus Docker repo)  │
│  Helm charts                   nexus.1sc.kz/helm-repo               │
│  Build cache                   GitLab CI cache / Nexus              │
│  Secrets (K8s)                 etcd (encrypted at rest)             │
│  Application config            ConfigMap + application.yml          │
│  Runtime data                  PostgreSQL, MinIO                    │
│  Message queues                RabbitMQ                             │
│  Service discovery             Consul                               │
│  Auth (JWT signing)            Keycloak (realm keys)                │
│  Logs                          Loki / ELK                           │
│  Metrics                       Prometheus + Grafana                 │
│  Traces                        Jaeger / Tempo                       │
│  K8s state                     etcd (в control plane)               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 12. ГДЕ ЧТО ДИАГНОСТИРОВАТЬ ПРИ ИНЦИДЕНТЕ

```
Симптом                           →   Где смотреть
─────────────────────────────────────────────────────────────────
git push отклонён                 →   GitLab UI (protected branch?)
Pipeline не стартовал             →   GitLab Sidekiq / hooks logs
Build упал                        →   Job logs, gradle output
Тесты упали                       →   JUnit XML в artifacts
Docker build упал                 →   Buildkit logs, Dockerfile
Docker push упал                  →   registry auth / disk space
ArgoCD не sync                    →   ArgoCD UI → Application events
Pod ImagePullBackOff              →   kubectl describe pod → Events
                                       imagePullSecrets настроены?
Pod CrashLoopBackOff              →   kubectl logs --previous
                                       exit code (137=OOM, 143=SIGTERM)
Pod Pending навсегда              →   kubectl describe → FailedScheduling
                                       resources / taints / PVC
Readiness never passes            →   kubectl port-forward + curl
                                       kubectl logs — что не запустилось?
OOMKilled (137)                   →   Xmx vs limits.memory
                                       NMT: jcmd VM.native_memory
Rolling stuck                     →   kubectl get pods -l app=X
                                       new Pods не Ready?
Трафик не идёт в новый Pod        →   EndpointSlice → есть Pod IP?
                                       kube-proxy на клиентской ноде
5xx во время rolling              →   preStop sleep настроен?
                                       graceful shutdown в Spring?
High latency                      →   Prometheus JVM metrics
                                       GC pauses? Direct memory?
                                       Downstream slow (БД, external API)?
```

---

## Итог

Полный путь от `git push` до трафика в prod'е — это ~10 отдельных систем, каждая со своим протоколом и failure modes. Понимая полную схему точечно:

**Build** — Gradle через 6 фаз: dependency resolution → compile → resources → test → static analysis → packaging. Результат: fat JAR с nested структурой BOOT-INF.

**Docker** — multi-stage build с layered JAR. Финальный image ~350 MB, но docker push шлёт только application layer (~15 MB).

**Локальный запуск** — 3 способа: IDE, `gradle bootRun`, docker-compose с полным стеком зависимостей.

**CI/CD** — 8 stages в `.gitlab-ci.yml`: validate → build → test → security → package → deploy-staging → integration-test → deploy-prod (manual).

**GitOps через ArgoCD** — CI не имеет kubectl прав. Обновляет только manifest в git. ArgoCD внутри кластера синхронизирует state.

**K8s Pod lifecycle** — apiserver → Deployment ctrl → ReplicaSet ctrl → Pod → Scheduler → Kubelet → CRI → containerd → runc → syscalls.

**Readiness → трафик** — EndpointSlice → kube-proxy iptables → 1-5 сек propagation на все ноды.

**Rolling update** — maxSurge=1, maxUnavailable=0 + preStop sleep + graceful shutdown = zero downtime.

**End-to-end**: ~7 минут для staging автоматически. Prod — плюс manual approve.

Практика: возьми любой микросервис КНП, пройди по этой схеме. Найди свой .gitlab-ci.yml, свой Dockerfile, свой Deployment manifest, свой ArgoCD Application. Понимая всю цепочку — любой инцидент диагностируется за минуты.
