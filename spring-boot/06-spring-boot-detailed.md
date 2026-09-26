# 06. Spring Boot: auto-configuration, конфигурация, MVC и Actuator

## Разница Spring Framework и Spring Boot

Spring Framework — гибкий конструктор. Даёт IoC контейнер, DI, AOP, поддержку MVC, JPA, security и десятки других модулей. Но всё нужно собирать вручную: подключать зависимости, объявлять beans, настраивать XML или Java-конфигурацию, разворачивать в application server. Простой проект с MVC и JPA до Spring Boot требовал сотен строк boilerplate configuration до первого работающего endpoint. Разработчик тратил дни на настройку инфраструктуры, а не на бизнес-логику.

Spring Boot — надстройка над Spring Framework, устраняющая эту сложность. Принцип "convention over configuration": если следовать разумным соглашениям, конфигурация не требуется. Хочешь web-приложение? Добавь `spring-boot-starter-web`, поставь `@SpringBootApplication` на класс, вызови `SpringApplication.run()` в main. Приложение работает: HTTP-сервер запущен, JSON serialization настроена, error handling есть.

Простейший Spring Boot проект целиком:

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

Это работающее HTTP-приложение. Пять строк. Всё остальное — auto-configuration.

Spring Boot не заменяет Spring Framework — он его надстраивает. Все стандартные концепции Spring (beans, DI, `@Transactional`, AOP) работают идентично. Boot добавляет автоматическую настройку, embedded servers, механизм внешней конфигурации через application.yml, actuator для мониторинга, plugin для сборки fat JAR. Всё это надстройка, устраняющая ручную работу.

В КНП все микросервисы — Spring Boot приложения. Знание Boot обязательно для работы с любым сервисом. Понимание что делает auto-configuration, как переопределить дефолтные beans, как читаются свойства из yaml — критично для регулярной разработки и диагностики.

В этом файле разберём Spring Boot глубоко. `@SpringBootApplication` и три аннотации внутри. Механизм auto-configuration с `@Conditional*` условиями. Starters как метапакеты зависимостей. Внешняя конфигурация через application.yml, profiles, порядок разрешения свойств. Actuator для operational endpoints и health checks. Spring MVC layer с DispatcherServlet. Spring Data JPA basics. Тестирование Spring Boot приложений на всех уровнях. Каждая тема — на уровне достаточном для реальной production работы.

## `@SpringBootApplication` изнутри

Одна аннотация, но композитная. Она разворачивается в три другие:

```java
@SpringBootConfiguration    // = @Configuration (класс конфигурации Spring)
@EnableAutoConfiguration     // магия auto-configuration
@ComponentScan               // сканирование пакета
public @interface SpringBootApplication {}
```

Каждая делает свою часть работы.

**`@SpringBootConfiguration`** — это по сути `@Configuration` с дополнительной семантикой "это Spring Boot приложение". Позволяет Spring Boot Test автоматически находить главный класс конфигурации при написании интеграционных тестов. Функционально идентично `@Configuration`.

**`@ComponentScan`** — сканирует пакет, где лежит класс с `@SpringBootApplication`, и все подпакеты. Находит все классы со стереотипами (`@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`, `@Configuration`), регистрирует их как beans.

Отсюда критически важное правило: **класс с `@SpringBootApplication` должен быть в корневом пакете приложения**. Если структура:

```
kz/gov/kgd/isna/knp/
├── KnpApplication.java              (@SpringBootApplication здесь)
├── controller/
│   └── FnoController.java
├── service/
│   └── FnoService.java
└── repository/
    └── FnoRepository.java
```

Всё в `kz.gov.kgd.isna.knp.*` найдётся component scan'ом. Классы в других пакетах (например `kz.gov.kgd.isna.commons.*`) — не найдутся, если не расширить сканирование явно:

```java
@ComponentScan(basePackages = {"kz.gov.kgd.isna.knp", "kz.gov.kgd.isna.commons"})
```

Это частая ошибка при рефакторинге — вынес класс в отдельный пакет, забыл добавить в component scan, приложение не видит beans.

**`@EnableAutoConfiguration`** — самая интересная из трёх. Именно она включает механизм auto-configuration, за который Spring Boot ценят. Разберём отдельно и подробно.

## Auto-configuration: главная фишка Spring Boot

Auto-configuration — механизм автоматической настройки beans на основе того, что есть в classpath и какие beans уже определены пользователем. Идея: Spring Boot знает, что если в classpath есть `spring-web`, значит нужен web-сервер и MVC-инфраструктура. Если есть `spring-boot-starter-data-jpa` и PostgreSQL JDBC driver, значит нужны Hibernate, HikariCP, EntityManagerFactory.

Автоматическая настройка происходит через сотни классов `*AutoConfiguration`, поставляемых с Spring Boot. Каждый такой класс — обычный `@Configuration` с `@Bean` методами, плюс `@Conditional*` аннотации, определяющие когда конфигурация должна активироваться.

Пример типичного auto-configuration класса (упрощённо):

