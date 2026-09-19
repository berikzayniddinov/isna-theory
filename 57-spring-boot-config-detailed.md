# 57. Конфигурация Spring Boot (yml, env, приоритеты)

Все способы конфигурации Spring Boot приложения. Приоритеты. Реализация.

---

## 1. Проблема

Приложение должно работать по-разному в:
- **Local dev** — H2, HTTP :8080, DEBUG logs.
- **CI/test** — Testcontainers PG.
- **Preprod** — реальный PG, но test data.
- **Prod** — реальный PG, real users.

Код **одинаковый**, отличается только configuration.

Spring Boot даёт мощный механизм: **PropertySource** — иерархия источников конфига.

---

## 2. Что такое PropertySource

Абстракция «источник key=value».

Каждый PropertySource — либо файл, либо env, либо ещё что-то.

Приложение видит их через `Environment` bean:
```java
@Autowired Environment env;

String url = env.getProperty("spring.datasource.url");
```

Spring перебирает source'ы по приоритету → возвращает первое найденное значение.

---

## 3. Все источники PropertySource

**Полный порядок** (от **высшего** приоритета к **низшему**):

1. **DevTools global settings** (`~/.spring-boot-devtools.properties`).
2. **`@TestPropertySource`** аннотации в тестах.
3. **`SpringApplication.setDefaultProperties`** (programmatic).
4. **`@SpringBootTest` properties**.
5. **Command-line arguments** (`--server.port=9090`).
6. **JSON в `SPRING_APPLICATION_JSON`** env var.
7. **`ServletConfig` init parameters**.
8. **`ServletContext` init parameters**.
9. **JNDI attributes** (`java:comp/env`).
10. **Java System properties** (`-Dserver.port=9090`).
11. **OS environment variables** (`SERVER_PORT=9090`).
12. **Профиль-специфичные properties снаружи jar** (`./application-prod.yml`).
13. **Профиль-специфичные properties внутри jar** (`classpath:application-prod.yml`).
14. **Общие properties снаружи jar** (`./application.yml`).
15. **Общие properties внутри jar** (`classpath:application.yml`).
16. **`@PropertySource`** на `@Configuration`.
17. **Default properties** (`SpringApplication.setDefaultProperties`).

Чем **выше** — тем **важнее** (перекрывает нижнее).

### 3.1 На практике

Обычно в порядке важности сверху:
1. **Command-line** (`--foo=bar`).
2. **Env vars** (`FOO=bar`).
3. **JVM props** (`-Dfoo=bar`).
4. **`application-{profile}.yml`** (external / classpath).
5. **`application.yml`** (external / classpath).
6. Defaults.

---

## 4. application.yml / application.properties

Основные файлы конфига в `src/main/resources/`.

### 4.1 YAML

```yaml
server:
  port: 8080
  compression:
    enabled: true

spring:
  application:
    name: isna-knp
  datasource:
    url: jdbc:postgresql://localhost:5432/knp
    username: knp
    password: secret

logging:
  level:
    root: INFO
    kz.gov.kgd.isna: DEBUG
```

### 4.2 Properties

Эквивалент:
```properties
server.port=8080
server.compression.enabled=true

spring.application.name=isna-knp
spring.datasource.url=jdbc:postgresql://localhost:5432/knp
spring.datasource.username=knp
spring.datasource.password=secret

logging.level.root=INFO
logging.level.kz.gov.kgd.isna=DEBUG
```

Обычно **YAML** — удобнее для вложенных структур.

### 4.3 Форматы обоих

YAML поддерживает:
- Строки (`"foo"` или без кавычек).
- Числа (`123`, `1.5`).
- Boolean (`true`, `false`).
- Списки:
  ```yaml
  hosts:
    - server1
    - server2
  ```
- Multiline:
  ```yaml
  banner: |
    Line 1
    Line 2
  ```
- Флэт-запись:
  ```yaml
  spring.datasource.url: jdbc:...
  ```

---

## 5. Profiles

**Profile** — именованная группа конфигов, активная при определённых условиях.

### 5.1 Файлы

- `application.yml` — общий для всех.
- `application-dev.yml` — активен при `dev`.
- `application-prod.yml` — при `prod`.
- `application-test.yml` — обычно для тестов.

### 5.2 Активация

**A) Command-line**:
```
java -jar app.jar --spring.profiles.active=prod
```

**B) Env**:
```
SPRING_PROFILES_ACTIVE=prod
```

**C) System property**:
```
java -Dspring.profiles.active=prod -jar app.jar
```

**D) yml**:
```yaml
spring:
  profiles:
    active: prod
```

