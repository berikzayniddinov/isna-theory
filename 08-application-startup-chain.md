# 08. Полная цепочка запуска Spring Boot приложения

## Зачем разбираться в startup

Запустил `java -jar app.jar` — через 20-30 секунд приложение готово принимать HTTP-запросы. Что происходит в эти секунды? Для среднего разработчика — магия. Для senior — понятная последовательность из десятков шагов, каждый из которых может сломаться.

Понимание этой цепочки нужно для реальной работы в production. Приложение не стартует с BeanCreationException — на каком этапе? Приложение долго стартует — где потерялось время? Приложение стартовало, но K8s liveness probe fails первые 30 секунд — почему? Contact с DB не устанавливается — где проблема, в connection pool init или в Hibernate SessionFactory init? Все эти вопросы упираются в понимание конкретных шагов startup.

Плюс это база для правильной работы с lifecycle hooks. Куда воткнуть warmup код — `@PostConstruct`, `CommandLineRunner`, `ApplicationReadyEvent`? Разница есть. Как правильно настроить startup, liveness, readiness probes в K8s? Требует знания того, когда именно приложение "готово".

В этом файле пройдём весь путь от `java -jar` до момента "приложение принимает трафик" — детально. JVM initialization. JarLauncher и custom classloader. SpringApplication.run() и его фазы. Refresh контекста и создание beans. Старт embedded server. Регистрация в Consul. Публикация events. K8s probes. Всё в одной последовательной картине.

## Обзорная картина

Полная цепочка от команды до готовности:

```
java -jar isna-knp-integration.jar --spring.profiles.active=prod
  │
  ▼ JVM init: heap, GC, classloader
  │
  ▼ Read MANIFEST.MF → Main-Class = JarLauncher
  │
  ▼ JarLauncher.main:
  │   - создаёт LaunchedURLClassLoader
  │   - читает Start-Class = KnpApplication
  │   - через reflection вызывает KnpApplication.main
  │
  ▼ SpringApplication.run(KnpApplication.class, args):
  │   - создаёт Environment (yml, profiles, env, cli)
  │   - определяет тип приложения (SERVLET/REACTIVE/NONE)
  │   - создаёт ApplicationContext
  │   - refresh() ← основная фаза
  │
  ▼ ApplicationContext.refresh():
  │   - component scan
  │   - auto-configuration (условная регистрация beans)
  │   - создание singleton beans (bean lifecycle + прокси)
  │   - старт embedded server (Tomcat)
  │   - публикация ContextRefreshedEvent
  │
  ▼ Post-refresh:
  │   - регистрация в Consul (если starter в classpath)
  │   - выполнение CommandLineRunner / ApplicationRunner
  │   - публикация ApplicationReadyEvent
  │
  ▼ Приложение готово принимать HTTP-запросы
  │
  ▼ K8s: readinessProbe /actuator/health/readiness → 200
  │   → под добавляется в Service, начинает получать трафик
  │
  ▼ Первый HTTP-запрос
```

Разберём каждый этап детально.

## JVM initialization

Команда `java -jar isna-knp-integration.jar` запускает Java Virtual Machine. JVM — это отдельный процесс операционной системы, реализующий спецификацию JVM.

При запуске JVM:

Инициализируется среда исполнения. Загружается libjvm.so на Linux (или jvm.dll на Windows). Настраиваются области памяти: heap с указанными `-Xms` и `-Xmx` размерами, metaspace, code cache. Инициализируется garbage collector согласно указанному алгоритму (по умолчанию G1 с Java 9+).

Загружаются **Bootstrap classes** через Bootstrap ClassLoader — фундаментальные классы: `java.lang.*`, `java.util.*`, `java.io.*`. Это native код внутри JVM, они всегда доступны.

Затем JVM нужно найти класс с `main` методом. Для команды `java -jar` это делается через манифест JAR-файла. JVM открывает JAR как ZIP-архив, читает `META-INF/MANIFEST.MF`, ищет ключ `Main-Class`.

