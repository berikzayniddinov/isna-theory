# 57. Spring Boot конфигурация: PropertySource, profiles, ConfigurationProperties, secrets

## Зачем нужна глубокая конфигурация

Разработчик который недавно в Spring Boot обычно использует application.yml как «place где положить настройки». Знает про profiles — dev, prod. Считает достаточным. Реальность production приносит complications. Как передать secret database password? Не хочется коммитить в git. Как override configuration для specific environment без rebuild приложения? Как валидировать что required properties provided на startup? Как читать configuration из external systems (Consul, Vault) а не только файлов?

Разница между разработчиком «использующим yml» и «понимающим Spring configuration» проявляется в operational flexibility. Первый застревает при deployment scenarios — переопределение property требует rebuild, secrets вваливают plain text в yml, missing properties detected только когда endpoint hit. Второй знает PropertySource hierarchy — command-line args override env vars override yml. Знает что @ConfigurationProperties provides typed configuration с validation vs @Value string extraction. Знает что secrets injected через env vars from K8s Secrets or Vault vault. Знает spring.config.import для reading configuration from Consul или other external sources.

В этом файле разберём Spring configuration comprehensively. Fundamental problem конфигурации для разных environments. PropertySource как abstraction. Full hierarchy priorities. application.yml plus properties files. Profiles mechanism. Environment variables mapping (relaxed binding). Command-line arguments. Placeholders. @Value simple injection. @ConfigurationProperties typed groups. bootstrap.yml legacy. spring.config.import modern approach. External configuration. Secret encryption strategies. Runtime configuration refresh. Introspection through Actuator. Реализация под капотом.

## Fundamental problem конфигурации

Приложение должно работать по-разному в. Local dev — H2, HTTP :8080, DEBUG logs. CI/test — Testcontainers PostgreSQL. Preprod — реальный PG но test data. Prod — реальный PG, real users.

Код одинаковый, отличается только configuration. This principle 12-factor apps — configuration в environment, code portable across deployments.

Spring Boot даёт мощный механизм — PropertySource как abstraction иерархия источников конфига. Multiple sources combined в predictable priority order. Application code queries Environment abstraction без knowing exact source.

## PropertySource

Абстракция «источник key=value». Каждый PropertySource — либо файл, либо env, либо ещё что-то (JNDI, ServletContext parameters).

Приложение видит их через Environment bean:
```java
@Autowired Environment env;

String url = env.getProperty("spring.datasource.url");
```

Spring перебирает source'ы по приоритету — возвращает первое найденное значение. First match wins. Higher priority sources override lower.

## Полный порядок приоритетов

От высшего приоритета к низшему:

1. DevTools global settings (~/.spring-boot-devtools.properties).
2. @TestPropertySource аннотации в тестах.
3. SpringApplication.setDefaultProperties (programmatic).
4. @SpringBootTest properties.
5. Command-line arguments (--server.port=9090).
6. JSON в SPRING_APPLICATION_JSON env var.
7. ServletConfig init parameters.
8. ServletContext init parameters.
9. JNDI attributes (java:comp/env).
10. Java System properties (-Dserver.port=9090).
11. OS environment variables (SERVER_PORT=9090).
12. Профиль-специфичные properties снаружи jar (./application-prod.yml).
13. Профиль-специфичные properties внутри jar (classpath:application-prod.yml).
14. Общие properties снаружи jar (./application.yml).
15. Общие properties внутри jar (classpath:application.yml).
16. @PropertySource на @Configuration.
17. Default properties (SpringApplication.setDefaultProperties).

Чем выше — тем важнее (перекрывает нижнее).

На практике обычно в порядке важности сверху. Command-line (--foo=bar). Env vars (FOO=bar). JVM props (-Dfoo=bar). application-{profile}.yml (external / classpath). application.yml (external / classpath). Defaults.

Overriding cascade. Deployment-time changes через command-line или env. Environment-specific через profile files. Default в code через yml. Predictable behavior.

## application.yml и application.properties

Основные файлы конфига в src/main/resources.

YAML format popular из-за vложенных structures:
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

Properties equivalent:
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

Обычно YAML — удобнее для nested structures. Properties может быть preferred для simple flat configuration.

YAML supports. Строки ("foo" или без кавычек). Числа (123, 1.5). Boolean (true, false). Списки:
```yaml
hosts:
  - server1
  - server2
```

Multiline strings:
```yaml
banner: |
  Line 1
  Line 2
```

Flat notation:
```yaml
spring.datasource.url: jdbc:...
```

## Profiles

Profile — именованная группа конфигов, активная при определённых условиях. Central mechanism для environment-specific configuration.

Files. application.yml — общий для всех. application-dev.yml — активен при profile dev. application-prod.yml — при prod. application-test.yml — обычно для тестов.

Активация несколькими способами.

Command-line:
```
java -jar app.jar --spring.profiles.active=prod
```

Env:
```
SPRING_PROFILES_ACTIVE=prod
```

System property:
```
java -Dspring.profiles.active=prod -jar app.jar
```

Или в yml:
```yaml
spring:
  profiles:
    active: prod
```