**E) Множество**:
```
--spring.profiles.active=prod,eu-west
```

Активны оба.

### 5.3 Профили в yml

Один файл, много профилей:
```yaml
# общие
spring:
  application:
    name: isna-knp

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:test

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://prod-db/knp
```

Разделитель `---`.

### 5.4 Profile-conditional beans

```java
@Configuration
@Profile("prod")
class ProdConfig {
    @Bean DataSource ds() { ... }
}

@Configuration
@Profile({"dev", "test"})
class DevConfig {
    @Bean DataSource ds() { ... }
}

@Configuration
@Profile("!prod")   // не prod
class NonProdConfig { }
```

### 5.5 Default profile

Если не указан профиль:
```yaml
spring:
  profiles:
    default: dev
```

---

## 6. Env variables — маппинг

Spring **relaxed binding**:

- Точки → underscores: `spring.datasource.url` → `SPRING_DATASOURCE_URL`.
- Kebab-case → UPPER_CASE: `some-property` → `SOME_PROPERTY`.
- CamelCase → UPPER_CASE: `someProperty` → `SOMEPROPERTY`.

Примеры:
```
SPRING_DATASOURCE_URL=jdbc:postgresql://...
SERVER_PORT=9090
LOGGING_LEVEL_ROOT=DEBUG
LOGGING_LEVEL_KZ_GOV_KGD_ISNA=TRACE
```

**Правило**: **env для секретов и environment-specific** (URLs, passwords). Не hardcode в yml.

### 6.1 Docker / K8s

Обычная схема:
```dockerfile
ENV SPRING_PROFILES_ACTIVE=prod
```

K8s:
```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: prod
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

---

## 7. Command-line arguments

```
java -jar app.jar --server.port=9090 --spring.profiles.active=prod
```

Двойные тире `--` перед свойством.

**Приоритет выше env / yml** — удобно для overrides.

---

## 8. Placeholders

Внутри yml — референсы на другие значения:

```yaml
app:
  name: isna-knp
  full-name: ${app.name}-integration
```

С default значением:
```yaml
spring:
  datasource:
    password: ${DB_PASSWORD:defaultpass}
```

Если `DB_PASSWORD` env есть → используется; нет → "defaultpass".

Nested:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:knp}
```

---

## 9. @Value

Простая injection свойства в поле/параметр:

```java
@Component
class MyBean {

    @Value("${app.name}")
    private String appName;

    @Value("${knp.max-batch-size:100}")
    private int batchSize;

    @Value("#{${knp.retries:3} * 2}")   // SpEL expression
    private int totalRetries;

    @Value("${feature.enabled}")
    private boolean featureEnabled;

    @Value("${servers}")   // "a,b,c"
    private List<String> servers;
}
```

Плюсы:
- Простой.

Минусы:
- **Не typed** — строка → надо парсить.
- **Не centralized** — разбросано по классам.
- **Нет validation**.

---

## 10. @ConfigurationProperties (лучше)

Типизированная **группа** свойств.

```java
@ConfigurationProperties(prefix = "knp")
@Component
public class KnpProperties {
    private int maxBatchSize = 100;
    private String outerSystemUrl;
    private Retry retry = new Retry();
    private Duration timeout = Duration.ofSeconds(30);

    public static class Retry {
        private int maxAttempts = 3;
        private Duration delay = Duration.ofSeconds(1);
        private double multiplier = 2.0;

        // getters/setters
    }

    // getters/setters
}
```

Конфиг:
```yaml
knp:
  max-batch-size: 200
  outer-system-url: https://outer.isna
  timeout: 60s
  retry:
    max-attempts: 5
    delay: 500ms
    multiplier: 1.5
```

Использование:
```java
@Autowired KnpProperties props;

int size = props.getMaxBatchSize();
Duration timeout = props.getTimeout();
int attempts = props.getRetry().getMaxAttempts();
```

### 10.1 Плюсы

- **Типизировано** — не строка, а `int`, `Duration`, `List`.
- **Централизованно** — все свойства в одном классе.
- **IDE support** — autocomplete в yml (при `spring-boot-configuration-processor`).
- **Validation** через JSR-380:
  ```java
  @ConfigurationProperties(prefix = "knp")
  @Validated
  public class KnpProperties {
      @NotBlank String outerSystemUrl;
      @Min(1) @Max(1000) int maxBatchSize;
      @NotNull Duration timeout;
  }
  ```
  При старте — падает если инвалид.

### 10.2 Relaxed binding

Все эти в yml маппятся на `maxBatchSize`:
```yaml
knp.maxBatchSize: 200
knp.max-batch-size: 200
knp.MAX_BATCH_SIZE: 200
```