Аргументы командной строки делятся на две части. Всё до `-jar` (или `-cp`) — **JVM options**: `-Xmx1024m`, `-XX:+UseG1GC`, `-Dspring.profiles.active=prod`. Их обрабатывает JVM. Всё после JAR-файла — **аргументы приложения**, передадутся в метод `main` как параметр `String[] args`.

Типичные JVM options для Spring Boot приложения в Docker:

```
JAVA_OPTS="-Xms512m -Xmx1024m -XX:+UseG1GC -XX:MaxRAMPercentage=75 \
           -Dfile.encoding=UTF-8 -Duser.timezone=Asia/Almaty \
           -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp"
```

`-Xms=-Xmx` — фиксированный размер heap, избегает pause при resize. `MaxRAMPercentage=75` — использовать 75% доступной container memory (cgroup limit). `+UseG1GC` — G1 collector. `-Duser.timezone` — важно, потому что по умолчанию JVM может взять UTC вместо ожидаемой Asia/Almaty.

## MANIFEST и JarLauncher

JVM читает `META-INF/MANIFEST.MF` из JAR:

```
Manifest-Version: 1.0
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: kz.gov.kgd.isna.knp.KnpApplication
Spring-Boot-Version: 2.2.4.RELEASE
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
```

Ключевое: `Main-Class` — это **не** твой класс. Это `JarLauncher` из Spring Boot loader. JVM вызывает `JarLauncher.main(args)`.

Почему такая косвенность? Потому что стандартный JVM classloader не умеет читать JAR внутри JAR. У Spring Boot fat JAR структура — твои классы в `BOOT-INF/classes/`, зависимости в `BOOT-INF/lib/` целыми JAR-файлами. Обычный classloader не видит classes внутри вложенных JAR. Нужен специальный.

JarLauncher выполняет эту специальную инициализацию. Открывает свой JAR как ZIP-архив. Строит список URL: сам JAR (для чтения `BOOT-INF/classes/`) плюс каждый `BOOT-INF/lib/*.jar` как виртуальный URL с двойным `!/` (`jar:file:/path/to/myapp.jar!/BOOT-INF/lib/spring-web.jar!/`).

Создаёт `LaunchedURLClassLoader` с этими URL. Этот classloader — расширение `URLClassLoader`, регистрирующее custom `URLStreamHandler` для протокола `jar:`, умеющее читать JAR-в-JAR.

Устанавливает `Thread.currentThread().setContextClassLoader(launchedLoader)` — теперь текущий поток видит все classes из вложенных JAR-ов.

Читает `Start-Class` из манифеста — это `KnpApplication`. Через reflection загружает его: `Class.forName("kz.gov.kgd.isna.knp.KnpApplication", true, launchedLoader)`. Находит метод `main(String[])`, вызывает его с переданными аргументами.

С этого момента управление в твоём коде. Всё что дальше загружается — через LaunchedURLClassLoader. Spring, Jackson, PostgreSQL driver — все находятся в `BOOT-INF/lib/` и доступны.

## KnpApplication.main и SpringApplication.run

Твой главный класс:

```java
@SpringBootApplication
public class KnpApplication {
    public static void main(String[] args) {
        SpringApplication.run(KnpApplication.class, args);
    }
}
```

`SpringApplication.run` — статический метод. Эквивалент:

```java
new SpringApplication(KnpApplication.class).run(args);
```

Внутри `SpringApplication` конструктор делает подготовительную работу. Определяет тип приложения по classpath: если есть `spring-webmvc` — SERVLET, если `spring-webflux` — REACTIVE, ни того ни другого — NONE (обычно CLI утилита). Регистрирует `ApplicationContextInitializer`-ы и `ApplicationListener`-ы, зарегистрированные в `spring.factories` файлах всех JAR-ов classpath.

Метод `run(args)` выполняет фазы старта.