```java
@Configuration
@ConditionalOnClass(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(name = "spring.datasource.url")
    public DataSource dataSource(DataSourceProperties props) {
        return DataSourceBuilder.create()
            .url(props.getUrl())
            .username(props.getUsername())
            .password(props.getPassword())
            .build();
    }
}
```

Логика читается: "если в classpath есть класс DataSource (значит подключена JDBC поддержка), и пользователь не определил свой bean DataSource, и указано свойство `spring.datasource.url` — создай стандартный DataSource".

Каждая `@Conditional*` — это фильтр. Все условия должны выполняться, чтобы конфигурация активировалась. Условия проверяются при старте приложения. Если хоть одно не выполнено, весь класс auto-configuration пропускается, никакие его `@Bean` методы не регистрируются.

Основные `@Conditional*` аннотации.

**`@ConditionalOnClass(SomeClass.class)`** — только если указанный класс есть в classpath. Используется для проверки "подключена ли определённая библиотека". Проверка через `Class.forName`, дешёвая и быстрая.

**`@ConditionalOnMissingClass("some.OldClass")`** — только если класса нет. Обратная логика.

**`@ConditionalOnBean(SomeBean.class)`** — только если такой bean уже зарегистрирован в контексте. Используется для добавления дополнительной конфигурации к существующей.

**`@ConditionalOnMissingBean(SomeBean.class)`** — только если такого bean нет. Классический паттерн: "создай стандартную реализацию, если пользователь не определил свою". Позволяет пользователю переопределить дефолт просто объявив свой bean.

**`@ConditionalOnProperty(name = "app.feature.enabled", havingValue = "true")`** — только если свойство установлено (и опционально имеет определённое значение). Используется для feature toggles: определённый auto-config активируется только когда пользователь включает функциональность через yml.

**`@ConditionalOnWebApplication`** и **`@ConditionalOnNotWebApplication`** — только для web/non-web приложений.

**`@Conditional(CustomCondition.class)`** — своя логика через реализацию интерфейса `Condition`.

Комбинация этих условий даёт мощный механизм: auto-config автоматически включает нужное, автоматически отключается когда не нужно, автоматически уступает пользовательскому переопределению.

## Как auto-configuration находит нужные классы

Spring Boot нужно знать, какие auto-configuration классы существуют, чтобы проверить их условия. Список хранится в специальных ресурсных файлах.

В Spring Boot 2.x — файл `META-INF/spring.factories`:

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
...
```

В Spring Boot 3.x перешли на новый формат — `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
...
```

Один класс на строку, без backslash-terminators.

При старте приложения Spring Boot сканирует все JAR-ы в classpath, читает эти файлы из каждого. Собирает полный список auto-configuration классов из Spring Boot, Spring Cloud, любых third-party starters. Проходит по каждому, проверяет условия. Активные регистрируются в контексте, их `@Bean` методы становятся источниками beans.

Именно это позволяет starters работать. Ты подключаешь `spring-cloud-starter-consul-discovery` в build.gradle — вместе с ним приезжает JAR с ConsulAutoConfiguration. Spring Boot видит его при старте, проверяет условия, регистрирует Consul-клиент. Никакой ручной настройки не нужно.

## Отладка auto-configuration

Часто нужно понять, почему определённая auto-configuration активировалась или не активировалась. Инструменты Spring Boot дают полную картину.

**Debug при старте** — активируется флагом `--debug` в командной строке или `debug: true` в application.yml. При старте выводится "AUTO-CONFIGURATION REPORT":

```
Positive matches (сработали):
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required class 'javax.sql.DataSource' (OnClassCondition)

Negative matches (не сработали):
   RedisAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'org.springframework.data.redis.core.RedisOperations' (OnClassCondition)

Exclusions:
   None

Unconditional classes:
   ...
```

Positive matches — активированные auto-configurations. Negative matches — те, что не активировались, с указанием причины (какое условие не выполнено).

**Actuator endpoint `/actuator/conditions`** — то же самое доступно через HTTP в runtime. Полезно когда приложение уже работает и нужно понять его конфигурацию.

**`@ConditionalOnBean(DataSource.class)` не срабатывает?** Часто причина в порядке evaluation условий. `@ConditionalOnBean` работает после регистрации всех BeanDefinitions, но не после их создания. Если DataSource определяется другим auto-config, может быть race condition. Обычно решается правильным использованием `@AutoConfigureAfter(DataSourceAutoConfiguration.class)`.

## Отключение auto-configuration

Иногда auto-configuration мешает — например, включает поведение несовместимое с текущими версиями библиотек, или дублирует ручную конфигурацию.

**Явно через аннотацию**:

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
public class MyApp { ... }
```

**Через properties** в application.yml:

```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
      - org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
```

Реальный кейс из КНП: `knp-form-hz5-actuator-cache-nosuchmethod` — при обновлении на Hazelcast 5 сломались две auto-configurations. `CacheMetricsAutoConfiguration` пытался вызвать `Cache.getNativeCache()`, который в Hazelcast 5 API уже не существует. `HazelcastHealthContributorAutoConfiguration` падал в health-check. Приложение не стартовало.

Решение — исключить проблемные auto-configurations:

```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.actuate.autoconfigure.metrics.cache.CacheMetricsAutoConfiguration
      - org.springframework.boot.actuate.autoconfigure.hazelcast.HazelcastHealthContributorAutoConfiguration
```

Изменение внесли в 4 модуля. На prod обошли через env variable `SPRING_AUTOCONFIGURE_EXCLUDE` (не в git), чтобы не задерживать hotfix релизными процедурами. Классический пример когда auto-configuration нужно отключить, потому что она автоматически включается, но некорректно работает в конкретной инсталляции.

## Starters: композитные зависимости

**Starter** — метапакет-зависимость, тянущий набор согласованных библиотек и auto-configurations для определённой функциональности. Ты указываешь один starter, получаешь всё нужное для этой задачи.

Ключевые Spring Boot starters, которые встречаются повсеместно.

**`spring-boot-starter`** — ядро. Включает основную функциональность Spring Boot: SpringApplication, auto-configuration, external configuration, logging (Logback). Другие starters транзитивно зависят от него.

**`spring-boot-starter-web`** — MVC-стек. Spring MVC, embedded Tomcat, Jackson для JSON, валидация, error handling. Всё для REST API одним подключением.

**`spring-boot-starter-webflux`** — Reactive стек. Spring WebFlux, Netty, Project Reactor. Альтернатива MVC для async workload'ов.

**`spring-boot-starter-data-jpa`** — JPA/Hibernate. Spring Data JPA, Hibernate, HikariCP как connection pool, transaction management.

**`spring-boot-starter-data-redis`** — Redis client. Обычно Lettuce как driver.

**`spring-boot-starter-security`** — Spring Security. Аутентификация, авторизация, CSRF, headers.

**`spring-boot-starter-actuator`** — operational endpoints. Health, metrics, env, info через HTTP.

**`spring-boot-starter-test`** — тестирование. JUnit 5, Mockito, AssertJ, Spring Test.

**`spring-boot-starter-validation`** — Bean Validation через Hibernate Validator.

**`spring-boot-starter-mail`** — JavaMail.

**`spring-boot-starter-quartz`** — Quartz Scheduler для сложных cron задач.

Плюс Spring Cloud starters для микросервисной архитектуры.

**`spring-cloud-starter-consul-discovery`** — интеграция с Consul.

**`spring-cloud-starter-openfeign`** — Feign декларативный HTTP client.

**`spring-cloud-starter-config`** — Spring Cloud Config Server.

Собственные starters — можно и нужно создавать для переиспользуемых компонентов. В КНП есть `isna-commons-*` и `isna-global-*` starters, обеспечивающие общую функциональность (аутентификация с Kalkan ECP, общие утилиты, стандартные health indicators). Подключаешь starter — получаешь всю функциональность плюс auto-configuration.

## Внешняя конфигурация

Одна из ключевых особенностей Spring Boot — гибкий механизм внешней конфигурации. Свойства читаются из множества источников, комбинируются с учётом приоритетов, доступны через единый Environment API.

**application.yml или application.properties** — основной файл конфигурации. Лежит в `src/main/resources`, попадает в JAR при сборке. YAML предпочтительнее properties для сложных иерархических структур.

Простой пример:

```yaml
server:
  port: 8080
  compression:
    enabled: true

spring:
  application:
    name: isnaKnpIntegration
  datasource:
    url: jdbc:postgresql://db-knp:5432/knp
    username: knp
    password: ${DB_PASSWORD}

logging:
  level:
    root: INFO
    kz.gov.kgd.isna: DEBUG
```

Плейсхолдеры `${...}` разрешаются из других источников свойств — переменных окружения, system properties, других файлов.

## Profiles

**Profiles** — механизм для разных окружений. Основной файл `application.yml` содержит общие свойства. Дополнительные `application-{profile}.yml` — специфичные для окружения.

Файлы:

- `application.yml` — общие для всех.
- `application-dev.yml` — активируется при profile `dev`.
- `application-prod.yml` — при `prod`.
- `application-local.yml` — для локальной разработки.

Активация через command line:

```bash
java -jar app.jar --spring.profiles.active=prod
```

Или через environment variable:

```bash
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

Свойства из profile-specific файла перезаписывают общие. Если в `application.yml` есть `database.url=localhost`, а в `application-prod.yml` — `database.url=prod-server`, то при active profile `prod` используется prod-server.

В Spring коде `@Profile` аннотация условно регистрирует beans:

```java
@Configuration
@Profile("prod")
public class ProdDbConfig {
    @Bean
    public DataSource dataSource() { /* prod version */ }
}

@Configuration
@Profile({"dev", "test"})
public class DevDbConfig {
    @Bean
    public DataSource dataSource() { /* dev/test version */ }
}
```

При активном profile `prod` регистрируется ProdDbConfig, DevDbConfig игнорируется.

В КНП стандартный набор profiles: `dev`, `test`, `preprod`, `prod`, `local`. Каждый со своим `application-{profile}.yml`. Local для разработчика (подключение к локальной БД, отключение внешних интеграций), dev для CI, test для integration тестов, preprod для staging, prod для production.

## bootstrap.yml — конфигурация Spring Cloud

Есть ещё один тип конфигурационного файла — `bootstrap.yml`. Читается **до** application.yml. Нужен для Spring Cloud integration.

Смысл. Когда используешь Spring Cloud Config Server или Consul KV для конфигурации, приложение должно знать как подключиться к этому config source перед чтением основной конфигурации. Настройки самого клиента должны быть доступны до загрузки application.yml.

```yaml
# bootstrap.yml
spring:
  application:
    name: isnaKnpIntegration
  cloud:
    consul:
      host: consul.isna.internal
      port: 8500