Множество profiles:
```
--spring.profiles.active=prod,eu-west
```

Активны оба. Multiple simultaneous profiles combine configurations.

Профили в одном yml через разделитель ---:
```yaml
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

Один файл, много профилей. Alternative to separate files. Preferable когда файлы small или sharing common structure.

Profile-conditional beans через @Profile:
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

Beans registered только when matching profile active. Enables completely different bean configurations per environment.

Default profile если не указан профиль:
```yaml
spring:
  profiles:
    default: dev
```

Applied когда no profile active. Прevents ambiguous default behavior.

## Environment variables

Spring имеет relaxed binding для env vars mapping.

Точки → underscores. spring.datasource.url → SPRING_DATASOURCE_URL.

Kebab-case → UPPER_CASE. some-property → SOME_PROPERTY.

CamelCase → UPPER_CASE. someProperty → SOMEPROPERTY.

Примеры:
```
SPRING_DATASOURCE_URL=jdbc:postgresql://...
SERVER_PORT=9090
LOGGING_LEVEL_ROOT=DEBUG
LOGGING_LEVEL_KZ_GOV_KGD_ISNA=TRACE
```

Правило. Env для секретов и environment-specific (URLs, passwords). Не hardcode в yml. Application code stays same across environments, configuration через env varies.

Docker / K8s conventions:
```dockerfile
ENV SPRING_PROFILES_ACTIVE=prod
```

K8s Secret injection:
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

Secrets stored в K8s Secret resource. Environment variables reference them. Never in image или yml plaintext.

## Command-line arguments

```
java -jar app.jar --server.port=9090 --spring.profiles.active=prod
```

Двойные тире перед свойством. Convention для Spring Boot arguments.

Приоритет выше env / yml — удобно для overrides. Ad hoc reconfiguration без changing environment or rebuilding.

## Placeholders

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

Если DB_PASSWORD env есть — используется. Нет — defaultpass. Fallback mechanism для missing configuration.

Nested placeholders:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:knp}
```

Composition из multiple placeholders. Powerful для flexible configuration.

## @Value simple injection

Простая injection свойства в поле или параметр:
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

Плюсы. Простой. Straightforward one-property injection.

Минусы. Не typed — конверсия automatic но limited. Не centralized — свойства разбросаны по классам. Нет validation. Difficult refactoring — property name changes require finding all @Value usages.

Для нескольких properties preferred @ConfigurationProperties.

## @ConfigurationProperties (лучше)

Типизированная группа свойств:
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

Configuration:
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

Плюсы. Типизировано — не строка, а int, Duration, List. Централизованно — все свойства в одном классе. IDE support — autocomplete в yml (при spring-boot-configuration-processor). Validation через JSR-380:
```java
@ConfigurationProperties(prefix = "knp")
@Validated
public class KnpProperties {
    @NotBlank String outerSystemUrl;
    @Min(1) @Max(1000) int maxBatchSize;
    @NotNull Duration timeout;
}
```

При старте — падает если инвалид. Fail-fast обнаружение configuration errors.

Relaxed binding. Все эти в yml маппятся на maxBatchSize:
```yaml
knp.maxBatchSize: 200
knp.max-batch-size: 200
knp.MAX_BATCH_SIZE: 200
```

Через env — KNP_MAX_BATCH_SIZE=200. Multiple naming conventions accepted.

Duration / DataSize types. Spring парсит специально:
```yaml
knp:
  timeout: 30s              # PT30S
  cache-max-size: 100MB
  retention: 7d
```

Типы. Duration, DataSize, Period. Human-friendly notation converted к Java types automatically.

@ConstructorBinding (Spring Boot 3+) для immutable properties:
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

Immutable (record) — потокобезопасно. С Boot 3 @ConstructorBinding не нужен для record (automatic).

@EnableConfigurationProperties регистрация без @Component:
```java
@Configuration
@EnableConfigurationProperties(KnpProperties.class)
class Config {}

@ConfigurationProperties(prefix = "knp")
public class KnpProperties { }   // без @Component
```

Alternative registration mechanism.

## bootstrap.yml (Spring Cloud legacy)

Читается до application.yml. Нужен для Spring Cloud:
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

Позволяет. Настроить Consul/Config Server до загрузки основного конфига. Основной yml читается из Consul KV.

С Spring Cloud 2020+ bootstrap отключён по default. Нужен spring-cloud-starter-bootstrap или spring.config.import (modern approach).

## spring.config.import (Boot 2.4+)

Замена bootstrap для reading external configuration:
```yaml
spring:
  config:
    import:
      - "consul:"
      - "vault:"
      - "optional:file:./config/"
```

Более гибко. Standard Spring Boot mechanism не Spring Cloud specific.

Prefix «optional:» — если source unavailable, не fail. Fallback graceful.

Modern replacement для bootstrap. Simpler configuration model.

## External configuration

Помимо classpath — внешние файлы.

Приоритет для external config. ./config/application.yml (рядом с jar). ./application.yml. classpath:/config/application.yml. classpath:/application.yml.