**Печать banner**. ASCII баннер из `banner.txt` в classpath или дефолтный Spring логотип. Не самая важная часть, но привычная.

**Создание Environment**. Это критический этап. `ConfigurableEnvironment` — интерфейс, вбирающий все property sources. Порядок загрузки определяет приоритеты (обсуждали в файле 06). Читаются `application.yml`, `application-{profile}.yml`, разрешаются плейсхолдеры, применяются active profiles. При завершении Environment содержит всю конфигурацию.

**Определение типа приложения**. Ранее уже определено, здесь фиксируется. Соответственно выбирается тип ApplicationContext:
- SERVLET → `AnnotationConfigServletWebServerApplicationContext`
- REACTIVE → `AnnotationConfigReactiveWebServerApplicationContext`
- NONE → `AnnotationConfigApplicationContext`

**Создание ApplicationContext**. Пока пустой, без beans. Только со средой (Environment) и указанием на класс конфигурации `@SpringBootApplication`. Регистрируются `ApplicationContextInitializer`-ы — они могут кастомизировать контекст до его refresh.

**Refresh** — самая важная часть. Основной метод создания контекста, где происходит всё: component scan, auto-configuration, создание beans, старт embedded server. Разбираем детально в следующих секциях.

**Публикация ApplicationReadyEvent**. После refresh и всех listeners — приложение готово. Публикуется event, кто заинтересован — реагирует.

## Refresh: главный метод старта

`AbstractApplicationContext.refresh()` — центральный метод. Порядок вложенных операций:

```java
public void refresh() throws BeansException, IllegalStateException {
    prepareRefresh();
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
    prepareBeanFactory(beanFactory);
    postProcessBeanFactory(beanFactory);
    invokeBeanFactoryPostProcessors(beanFactory);
    registerBeanPostProcessors(beanFactory);
    initMessageSource();
    initApplicationEventMulticaster();
    onRefresh();                                    // ← ЗДЕСЬ стартует Tomcat
    registerListeners();
    finishBeanFactoryInitialization(beanFactory);   // ← создание всех singleton beans
    finishRefresh();                                 // ← публикация ContextRefreshedEvent
}
```

Разберём ключевые шаги.

## Component scan и BeanDefinitions

`@ComponentScan` (внутри `@SpringBootApplication`) активирует сканирование пакета и подпакетов. Ищутся классы со стереотипами: `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`, `@Configuration`.

Для каждого найденного класса создаётся **BeanDefinition** — метаданные о том, как этот bean должен быть создан. Не сам bean, а описание: класс, конструктор, зависимости, scope, lifecycle callbacks. BeanDefinitions регистрируются в BeanFactory.

Плюс `@Configuration` классы обрабатываются отдельно — их `@Bean`-методы становятся дополнительными BeanDefinitions.

## Auto-configuration активируется