```

При старте:
1. Spring Boot читает `bootstrap.yml`.
2. Создаётся bootstrap ApplicationContext.
3. Подключается к Consul, читает конфигурацию оттуда.
4. Создаётся основной ApplicationContext с полной конфигурацией.

С Spring Cloud 2020.0+ bootstrap отключён по умолчанию — вся конфигурация должна быть в application.yml. Чтобы включить bootstrap обратно, нужно подключить `spring-cloud-starter-bootstrap`.

## Порядок разрешения свойств

Одна из самых важных практических тем — порядок приоритета источников свойств. Spring Boot читает свойства из множества источников, и одно и то же свойство может быть определено в нескольких. Приоритет определяет какое значение реально используется.

Порядок от наивысшего к низшему (более высокий перезаписывает более низкий):

1. **`spring.config.import`** — внешние источники (Vault, Consul KV) через явный import.
2. **Command-line аргументы** (`--server.port=9090`).
3. **JVM system properties** (`-Dserver.port=9090`).
4. **OS environment variables** (`SERVER_PORT=9090`).
5. **`application.yml` снаружи JAR** (рядом с JAR-файлом).
6. **profile-specific `application-{profile}.yml`** внутри JAR.
7. **`application.yml`** внутри JAR.
8. **`@PropertySource`** на классе конфигурации.
9. **`SpringApplication.setDefaultProperties`** — программные defaults.

Это ключ для правильной работы с окружениями. В Docker/K8s ты не редактируешь application.yml внутри image (сборка стабильна, immutable). Вместо этого передаёшь env variables и command-line arguments — они перезаписывают дефолты из application.yml.

**Правило маппинга env → property**: имя property записывается в UPPER_CASE с подчёркиваниями вместо точек. `spring.datasource.url` → `SPRING_DATASOURCE_URL`. `management.endpoint.health.probes.enabled` → `MANAGEMENT_ENDPOINT_HEALTH_PROBES_ENABLED`. Camel case превращается в snake case.

Это делает контейнерный деплой удобным. Одна и та же JAR (immutable) работает в разных окружениях с разными настройками, передаваемыми через env variables. В K8s ConfigMap и Secret проецируются как env variables в поды.

## Плейсхолдеры и дефолты

Синтаксис `${property.name:defaultValue}` — использует значение свойства, если не установлено — берёт дефолт.

```yaml
spring:
  datasource:
    password: ${DB_PASSWORD:defaultpass}   # env DB_PASSWORD, дефолт defaultpass
    url: ${DB_URL:jdbc:h2:mem:test}
```

Позволяет писать конфигурацию, работающую и в prod (env variables установлены), и в local dev (дефолты).

Плейсхолдеры могут ссылаться на другие свойства:

```yaml
app:
  base-url: http://${server.host:localhost}:${server.port:8080}
```

## @Value vs @ConfigurationProperties

Два способа читать свойства в код Spring beans.

**`@Value`** — простая инъекция одного свойства:

```java
@Value("${knp.max-batch-size:100}")
private int batchSize;

@Value("${knp.outer-system-url}")
private String outerUrl;
```

Работает для отдельных значений. Полезно для быстрого доступа к одному свойству. Ограничения: тип определяется по типу поля (легко ошибиться), нет валидации, ошибка при отсутствии свойства проявляется в runtime.

**`@ConfigurationProperties`** — типизированная группа свойств:

```java
@ConfigurationProperties(prefix = "knp")
@Component
@Validated
public class KnpProperties {
    @Min(1)
    private int maxBatchSize = 100;
    
    @NotBlank
    private String outerSystemUrl;
    
    private Retry retry = new Retry();
    
    public static class Retry {
        private int maxAttempts = 3;
        private Duration delay = Duration.ofSeconds(1);
        // getters/setters
    }
    // getters/setters
}
```

Все свойства с префиксом `knp.` мапятся на поля класса. `knp.max-batch-size` → `maxBatchSize`. Плюсы:

- Type safety — компилятор проверяет типы.
- IDE support — auto-completion в application.yml через spring-configuration-processor.
- Validation через JSR-380 (`@NotNull`, `@Min`, `@Pattern`).
- Иерархическая структура — сложные вложенные объекты естественны.
- Всё в одном месте — легко читать и модифицировать.

Использование:

```java
@Autowired
private KnpProperties props;

// props.getMaxBatchSize(), props.getOuterSystemUrl(), props.getRetry().getDelay()
```

Правило: для отдельных значений — `@Value`. Для групп связанных свойств — `@ConfigurationProperties`. В КНП всё серьёзная конфигурация — через `@ConfigurationProperties`.

## Actuator: operational endpoints

**Spring Boot Actuator** — модуль для production monitoring. Подключение:

```gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