Через env: `KNP_MAX_BATCH_SIZE=200`.

### 10.3 Duration / DataSize

Spring парсит специально:
```yaml
knp:
  timeout: 30s              # PT30S
  cache-max-size: 100MB
  retention: 7d
```

Типы: `Duration`, `DataSize`, `Period`.

### 10.4 @ConstructorBinding (Spring Boot 3+)

Для **immutable** properties:
```java
@ConfigurationProperties(prefix = "knp")
@ConstructorBinding
public record KnpProperties(
    @DefaultValue("100") int maxBatchSize,
    String outerSystemUrl,
    Retry retry
) {
    public record Retry(int maxAttempts, Duration delay) {}
}
```

Иммутабельно (record) — потокобезопасно.

С Boot 3 — `@ConstructorBinding` не нужен для record (автоматически).

### 10.5 @EnableConfigurationProperties

Регистрация без `@Component`:
```java
@Configuration
@EnableConfigurationProperties(KnpProperties.class)
class Config {}

@ConfigurationProperties(prefix = "knp")
public class KnpProperties { }   // без @Component
```

---

## 11. bootstrap.yml (Spring Cloud)

Читается **до** `application.yml`. Нужен для Spring Cloud:

```yaml
# bootstrap.yml
spring:
  application:
    name: isna-knp
  cloud:
    consul:
      host: consul.isna
      port: 8500
      config:
        enabled: true
```

Позволяет:
- Настроить Consul/Config Server до загрузки основного конфига.
- Основной yml читается из Consul KV.

С **Spring Cloud 2020+** bootstrap отключён по default → нужен `spring-cloud-starter-bootstrap` или `spring.config.import`.

### 11.1 spring.config.import (Boot 2.4+)

Замена bootstrap:
```yaml
spring:
  config:
    import:
      - "consul:"
      - "vault:"
      - "optional:file:./config/"
```

Более гибко.

---

## 12. External configuration

Помимо classpath — внешние файлы.

### 12.1 Приоритет

- `./config/application.yml` (рядом с jar).
- `./application.yml`.
- `classpath:/config/application.yml`.
- `classpath:/application.yml`.

Внешние **перекрывают** classpath — можно переопределить не пересобирая jar.

### 12.2 Custom location

```
java -jar app.jar --spring.config.location=/etc/myapp/config.yml
```

Или множественные:
```
--spring.config.location=classpath:/,file:./custom.yml
```

### 12.3 additional-location (лучше)

Добавляет к defaults, не заменяет:
```
--spring.config.additional-location=file:./custom.yml
```

---

## 13. Encryption секретов

Пароли в yml — плохо (git).

Варианты:
- **Env variables** — `${DB_PASSWORD}` из env.
- **Vault** (HashiCorp).
- **AWS Secrets Manager / GCP Secret Manager**.
- **Kubernetes Secret**.
- **Jasypt** — Spring Boot encryption:

```yaml
spring:
  datasource:
    password: ENC(encrypted-value)
```

```java
implementation 'com.github.ulisesbocchio:jasypt-spring-boot-starter:3.0.5'
```

Master password через env:
```
JASYPT_ENCRYPTOR_PASSWORD=master-key
```

Правило: **никогда plain secrets в git**.

---

## 14. Refresh конфига в рантайме

По умолчанию Spring Boot не обновляет конфиг в рантайме (нужен рестарт).

### 14.1 @RefreshScope + Actuator

С **Spring Cloud**:
```java
@RestController
@RefreshScope
class MyController {
    @Value("${feature.enabled}") boolean enabled;
}
```

POST `/actuator/refresh` → перечитывает конфиг → бины с `@RefreshScope` пересоздаются.

Изменения из Consul / Config Server подтягиваются автоматом (через bus refresh).

### 14.2 Кавет

**Не все** бины могут `@RefreshScope` — не singleton (пересоздаются). Осторожно с state.

---

## 15. Как узнать текущий конфиг

### 15.1 Actuator

```
GET /actuator/env
```

Показывает ВСЕ property sources + значения (с маскированием секретов).

```
GET /actuator/configprops
```

Показывает все `@ConfigurationProperties` бины с текущими значениями.

### 15.2 Программно

```java
@Autowired Environment env;

env.getActiveProfiles();
env.getProperty("spring.datasource.url");

// или через PropertySources
ConfigurableEnvironment cenv = (ConfigurableEnvironment) env;
cenv.getPropertySources().forEach(ps -> {
    log.info("PropertySource: {}", ps.getName());
});
```