`@EnableAutoConfiguration` (тоже внутри `@SpringBootApplication`) читает `META-INF/spring.factories` (Boot 2) или `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 3) из всех JAR-ов classpath. Получает список сотен auto-configuration классов.

Для каждого — проверяются `@Conditional*` условия. Если условия выполнены — auto-configuration класс регистрируется как обычный `@Configuration`, его `@Bean`-методы становятся BeanDefinitions.

Именно на этом этапе автоматически появляются основные beans Spring Boot приложения:
- `HikariDataSource` от `DataSourceAutoConfiguration`.
- `EntityManagerFactory`, `PlatformTransactionManager` от `HibernateJpaAutoConfiguration`.
- `RabbitTemplate` от `RabbitAutoConfiguration`.
- `RestTemplate` builder.
- `TomcatServletWebServerFactory` от `ServletWebServerFactoryAutoConfiguration`.
- `DispatcherServlet` от `DispatcherServletAutoConfiguration`.
- `ConsulServiceRegistry` от Spring Cloud Consul auto-configuration.

Всё это регистрируется без единой строчки конфигурации пользователем — auto-config работает.

## Создание singleton beans

`finishBeanFactoryInitialization(beanFactory)` — здесь Spring реально создаёт beans. Идёт по BeanDefinitions, создаёт объекты в правильном порядке — топологическая сортировка графа зависимостей.

Для каждого bean:

**Instantiation**. Вызов конструктора. Если аргументы конструктора — другие beans, они создаются рекурсивно первыми. Так работает constructor injection.

**Populate**. Заполнение полей с `@Autowired`, вызов setter'ов. Для constructor injection этот шаг уже сделан.

**Aware интерфейсы**. Если bean реализует `BeanNameAware`, `ApplicationContextAware` и подобные — Spring вызывает соответствующие setter'ы, подсовывая нужные объекты.

**BeanPostProcessor.postProcessBeforeInitialization**. Пре-хук для BeanPostProcessors. Здесь обрабатывается `@ConfigurationProperties`, применяется `@Autowired` через `AutowiredAnnotationBeanPostProcessor`.

**`@PostConstruct` метод**. Если у bean есть метод с этой аннотацией — вызывается. Инициализация после того как зависимости инжектированы.

**`InitializingBean.afterPropertiesSet`** — если реализован интерфейс.

**Custom init-method** — если объявлен через `@Bean(initMethod = "myInit")`.

**BeanPostProcessor.postProcessAfterInitialization**. Пост-хук. Здесь Spring оборачивает bean в **CGLIB proxy** если нужно (для `@Transactional`, `@Async`, `@Cacheable`). Возвращённый proxy идёт в контекст вместо оригинала.

Если циклическая зависимость через constructor — падает `BeanCurrentlyInCreationException`. Через setter — Spring разруливает через **early reference**: кладёт полусозданный bean в singleton cache, позволяет второму получить ссылку, потом дозаполняет первый.

Порядок создания — топологическая сортировка графа зависимостей. Если A зависит от B, B создаётся первым. Иногда нужно явно указать порядок через `@DependsOn("otherBean")`.

## Прокси для AOP

Ключевой момент — когда bean с `@Transactional` создан, `BeanPostProcessor` смотрит "этому нужен proxy". Оборачивает через CGLIB (или JDK Dynamic Proxy для interface-based beans).

```
FnoService (реальный класс)
       ↑
       │ extends
FnoService$$EnhancerBySpringCGLIB$$xyz (proxy)
       ↑
       │