По умолчанию доступны только `/actuator/health` и `/actuator/info`. Остальные endpoints нужно включить явно в application.yml:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,configprops,mappings,loggers,threaddump,heapdump,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true      # даёт /health/liveness и /health/readiness
```

Ключевые endpoints и что показывают.

**`/actuator/health`** — общий статус приложения (UP/DOWN). Со `show-details: always` показывает детали: статус каждого компонента (DB, disk space, Consul), причины failure если DOWN.

**`/actuator/health/liveness`** — только liveness часть. "Приложение живо, не нужно перезапускать". Проверяется K8s liveness probe.

**`/actuator/health/readiness`** — только readiness. "Приложение готово принимать трафик". Проверяется K8s readiness probe.

**`/actuator/info`** — build info, git commit (если настроено плагином git). Полезно для tracking какая версия задеплоена.

**`/actuator/metrics`** — метрики: memory usage, GC pauses, HTTP request duration, JDBC connections, thread pool state. По одной за раз через `/actuator/metrics/{name}`.

**`/actuator/env`** — все свойства и их источники. Полезно для diagnostic — понять откуда пришло значение.

**`/actuator/configprops`** — все `@ConfigurationProperties` beans с их значениями. Проверить как разрешились свойства.

**`/actuator/mappings`** — все URL routes зарегистрированные в MVC. Полезно для документирования и debugging.

**`/actuator/loggers`** — уровни логеров. **Можно менять в runtime через POST** — включить DEBUG для конкретного пакета без рестарта. Мощный инструмент для troubleshooting.

**`/actuator/threaddump`** — дамп всех потоков в JSON. Аналог `jstack`.

**`/actuator/heapdump`** — бинарный heap dump. Аналог `jmap`. Большой файл (сотни MB), скачивать осторожно.

**`/actuator/beans`** — все beans в контексте. Обширно, полезно для понимания что зарегистрировано.

**`/actuator/conditions`** — auto-configuration report.

**`/actuator/prometheus`** — метрики в формате Prometheus. Prometheus scrape'ит этот endpoint каждые 15 секунд.

Actuator в production — обязательно. Через него мониторинг, health checks для K8s и Consul, метрики для Prometheus/Grafana, troubleshooting.

## Health checks для K8s и Consul

Kubernetes probes проверяют здоровье пода. Actuator даёт endpoints, которые probes могут использовать.

В deployment yaml:

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
```

Liveness — "жив ли". Fail → K8s убивает контейнер и запускает новый. Не должен зависеть от внешних систем — если DB упала, liveness должен остаться UP, чтобы K8s не убивал под без причины.

Readiness — "готов ли принимать трафик". Fail → под остаётся, но исключается из Service — трафик не идёт. Может зависеть от внешних систем — если DB упала, приложение не готово работать корректно, лучше вывести из ротации.

Consul использует другой endpoint для health check:

```yaml
spring:
  cloud:
    consul:
      discovery:
        health-check-path: /actuator/health
        health-check-interval: 15s
```

Consul дёргает этот URL раз в 15 секунд. 200 OK — сервис `passing`, регистрируется в discovery. Другой код — `critical`, исключается.

## Custom health indicators