---

## 16. Реализация под капотом

### 16.1 Environment

`ConfigurableEnvironment` — интерфейс с списком `PropertySources`.

При старте `SpringApplication`:
1. Создаёт `StandardServletEnvironment`.
2. Добавляет **system properties**, **env vars** как PropertySources.
3. **`ConfigDataEnvironmentPostProcessor`** ищет `application.yml/properties` в стандартных местах.
4. Для каждого activate profile — добавляет profile-specific.
5. Обрабатывает `spring.config.import` (Consul, Vault).
6. Результат — упорядоченный список PropertySources.

### 16.2 Property resolution

`env.getProperty("foo")`:
1. Идёт по PropertySources по порядку (высший приоритет первый).
2. Первое совпадение — возвращает.
3. Разрешает placeholders `${...}` (может рекурсивно).

### 16.3 @ConfigurationProperties binding

`ConfigurationPropertiesBinder` через reflection:
1. Читает bean class.
2. Ищет свойства по префиксу.
3. Конвертирует (String → int / Duration / etc).
4. Валидирует (@Validated).
5. Заполняет.

Всё на старте — если конфиг инвалид, приложение падает.

---

## 17. Пример полной конфигурации ИСНА

```yaml
# application.yml (общий)
spring:
  application:
    name: isna-knp-integration

server:
  port: 8080

management:
  endpoints:
    web.exposure.include: health,info,metrics,prometheus,configprops

---
# профиль dev
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:test
    username: sa
    password: ""
  jpa:
    hibernate.ddl-auto: create-drop
  cloud:
    consul:
      enabled: false

logging:
  level:
    kz.gov.kgd.isna: DEBUG

---
# профиль prod
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://${DB_HOST:db-knp}:5432/knp
    username: ${DB_USER:knp}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
  jpa:
    hibernate.ddl-auto: validate
  cloud:
    consul:
      host: ${CONSUL_HOST:consul.isna.internal}
      discovery:
        health-check-path: /actuator/health
        query-passing: true

logging:
  level:
    root: INFO
    kz.gov.kgd.isna: INFO

knp:
  max-batch-size: 200
  outer-system-url: ${OUTER_SYSTEM_URL:http://outer.isna:8080}
  timeout: 30s
  retry:
    max-attempts: 3
    delay: 1s
    multiplier: 2
```

Deploy:
- Local: `--spring.profiles.active=dev`.
- Prod: env `SPRING_PROFILES_ACTIVE=prod` + `DB_PASSWORD=secret` из K8s Secret.

---

## 18. Собесные вопросы

1. **Что такое PropertySource?** — Абстракция источника key=value; иерархия в Environment.
2. **Порядок приоритетов?** — Command-line > env > system > profile-yml > yml > defaults.
3. **Что такое profile?** — Именованная группа конфигов, активная по условию.
4. **Как активировать profile?** — `--spring.profiles.active`, `SPRING_PROFILES_ACTIVE`, `-Dspring.profiles.active`.
5. **Env → property маппинг?** — Relaxed binding: точки → underscores, upper case.
6. **Разница @Value и @ConfigurationProperties?** — Value — одна строка; ConfigurationProperties — типизированная группа + validation.
7. **@ConstructorBinding — что?** — Immutable properties через конструктор (record).
8. **bootstrap.yml vs application.yml?** — Bootstrap читается ДО application; для Spring Cloud (Consul).
9. **spring.config.import — что?** — Boot 2.4+ замена bootstrap для внешних источников.
10. **Как передать секрет в приложение?** — Env var (K8s Secret), Vault, Jasypt.
11. **Как посмотреть текущий конфиг?** — `/actuator/env`, `/actuator/configprops`.
12. **@RefreshScope — что?** — Бин пересоздаётся при POST `/actuator/refresh` (для конфига без рестарта).
13. **Duration тип в @ConfigurationProperties?** — Spring парсит `30s`, `5m`, `1h`.
14. **placeholder default value?** — `${VAR:default}`.
15. **Разница `spring.config.location` и `additional-location`?** — Location заменяет defaults; additional добавляет.

---

## Итог

- **PropertySource иерархия**: command-line > env > system > yml > defaults.
- **Profiles** для разных окружений.
- **@ConfigurationProperties** лучше @Value (типизация + validation).
- **Env vars** для секретов.
- **bootstrap.yml / spring.config.import** для Spring Cloud.
- **`/actuator/env`** для проверки в рантайме.
- **Relaxed binding** — env `SPRING_DATASOURCE_URL` = yml `spring.datasource.url`.

Следующий — `58-helm-helmsman.md`.