Injected everywhere as "FnoService"
```

Все вызовы `svc.submit(f)` идут через proxy: `openTransaction()` → `super.submit(f)` (реальный метод) → `commit()`.

Это ключ к тому, почему `@Transactional` работает — proxy оборачивает вызовы. И источник self-invocation gotcha — вызов через `this.method()` минует proxy.

## ContextRefreshedEvent

Публикуется в конце `finishRefresh()`. Все beans созданы, все инъекции сделаны, контекст полностью готов. Подписчики (`@EventListener(ContextRefreshedEvent.class)`) могут отреагировать.

Часто используется для программной инициализации: прогрев кэшей, регистрация в реестре сервисов.

## Старт embedded server

Для web-контекста `onRefresh()` внутри `refresh()` вызывает `createWebServer()`:

Ищется `ServletWebServerFactory` bean — обычно `TomcatServletWebServerFactory`. Он создаётся auto-configuration'ом.

Создаётся `WebServer` через фабрику: `factory.getWebServer(...)`.

`WebServer.start()`:
- Tomcat создаёт `Connector` на настроенном порту (по умолчанию 8080).
- Стартует `ThreadPoolExecutor` с настроенным количеством threads.
- Регистрируется `DispatcherServlet`.
- Настраиваются filters, error handlers.

После этого сервер уже принимает TCP соединения на порту. Но приложение не готово — refresh продолжается. Если запрос придёт на этом этапе, скорее всего обработается через DispatcherServlet — но может произойти error, если ещё не все beans созданы.

Настройка `server.port: 0` — случайный свободный порт. Используется в тестах чтобы не конфликтовать. Реальный порт можно получить через `WebServerApplicationContext.getWebServer().getPort()`.

## Регистрация в Consul

Если в classpath есть `spring-cloud-starter-consul-discovery`, ConsulAutoConfiguration регистрирует нужные beans, включая `ConsulServiceRegistry` и `ConsulAutoServiceRegistration`.

По `SmartApplicationListener` они подписаны на `WebServerInitializedEvent` — событие "web-сервер стартовал и знает свой порт". Когда событие приходит, ConsulAutoServiceRegistration отправляет в Consul HTTP запрос:

```json
PUT /v1/agent/service/register
{
  "Name": "isnaKnpIntegration",
  "ID": "isnaKnpIntegration-<uuid>",
  "Address": "10.0.1.5",
  "Port": 8080,
  "Check": {
    "HTTP": "http://10.0.1.5:8080/actuator/health",
    "Interval": "15s"
  }
}
```

Consul сохраняет сервис в своей БД. Начинает дёргать health check каждые 15 секунд. Пока endpoint возвращает 200 — сервис `passing`, включён в discovery.

Важный gotcha: при shutdown приложения нужно **deregister** из Consul. Spring делает это через shutdown hook. Но если под убили `kill -9` (без graceful) — hook не выполнится. Consul сам через `deregisterCriticalServiceAfter` (обычно 30 минут) удалит stale entry.

Реальный кейс из КНП: `knp-fs-consul-deregister-after-db-flap` — при флапе DB приложение оставалось живым, но Consul health check фейлил → сервис исключался. После восстановления DB приложение продолжало работать, но не re-регистрировалось. Лечили через `rollout restart`.

## CommandLineRunner и ApplicationRunner

После создания всех beans, но до `ApplicationReadyEvent`, выполняются `CommandLineRunner` и `ApplicationRunner` — специальные интерфейсы для инициализации.

```java
@Component
public class Warmup implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        // выполнится после старта, до publishing ApplicationReadyEvent
        log.info("Warmup started");
        cacheService.warmup();
    }
}
```

Разница от `@PostConstruct`. PostConstruct вызывается **при создании конкретного bean**, до старта embedded server, до других beans. CommandLineRunner — **после того как контекст полностью готов**, embedded server стартовал. Подходит для warmup, требующего доступа ко всем сервисам приложения.

Разница от `ApplicationReadyEvent`. CommandLineRunner — до события ready, event handlers — после. Практически идентично для большинства случаев.

## ApplicationReadyEvent

Финальное событие startup. Публикуется когда:
- Refresh контекста завершён.
- Embedded server стартовал.
- Все CommandLineRunner/ApplicationRunner выполнились.

Приложение полностью готово принимать трафик. Можно подписаться:

```java
@EventListener(ApplicationReadyEvent.class)
public void onReady() {
    log.info("Application is ready to serve traffic");
    // регистрация в внешних системах
    // отправка уведомлений
    // старт scheduled jobs
}
```

Полезно для действий, требующих полной готовности системы, включая web-сервер.

## Kubernetes probes

K8s поднял под с Docker контейнером. Контейнер стартовал, процесс Java пошёл через все описанные фазы. K8s не знает когда приложение готово — он проверяет через probes.

**Startup probe** (K8s 1.16+) — для медленных стартов. Указывается endpoint и таймауты. Пока startup probe не пройдёт, liveness и readiness probes не проверяются. Это защищает медленно стартующие приложения от преждевременного kill'а.

**Liveness probe** — периодически проверяет "жив ли". Fail несколько раз подряд (по `failureThreshold`) → kubelet убивает контейнер, поднимает новый. Используется endpoint `/actuator/health/liveness`. Он должен возвращать UP пока приложение живо, независимо от внешних систем. Если DB упала — liveness всё равно UP, потому что приложение работает.

**Readiness probe** — периодически проверяет "готов ли принимать трафик". Fail → под остаётся живым, но исключается из Service. Трафик не идёт на него. Используется endpoint `/actuator/health/readiness`. Может зависеть от внешних систем — если DB недоступна, приложение не готово обслуживать, лучше вывести из ротации.

Типовая конфигурация в deployment yaml:

```yaml
containers:
- name: app
  image: isna/knp-integration:1.0.42
  ports:
  - containerPort: 8080
  startupProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    failureThreshold: 30       # 30 * 10s = 5 минут на старт
    periodSeconds: 10
  livenessProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    initialDelaySeconds: 60    # старт probe не сразу
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