Стандартные health indicators покрывают DB, disk, ping. Иногда нужна кастомная проверка — доступность внешнего API, свой health check логики.

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    private final ExternalApi api;
    
    public ExternalApiHealthIndicator(ExternalApi api) {
        this.api = api;
    }
    
    @Override
    public Health health() {
        try {
            var status = api.ping();
            return Health.up().withDetail("api", status).build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

Автоматически регистрируется. Появляется в `/actuator/health` как отдельный компонент. Если down — общий статус тоже down.

## Micrometer: метрики

Actuator интегрируется с **Micrometer** — facade для метрик, поддерживающий разные backends (Prometheus, Datadog, Graphite, CloudWatch).

Для Prometheus подключается:

```gradle
implementation 'io.micrometer:micrometer-registry-prometheus'
```

Появляется endpoint `/actuator/prometheus` в формате, который понимает Prometheus. Auto-configuration регистрирует стандартные метрики: JVM memory, GC, thread pools, HTTP request duration (`http_server_requests_seconds`), JDBC connections.

Кастомные метрики через MeterRegistry:

```java
@Service
public class FnoService {
    private final Counter submitted;
    private final Timer processing;
    
    public FnoService(MeterRegistry registry) {
        this.submitted = registry.counter("fno.submitted");
        this.processing = registry.timer("fno.processing");
    }
    
    public void submit(Fno f) {
        processing.record(() -> {
            // бизнес-логика
            doSubmit(f);
            submitted.increment();
        });
    }
}
```

Метрики отображаются в Prometheus, визуализируются в Grafana.

## Logging

По умолчанию Spring Boot использует **Logback**. Конфигурация через application.yml или отдельный `logback-spring.xml`.

```yaml
logging:
  level:
    root: INFO
    kz.gov.kgd.isna: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type: TRACE   # значения bind-параметров
  file:
    name: /var/log/app.log
  pattern:
    console: "%d{HH:mm:ss.SSS} %-5level [%X{traceId:-}] %logger{36} - %msg%n"
```

Уровни: TRACE, DEBUG, INFO, WARN, ERROR. `root` — дефолтный уровень для всех логеров. Специфичные пакеты — свои уровни.

В коде используешь SLF4J:

```java
private static final Logger log = LoggerFactory.getLogger(FnoService.class);

log.info("Processed fno {}", fnoId);
log.error("Failed to process fno {}", fnoId, exception);
```

Или через Lombok `@Slf4j` — короче:

```java
@Slf4j
public class FnoService {
    public void doSomething() {
        log.info("Hello");
    }
}
```

**MDC (Mapped Diagnostic Context)** — thread-local контекст для structured logging. Полезно для correlation ID, trace ID:

```java
MDC.put("traceId", generateTraceId());
try {
    // обработка запроса, все логи получат traceId в контексте
} finally {
    MDC.clear();
}
```

Через pattern `%X{traceId}` выводится в логах.

В КНП логи льются в централизованный ELK stack (Elasticsearch + Kibana). Требования:
- Логи в JSON формате (logstash-encoder).
- Correlation ID через MDC для tracking запроса через микросервисы.
- Никакого `printStackTrace()` — идёт в stderr, ELK не парсит правильно.

Реальный кейс из КНП: `knp-fo-sync-notification-bugs` — `printStackTrace()` создал "ELK-слепую зону": ошибки были в stderr, но ELK индексировал только stdout, критические ошибки пропускались.

Ротация логов в K8s — не через logback, а через kubelet. `kubectl logs` показывает только текущий инстанс контейнера с последней ротации. История — в ELK (реальный кейс `knp-prod-historical-logs-elk`: логи старше суток только там).

## Spring MVC layer

`spring-boot-starter-web` подключает Spring MVC. Основной flow HTTP запроса.

Клиентский HTTP запрос приходит на embedded Tomcat. Tomcat парсит HTTP, создаёт HttpServletRequest, передаёт в **DispatcherServlet** — единственный Servlet Spring MVC, обрабатывающий все URL приложения.

DispatcherServlet работает через несколько компонентов.

**HandlerMapping** — маппит URL на handler methods. По умолчанию `RequestMappingHandlerMapping` — на основе `@RequestMapping` аннотаций в контроллерах.

**HandlerAdapter** — вызывает handler method. Для `@RequestMapping` методов — `RequestMappingHandlerAdapter`.

**HandlerInterceptor** — Spring MVC уровень, срабатывает до/после handler. Аутентификация, логирование, common processing.

**HttpMessageConverter** — сериализация. Автоматически конвертирует Java объекты в JSON/XML для response, и обратно для request body.

**ExceptionHandler** — обработка exceptions. Через `@ControllerAdvice`.

Пример controller:

```java
@RestController
@RequestMapping("/api/fno")
public class FnoController {
    private final FnoService svc;
    
    public FnoController(FnoService svc) {
        this.svc = svc;
    }
    
    @GetMapping("/{id}")
    public Fno get(@PathVariable Long id) {
        return svc.get(id);
    }
    
    @PostMapping
    public Fno create(@Valid @RequestBody FnoDto dto) {
        return svc.create(dto);
    }
    
    @GetMapping("/search")
    public List<Fno> search(@RequestParam String regNum) {
        return svc.search(regNum);
    }
    
    @PutMapping("/{id}")
    public void update(@PathVariable Long id, @RequestBody FnoDto dto) {
        svc.update(id, dto);
    }
    
    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) {
        svc.delete(id);
    }
}
```

`@RestController` = `@Controller` + `@ResponseBody` (на всех методах). Возвращаемые объекты автоматически сериализуются в JSON через Jackson.

## Global exception handling

Для централизованной обработки exceptions:

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorDto> notFound(EntityNotFoundException e) {
        return ResponseEntity.status(404).body(new ErrorDto(e.getMessage()));
    }
    
    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorDto> validation(ValidationException e) {
        return ResponseEntity.badRequest().body(new ErrorDto(e.getMessage()));
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorDto> validationFail(MethodArgumentNotValidException e) {
        var errors = e.getBindingResult().getAllErrors().stream()
            .map(err -> err.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return ResponseEntity.badRequest().body(new ErrorDto(errors));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorDto> generic(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity.status(500).body(new ErrorDto("Internal error"));
    }
}
```

Один класс для всей обработки. Порядок аннотаций важен — более специфичные должны быть выше generic Exception.

## Validation

Bean Validation через JSR-380. Аннотации на fields DTO:

```java
public class FnoDto {
    @NotBlank
    private String regNum;
    
    @Min(0)
    private long amount;
    
    @Email
    private String contactEmail;
    
    @NotNull
    @Valid          // рекурсивная валидация вложенного
    private AddressDto address;
}
```

Активация через `@Valid` в controller:

```java
@PostMapping
public Fno create(@Valid @RequestBody FnoDto dto) {
    return svc.create(dto);
}
```

При невалидных данных — MethodArgumentNotValidException, обрабатывается GlobalExceptionHandler.

Реальный кейс из КНП: `knp-subdivision-sync-validation-broke-prod` — `@NotBlank` был случайно поставлен на optional поле DTO. Все запросы, где это поле не было заполнено, стали failing с 400. Проблему выкатили в прод. Пришлось откатывать. Урок — валидация должна тестироваться на всех вариантах входных данных, включая пропущенные optional поля.

## Spring Data JPA

`spring-boot-starter-data-jpa` даёт Hibernate + Spring Data JPA магию.

Entity:

```java
@Entity
@Table(name = "fno")
public class Fno {
    @Id 
    @GeneratedValue
    private Long id;
    
    @Column(name = "reg_num", nullable = false, unique = true)
    private String regNum;
    
    @Enumerated(EnumType.STRING)
    private FnoStatus status;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "taxpayer_id")
    private Taxpayer taxpayer;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    // getters/setters
}
```

Repository — только интерфейс, реализация генерируется Spring Data:

```java
public interface FnoRepository extends JpaRepository<Fno, Long> {
    Optional<Fno> findByRegNum(String regNum);
    List<Fno> findAllByStatusAndCreatedAtAfter(FnoStatus status, LocalDateTime after);
    
    @Query("SELECT f FROM Fno f JOIN FETCH f.taxpayer WHERE f.status = :status")
    List<Fno> findByStatusWithTaxpayer(@Param("status") FnoStatus status);
    
    @Modifying
    @Query("UPDATE Fno f SET f.status = :status WHERE f.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") FnoStatus status);
}
```

Spring Data JPA парсит имена методов (`findByRegNum`), генерирует SQL. `@Query` для сложных запросов через JPQL. `@Modifying` для UPDATE/DELETE.

Классическая проблема — **N+1**. Читаешь список сущностей, для каждой обращаешься к lazy-loaded relation, для каждой генерируется отдельный SELECT:

```java
List<Fno> fnos = repo.findAll();
for (Fno f : fnos) {
    System.out.println(f.getTaxpayer().getName());  // каждый вызов = SELECT
}
```

100 fno → 1 SELECT для fno + 100 SELECT для taxpayers. Медленно. Решения: `JOIN FETCH` в JPQL, `@EntityGraph`, batch fetching через `@BatchSize`.

## Транзакции

`@Transactional` — декларативное управление транзакциями. Работает через AOP proxy:

```java
@Service
public class FnoService {
    private final FnoRepository repo;
    private final EventPublisher publisher;
    
    @Transactional
    public void submit(FnoDto dto) {
        Fno f = mapper.toEntity(dto);
        repo.save(f);
        publisher.publish(f);   // если бросит — весь метод rollback
    }
    
    @Transactional(readOnly = true)
    public List<Fno> list() {
        return repo.findAll();
    }
    
    @Transactional(propagation = REQUIRES_NEW, timeout = 30)
    public void logAudit(AuditEvent event) {
        // отдельная транзакция независимо от вызывающего
    }
}
```

Опции `@Transactional`:
- **propagation** — REQUIRED (default), REQUIRES_NEW, NESTED, SUPPORTS, MANDATORY, NEVER, NOT_SUPPORTED.
- **isolation** — READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE.
- **readOnly** — оптимизация (Hibernate не проверяет dirty state, driver может оптимизировать).
- **rollbackFor** — на какие exceptions rollback (default только RuntimeException + Error, checked не rollback'ит).
- **timeout** — секунды.

Основной gotcha — **self-invocation** (обсуждался в файле 05). Вызов `@Transactional` метода изнутри того же класса через `this.method()` не идёт через proxy, транзакция не открывается. Real production кейс из КНП: `knp-fo-sync-notification-bugs` — `@Transactional(REQUIRES_NEW)` был мёртв из-за self-invocation.

## Тестирование Spring Boot приложений

`spring-boot-starter-test` включает JUnit 5, Mockito, AssertJ, Hamcrest, Spring Test.

**Unit test** — без Spring, только Mockito:

```java
class FnoServiceTest {
    @Test
    void submit_savesFno() {
        FnoRepository repo = Mockito.mock(FnoRepository.class);
        FnoService svc = new FnoService(repo);
        
        svc.submit(new FnoDto("123"));
        
        verify(repo).save(any());
    }
}
```

Быстро, не поднимает Spring контекст. Правильный подход для чистой бизнес-логики.

**Slice test** — часть Spring контекста. `@WebMvcTest` для controller layer, `@DataJpaTest` для persistence layer, `@JsonTest` для JSON serialization и другие.

```java
@WebMvcTest(FnoController.class)
class FnoControllerTest {
    @Autowired 
    private MockMvc mvc;
    
    @MockBean
    private FnoService svc;
    
    @Test
    void getFno() throws Exception {
        when(svc.get(1L)).thenReturn(new Fno("123"));
        
        mvc.perform(get("/api/fno/1"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.regNum").value("123"));
    }
}
```

Поднимается только MVC infrastructure (DispatcherServlet, controllers, mappings, converters). Остальные beans mocked через `@MockBean`. Быстрее полного контекста, тестирует именно controller layer.

**Full integration test** — полный Spring контекст:

```java
@SpringBootTest
@AutoConfigureMockMvc
class FnoIntegrationTest {
    @Autowired 
    private MockMvc mvc;
    
    @Test
    void endToEnd() throws Exception {
        mvc.perform(post("/api/fno")
                .contentType(APPLICATION_JSON)
                .content("{...}"))
           .andExpect(status().isOk());
    }
}
```

Поднимается весь контекст, все beans реальные. Медленнее, но полнее. Для настоящей интеграции с DB — **Testcontainers**:

```java
@Testcontainers
@SpringBootTest
class FnoIntegrationTest {
    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:15");
    
    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }
    
    @Autowired
    private FnoRepository repo;
    
    @Test
    void savesAndReads() {
        Fno f = new Fno("123");
        repo.save(f);
        
        assertThat(repo.findByRegNum("123")).isPresent();
    }
}
```

Testcontainers поднимает реальный PostgreSQL в Docker для теста. Спринг подключается к нему. Полная интеграция без моков БД.

Стратегия — пирамида тестов. Много unit-тестов (быстро). Средне slice-тестов. Мало full integration-тестов (медленно, для сквозной проверки).

## Пример полной application.yml из КНП

Реальная конфигурация типичного сервиса КНП:

```yaml
server:
  port: 8080
  servlet:
    context-path: /

spring:
  application:
    name: isnaKnpIntegration
  
  datasource:
    url: jdbc:postgresql://db-knp:5432/knp
    username: ${DB_USER:knp}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      transaction-isolation: TRANSACTION_READ_COMMITTED
  
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        jdbc:
          time_zone: UTC
        default_batch_fetch_size: 20
        query:
          preferred_instant: TIMESTAMP
  
  cloud:
    consul:
      host: ${CONSUL_HOST:consul.isna.internal}
      port: 8500
      discovery:
        service-name: ${spring.application.name}
        health-check-path: /actuator/health
        health-check-interval: 15s
        prefer-ip-address: true
        query-passing: true
  
  rabbitmq:
    host: ${RABBITMQ_HOST:rabbit.isna.internal}
    port: 5672
    username: knp
    password: ${RABBITMQ_PASSWORD}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true

logging:
  level:
    root: INFO
    kz.gov.kgd.isna: INFO
  pattern:
    console: "%d{HH:mm:ss.SSS} %-5level [%X{traceId:-}] [%thread] %logger{36} - %msg%n"

knp:
  max-batch-size: 100
  outer-system-url: ${OUTER_SYSTEM_URL:http://outer.isna:8080}
  retry:
    max-attempts: 3
    delay: 1s
```

Компоненты: server (порт), spring.datasource (JDBC + HikariCP), spring.jpa (Hibernate настройки), spring.cloud.consul (service discovery), spring.rabbitmq (messaging), management (Actuator), logging, custom `knp.*` properties для `@ConfigurationProperties`.

Значения с `${...}` — плейсхолдеры для env variables. В prod они устанавливаются через K8s ConfigMap/Secret. В local dev — дефолтные значения работают.

## Заключение

Spring Boot — надстройка над Spring Framework, устраняющая ручную конфигурацию через convention over configuration. `@SpringBootApplication` — три аннотации в одной: `@SpringBootConfiguration`, `@EnableAutoConfiguration`, `@ComponentScan`.

Auto-configuration — механизм автоматической настройки beans на основе classpath и conditional evaluation. Сотни `*AutoConfiguration` классов, каждый с `@Conditional*` фильтрами. Активируются только когда нужно, автоматически уступают пользовательскому переопределению. Отладка через `--debug`, `/actuator/conditions`.

Starters — метапакеты с согласованными зависимостями и auto-configurations. Один starter в build.gradle — вся функциональность. Стандартные для web, JPA, security, actuator, cloud, тестов. Собственные для переиспользуемых компонентов.

External configuration через application.yml, profiles для окружений, порядок разрешения свойств (command-line → env → system → yml). Плейсхолдеры для интеграции с env variables. `@Value` для одного свойства, `@ConfigurationProperties` для типизированных групп.

Actuator даёт operational endpoints: health для probes, metrics для мониторинга, env для diagnostic, loggers для runtime изменения уровней. Micrometer как facade для метрик, интеграция с Prometheus.

Spring MVC layer через `spring-boot-starter-web`. DispatcherServlet, `@RestController`, `@RequestMapping`, `@RequestBody`. Global exception handling через `@ControllerAdvice`. Validation через Bean Validation.

Spring Data JPA для persistence. Repository интерфейсы, Spring генерирует реализацию. Классическая N+1 проблема — знать и лечить через JOIN FETCH или @EntityGraph. Транзакции через `@Transactional`, помнить self-invocation.

Тестирование на всех уровнях: unit (без Spring), slice (`@WebMvcTest`, `@DataJpaTest`), full integration (`@SpringBootTest` с Testcontainers).

Для КНП контекста практично: понимание auto-configuration для диагностики стартовых проблем, знание Actuator для monitoring и troubleshooting, правильная работа с profiles (dev/test/preprod/prod/local), знание Spring MVC для controller разработки. Дальше — практика на существующих сервисах и постепенно углубляющаяся диагностика реальных проблем.