Внешние перекрывают classpath. Можно переопределить не пересобирая jar. Deployment flexibility.

Custom location:
```
java -jar app.jar --spring.config.location=/etc/myapp/config.yml
```

Или множественные:
```
--spring.config.location=classpath:/,file:./custom.yml
```

additional-location добавляет к defaults не заменяет:
```
--spring.config.additional-location=file:./custom.yml
```

Preferred variant — augment defaults вместо replacement. Avoid accidentally missing configuration.

## Encryption секретов

Пароли в yml — плохо (git commits). Multiple mitigation strategies.

Env variables. ${DB_PASSWORD} из env. Most common approach.

Vault (HashiCorp). Dedicated secrets management. Runtime retrieval. Rotation supported.

AWS Secrets Manager / GCP Secret Manager. Cloud provider solutions. Integrated с IAM.

Kubernetes Secret. K8s native. Base64 encoded (not encrypted at rest by default but options exist).

Jasypt — Spring Boot encryption:
```yaml
spring:
  datasource:
    password: ENC(encrypted-value)
```

```gradle
implementation 'com.github.ulisesbocchio:jasypt-spring-boot-starter:3.0.5'
```

Master password через env:
```
JASYPT_ENCRYPTOR_PASSWORD=master-key
```

Правило. Никогда plain secrets в git. Multiple layer defense — encryption plus access control plus rotation.

## Refresh конфига в рантайме

По умолчанию Spring Boot не обновляет конфиг в рантайме (нужен рестарт).

@RefreshScope plus Actuator с Spring Cloud:
```java
@RestController
@RefreshScope
class MyController {
    @Value("${feature.enabled}") boolean enabled;
}
```

POST /actuator/refresh — перечитывает конфиг. Бины с @RefreshScope пересоздаются. Runtime configuration updates без restart.

Изменения из Consul / Config Server подтягиваются автоматом через bus refresh.

Кавет. Не все бины могут @RefreshScope. Не singleton — пересоздаются. Осторожно с state. State-heavy beans lose состояние at refresh.

## Как узнать текущий конфиг

Actuator endpoints:
```
GET /actuator/env
```

Показывает ВСЕ property sources plus значения (с маскированием секретов).

```
GET /actuator/configprops
```

Показывает все @ConfigurationProperties бины с текущими значениями. Type-safe view of configuration.

Программно:
```java
@Autowired Environment env;

env.getActiveProfiles();
env.getProperty("spring.datasource.url");

ConfigurableEnvironment cenv = (ConfigurableEnvironment) env;
cenv.getPropertySources().forEach(ps -> {
    log.info("PropertySource: {}", ps.getName());
});
```

Runtime introspection. Debugging misconfigured deployments.

## Реализация под капотом

Environment interface ConfigurableEnvironment имеет список PropertySources.

При старте SpringApplication. Создаёт StandardServletEnvironment. Добавляет system properties, env vars как PropertySources. ConfigDataEnvironmentPostProcessor ищет application.yml/properties в стандартных местах. Для каждого active profile — добавляет profile-specific. Обрабатывает spring.config.import (Consul, Vault). Результат — упорядоченный список PropertySources.

Property resolution через env.getProperty("foo"). Идёт по PropertySources по порядку (высший приоритет первый). Первое совпадение — возвращает. Разрешает placeholders ${...} рекурсивно.

@ConfigurationProperties binding. ConfigurationPropertiesBinder через reflection. Читает bean class. Ищет свойства по префиксу. Конвертирует (String → int / Duration / etc). Валидирует (@Validated). Заполняет.

Всё на старте — если конфиг инвалид, приложение падает. Fail-fast principle.

## Пример полной конфигурации ИСНА

Real-world example:
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

Deployment. Local — --spring.profiles.active=dev. Prod — env SPRING_PROFILES_ACTIVE=prod plus DB_PASSWORD=secret из K8s Secret.

Structure demonstrates. Common configuration в default section. Profile-specific overrides. Placeholders для env-specific values. Business configuration отдельно (knp namespace).

## Итоги

PropertySource как abstraction. Иерархия priorities — command-line > env > system > profile-yml > yml > defaults.

Profiles для разных окружений. @Profile для beans. Multiple simultaneous profiles.

Environment variables с relaxed binding — точки в underscores, upper case.

Placeholders ${var:default} для references plus defaults.

@ConfigurationProperties preferred over @Value. Типизированные группы. Validation через JSR-380. IDE support через spring-boot-configuration-processor.

Duration, DataSize types automatically parsed. Human-friendly notation.

@ConstructorBinding для immutable properties. Records в Spring Boot 3+ automatic.

bootstrap.yml legacy. spring.config.import modern approach для external sources (Consul, Vault).

External configuration через ./config/ folder или explicit location.

Secrets management критично. Env vars, Vault, K8s Secret, Jasypt для encryption at rest.

Runtime refresh через @RefreshScope plus Actuator refresh endpoint. Careful with stateful beans.

Introspection через /actuator/env и /actuator/configprops. Runtime debugging capability.

Дальше — Helm plus Helmsman для K8s deployment automation.