## Полная временная шкала примера

Реалистичный старт Spring Boot микросервиса из КНП:

```
t=0.0s   java -jar → JVM init
t=0.3s   JarLauncher, LaunchedURLClassLoader ready
t=0.5s   KnpApplication.main вызван
t=0.5s   SpringApplication.run, баннер
t=0.6s   Environment (yml, profiles, env, cli)
t=0.7s   Component scan начался
t=2.5s   Component scan завершён, auto-configuration processed
         Все BeanDefinitions зарегистрированы
t=2.6s   Создание singleton beans (JPA/Hibernate — самое долгое)
t=6.0s   EntityManagerFactory готов, JPA repositories созданы
t=6.5s   Tomcat старт на 8080, DispatcherServlet зарегистрирован
t=6.6s   Consul register (HTTP запрос к Consul)
t=6.7s   ContextRefreshedEvent published
t=6.8s   CommandLineRunners выполнены
t=6.9s   ApplicationReadyEvent published
t=6.9s   Приложение готово!

t=16s    K8s: первый успешный readinessProbe → под Ready → Service направляет трафик
```

Средний Spring Boot микросервис в КНП стартует 5-15 секунд. С JPA плюс Liquibase миграциями — до минуты. AOT-компилированный native image через GraalVM — 100-300 мс, но много Spring магии не работает без специального конфигурирования.

## Что может сломать старт

Классические проблемы startup:

**ClassNotFoundException / NoClassDefFoundError** — пропала зависимость или конфликт версий. Проверяется через `jar tf myapp.jar | grep <классnn>`, `./gradlew dependencies`.

**NoSuchMethodError** — версия библиотеки не совместима. Классы есть, но метод пропал или сигнатура другая. Реальный кейс КНП: `knp-form-hz5-actuator-cache-nosuchmethod` — Hazelcast 5 + Boot 2.2 actuator, `getNativeCache()` пропал.

**BeanCreationException** — не смог создать bean. Причины: циклическая зависимость (в Boot 2.6+ запрещено по умолчанию), не хватает bean для инъекции (`UnsatisfiedDependencyException`), ошибка в конструкторе, ошибка в `@PostConstruct`.

**Port 8080 already in use** — другой процесс держит порт. `netstat -tlnp | grep 8080` или другой контейнер на той же ноде K8s.

**Failed to configure a DataSource** — не хватает `spring.datasource.url` для auto-config, Spring не может понять что подключать. Либо задать URL, либо исключить `DataSourceAutoConfiguration` если DB реально не нужна.

**Liquibase / Flyway ошибка миграции** — pending миграция, конфликт версий, corrupted checksum. Проверять через changelog history table.

**Consul недоступен** — если `spring.cloud.consul.discovery.fail-fast=true`, приложение падает при невозможности зарегистрироваться. С `fail-fast=false` продолжит без registration.

**HikariCP configuration issues** — реальный кейс КНП: `knp-e2e-runner-hikari-isolation-poisoning` — pool отравлен `isolation=-1` из-за некорректного возврата connection в pool.

## Инструменты диагностики startup

Флаг **`--debug`** активирует auto-configuration report при старте. Показывает какие auto-configs сработали, какие нет, почему.

**Actuator `/actuator/conditions`** — то же самое через HTTP в runtime.

**Actuator `/actuator/startup`** (Boot 2.4+) — timeline старта, что сколько заняло. Мощный инструмент для оптимизации startup time.

**`-Xlog:class+load=info`** — JVM опция, логирует какие классы загружаются и откуда. Полезно для отладки classloader конфликтов.

**Thread dump на старте** — если приложение зависло на старте, `jcmd <pid> Thread.print` или `jstack` покажет что делает главный поток.

**Startup profile через flight recorder** — `-XX:StartFlightRecording=duration=60s,filename=startup.jfr`. Открыть в Java Mission Control — детальная профилировка первой минуты работы.

## Оптимизация startup time

Медленный startup — проблема для микросервисной архитектуры. При частых деплоях и autoscaling время старта напрямую влияет на agility.

Стратегии оптимизации.

**Уменьшить classpath** — меньше JAR-ов → быстрее auto-configuration evaluation. Убрать неиспользуемые зависимости.

**Исключить лишние auto-configurations** — то что не нужно, отключить через `spring.autoconfigure.exclude`. Особенно если много сomponents Spring Cloud или Actuator, которые не используются.

**Lazy initialization** — `spring.main.lazy-initialization=true`. Beans создаются лениво при первом обращении. Startup быстрый, первый запрос медленный. Trade-off.

**Class Data Sharing (CDS)** — JVM опция для кэширования метаданных загруженных классов между запусками. Спокойное улучшение 20-30% startup time.

**Ahead-of-Time compilation** через Spring AOT (Boot 3.0+). Часть работы Spring делается на этапе сборки, не runtime. Существенно быстрее старт.

**GraalVM Native Image** — full AOT компиляция в native binary. Startup 100-300 мс. Но много ограничений: reflection только с explicit config, ограниченная динамика, отсутствие некоторых Spring фичей.

Для типичного КНП сервиса — оставить как есть, 10-15 секунд старта приемлемо. Оптимизация только когда есть конкретная причина.

## Заключение

Startup Spring Boot приложения — сложная последовательность из десятков шагов, каждый с потенциальными проблемами. Понимание её структуры критично для разработки в enterprise контексте.

Основные фазы: JVM init → JarLauncher создаёт LaunchedURLClassLoader → SpringApplication создаёт Environment и определяет тип приложения → ApplicationContext.refresh() выполняет component scan, auto-configuration, создаёт beans (включая wrapping в CGLIB proxy для `@Transactional`), стартует embedded server → регистрация в Consul → CommandLineRunners → ApplicationReadyEvent → K8s probes проверяют готовность → трафик начинает идти.

Ключевые точки для практики. `@PostConstruct` для инициализации отдельного bean. `CommandLineRunner` для warmup после старта контекста. `@EventListener(ApplicationReadyEvent.class)` для действий требующих полной готовности включая web-сервер. K8s probes: startup для медленных стартов, liveness для "жив ли", readiness для "готов ли принимать трафик".

Диагностика проблем: `--debug` для auto-configuration report, `/actuator/conditions` через HTTP, `/actuator/startup` для timeline, thread dump для зависаний. Знание типичных проблем (ClassNotFoundException, NoSuchMethodError, BeanCreationException) и способов их отладки.

Для КНП контекста особенно важно понимание Consul integration — как приложение регистрируется, что происходит при флапе, важность deregister при shutdown. И знание Liquibase/Flyway migration flow, потому что миграции — частая точка сбоя старта.

Знание всей startup цепочки даёт возможность правильно локализовать проблему когда приложение не стартует. "500 при старте" — где именно? В auto-configuration? В bean creation? В embedded server start? В Consul registration? Ответ на этот вопрос — половина решения проблемы.

Дальше — Docker для контейнеризации Spring Boot приложения. Все эти фазы startup будут происходить внутри контейнера, и понимание Docker необходимо для правильного deployment.
