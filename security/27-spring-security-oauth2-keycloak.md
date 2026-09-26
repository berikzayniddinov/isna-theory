# 27. Spring Security плюс OAuth2 плюс Keycloak в production

## Роли Spring Boot приложения в OAuth2

Микросервис на Spring Boot может играть разные роли в OAuth2 инфраструктуре. Понимание этих ролей определяет выбор dependencies и конфигурации.

OAuth2 Resource Server это наиболее частая роль. Приложение принимает Bearer токены от вызывающих клиентов, проверяет их валидность, извлекает authorities для authorization. Не занимается процессом получения tokens — это ответственность authorization server. Большинство backend микросервисов в КНП именно Resource Server — принимают tokens и обслуживают API запросы.

OAuth2 Client роль для приложений которые сами инициируют OAuth2 flow — например BFF backend for frontend получающий tokens для UI, или backend integration получающий tokens для вызова downstream сервисов через Client Credentials.

Login клиент это специализированная роль обычной web-app с server-side login через OIDC. Пользователь logs in через redirect на authorization server, приложение получает tokens, устанавливает session. Классический server-rendered UI application.

Соответствующие Spring Boot starters. spring-boot-starter-oauth2-resource-server для Resource Server функциональности — валидация JWT, извлечение authorities, integration с Spring Security. spring-boot-starter-oauth2-client для Client функциональности — инициация OAuth2 flows, управление tokens, интерцепция HTTP запросов для добавления Authorization header.

Приложение может играть несколько ролей одновременно. Backend integration может быть и Resource Server (принимает tokens от frontend) и Client (получает tokens для downstream calls). В этом случае подключаются оба starters.

## Настройка Resource Server с JWT

Минимальная конфигурация Resource Server требует одну строку в application.yml:
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.isna/realms/knp
```

Spring автоматически выполняет несколько шагов при старте. Читает discovery document от issuer-uri через путь .well-known/openid-configuration. Из discovery берёт jwks_uri содержащий публичные ключи. Настраивает JwtDecoder использующий NimbusJwtDecoder из библиотеки Nimbus JWT. Регистрирует BearerTokenAuthenticationFilter в security filter chain для перехвата Authorization headers.

Каждый входящий HTTP запрос обрабатывается следующим образом. Filter извлекает Authorization header, ищет Bearer prefix, извлекает token. JwtDecoder проверяет подпись через ключи из JWKS, валидирует стандартные claims включая expiration и issuer. При успехе создаётся JwtAuthenticationToken с extracted authorities, помещается в SecurityContext. При неудаче — 401 Unauthorized в response.

Минимальная SecurityFilterChain для Resource Server:
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filter(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```

Этого достаточно для функционального Resource Server. Каждый endpoint кроме /actuator требует валидный JWT. Токен парсится, но по default только scope claim используется для authorities что обычно неподходяще для Keycloak.

## Проблема Keycloak roles в JWT

Keycloak размещает роли в специфической структуре не понимаемой Spring по default:
```json
{
    "sub": "b3f2a1c4-...",
    "preferred_username": "berik",
    "realm_access": {
        "roles": ["admin", "user"]
    },
    "resource_access": {
        "isna-knp-integration": {
            "roles": ["CREATE_FNO", "VIEW_FNO"]
        }
    }
}
```

Spring читает только claim scope преобразуя values в authorities с prefix SCOPE_. Keycloak роли из realm_access и resource_access полностью игнорируются. Endpoints с @PreAuthorize hasRole или hasAuthority на Keycloak роли не будут работать — Spring просто не видит эти роли как authorities.

Решение — custom JwtAuthenticationConverter преобразующий Keycloak-specific claims в стандартные Spring authorities. Converter получает Jwt объект (decoded token) и должен вернуть Collection GrantedAuthority.

Реализация. Извлечь realm_access claim как Map. Из него извлечь roles как List. Каждую роль преобразовать в SimpleGrantedAuthority с prefix ROLE_ (для использования через hasRole). Извлечь resource_access как Map. Из него извлечь конкретного client (например isna-knp-integration) как Map. Из client access извлечь roles как List. Каждую роль преобразовать в SimpleGrantedAuthority без prefix (для использования через hasAuthority).

Пример полного converter:
```java
@Bean
JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Collection<GrantedAuthority> authorities = new ArrayList<>();
        
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess != null) {
            List<String> roles = (List<String>) realmAccess.get("roles");
            if (roles != null) {
                roles.forEach(r -> 
                    authorities.add(new SimpleGrantedAuthority("ROLE_" + r)));
            }
        }
        
        Map<String, Object> resourceAccess = jwt.getClaim("resource_access");
        if (resourceAccess != null) {
            Map<String, Object> clientAccess = 
                (Map<String, Object>) resourceAccess.get("isna-knp-integration");
            if (clientAccess != null) {
                List<String> roles = (List<String>) clientAccess.get("roles");
                if (roles != null) {
                    roles.forEach(r -> 
                        authorities.add(new SimpleGrantedAuthority(r)));
                }
            }
        }
        
        return authorities;
    });
    return converter;
}
```

Подключение converter к SecurityFilterChain:
```java
http.oauth2ResourceServer(o -> 
    o.jwt(jwt -> jwt.jwtAuthenticationConverter(conv)));
```

После этого работают проверки. hasRole("admin") проверяет наличие authority ROLE_admin из realm roles. hasAuthority("CREATE_FNO") проверяет наличие CREATE_FNO из client roles. Оба стиля работают одновременно позволяя гибкую авторизацию.

## Использование в коде

Аутентифицированные пользователи доступны в коде через несколько способов. Самый простой — @AuthenticationPrincipal в контроллере:
```java
@PostMapping
@PreAuthorize("hasAuthority('CREATE_FNO')")
public Fno create(@RequestBody FnoDto dto, @AuthenticationPrincipal Jwt jwt) {
    String username = jwt.getSubject();
    String email = jwt.getClaimAsString("email");
    return svc.create(dto, username);
}
```

Автоматическая injection Jwt объекта содержащего все decoded claims. Методы getSubject, getClaimAsString, getClaimAsMap для типизированного доступа. Никакого manual парсинга.

Из service слоя доступ через SecurityContextHolder:
```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
Jwt jwt = (Jwt) auth.getPrincipal();
String userId = jwt.getSubject();
```

Полезно когда security context нужен не в контроллере а глубоко в бизнес-логике. Thread-local природа SecurityContext означает автоматическую доступность в текущем request thread.

Method Security через @PreAuthorize даёт декларативную авторизацию:
```java
@PreAuthorize("hasRole('user')")
public Fno get(@PathVariable Long id) { ... }

@PreAuthorize("hasRole('admin')")
public void delete(@PathVariable Long id) { ... }

@PreAuthorize("hasAuthority('CREATE_FNO') and #dto.userId == authentication.name")
public Fno create(@RequestBody FnoDto dto) { ... }
```

Комбинации через and, or, not. Доступ к method arguments через #argName. Доступ к authentication через authentication SpEL variable. Мощный механизм для fine-grained authorization.

## Дополнительные проверки JWT

Стандартные проверки выполняемые Spring — signature validity через public keys из JWKS, expiration через exp claim, not-before через nbf claim, issuer через iss claim (если задано в issuer-uri).

Audience валидация проверяет что токен предназначен для этого resource server. Настройка через свойство:
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.isna/realms/knp
          audiences:
            - isna-knp-integration
```

При настроенных audiences Spring проверяет что aud claim в токене содержит один из указанных значений. Иначе токен отклоняется.

Программная настройка через custom JwtDecoder для complex сценариев:
```java
@Bean
JwtDecoder jwtDecoder(@Value("${issuer}") String issuer) {
    NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuer);
    OAuth2TokenValidator<Jwt> withIssuer = new JwtIssuerValidator(issuer);
    OAuth2TokenValidator<Jwt> withAudience = new AudienceValidator("isna-knp-integration");
    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
        JwtValidators.createDefault(), withIssuer, withAudience));
    return decoder;
}
```

JwtValidators.createDefault включает проверки exp и nbf. Custom validators добавляются для audience, custom claims, business specific validation.

## OAuth2 Client для вызовов между сервисами

Микросервис isna-knp-integration должен вызвать isna-knp-user API. Для этого нужен valid access token, получаемый через Client Credentials flow.

Конфигурация client registration:
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          knp-service-client:
            client-id: isna-knp-integration-service
            client-secret: ${KEYCLOAK_SECRET}
            authorization-grant-type: client_credentials
            scope: openid
        provider:
          keycloak:
            issuer-uri: https://keycloak.isna/realms/knp
```

OAuth2AuthorizedClientManager это компонент управляющий tokens для клиентов. Автоматически получает новые tokens при истечении, кэширует до expiration:
```java
@Bean
OAuth2AuthorizedClientManager auth2ClientManager(
        ClientRegistrationRepository clientRegistrationRepository,
        OAuth2AuthorizedClientRepository authorizedClientRepository) {
    
    OAuth2AuthorizedClientProvider provider =
        OAuth2AuthorizedClientProviderBuilder.builder()
            .clientCredentials()
            .build();
    
    DefaultOAuth2AuthorizedClientManager manager =
        new DefaultOAuth2AuthorizedClientManager(
            clientRegistrationRepository, authorizedClientRepository);
    manager.setAuthorizedClientProvider(provider);
    return manager;
}
```

Использование через RestClient с interceptor автоматически добавляющим Bearer token:
```java
@Bean
RestClient restClient(OAuth2AuthorizedClientManager manager) {
    return RestClient.builder()
        .requestInterceptor((req, body, exec) -> {
            OAuth2AuthorizeRequest authReq = OAuth2AuthorizeRequest
                .withClientRegistrationId("knp-service-client")
                .principal("system")
                .build();
            OAuth2AuthorizedClient client = manager.authorize(authReq);
            req.getHeaders().setBearerAuth(client.getAccessToken().getTokenValue());
            return exec.execute(req, body);
        })
        .build();
}
```

Каждый исходящий запрос через этот RestClient автоматически имеет Authorization header с valid access token. Обработка refresh скрыта в OAuth2AuthorizedClientManager.

Feign integration через RequestInterceptor работает аналогично:
```java
@Component
class OAuth2FeignRequestInterceptor implements RequestInterceptor {
    @Autowired OAuth2AuthorizedClientManager manager;
    
    public void apply(RequestTemplate template) {
        OAuth2AuthorizeRequest req = OAuth2AuthorizeRequest
            .withClientRegistrationId("knp-service-client")
            .principal("system")
            .build();
        OAuth2AuthorizedClient client = manager.authorize(req);
        template.header("Authorization", 
            "Bearer " + client.getAccessToken().getTokenValue());
    }
}
```

## Пропагация user token

Иногда downstream сервис должен видеть оригинального пользователя не service account. Например для audit purposes или user-specific логики. Пропагация user token означает переиспользование JWT текущего пользователя в исходящих запросах.

Извлечение user token из SecurityContext:
```java
Jwt jwt = (Jwt) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
String userToken = jwt.getTokenValue();
```

Использование в исходящем запросе:
```java
restClient.get()
    .uri("/api/other-service/data")
    .header("Authorization", "Bearer " + userToken)
    .retrieve();
```

Такой подход даёт downstream сервису full контекст оригинального пользователя. Downstream может делать authorization checks на основе original user roles, логировать user id в audit, применять data filtering по user context.

Trade-off — token size, network overhead, dependency на original session. Если session истёк, downstream call тоже неудачен. Альтернатива — service-to-service Client Credentials с явной передачей user context как параметр (custom header или body field).

## Тестирование

Spring Security Test предоставляет utilities для mocking JWT в тестах. Обычно основанные на аннотациях или fluent API.

MockMvc с mock JWT:
```java
mockMvc.perform(post("/api/fno")
    .with(jwt().authorities(new SimpleGrantedAuthority("CREATE_FNO")))
    .content("{...}"))
    .andExpect(status().isOk());
```

Fluent API .with(jwt()) устанавливает mock JWT authentication для запроса. Authorities передаются через builder methods. Можно установить specific claims если тест их проверяет.

Custom @WithMockJwt через meta annotation для более удобного использования:
```java
@Test
@WithMockJwt(subject = "berik", authorities = "CREATE_FNO")
void authorized() {
    // тест выполняется с mock JWT
}
```

Real Keycloak в интеграционных тестах через TestContainers. KeycloakContainer запускает Docker контейнер с preconfigured realm. Приложение тестируется с реальным authentication:
```java
@Container
static KeycloakContainer keycloak = 
    new KeycloakContainer("quay.io/keycloak/keycloak:24.0")
        .withRealmImportFile("test-realm.json");

@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    r.add("spring.security.oauth2.resourceserver.jwt.issuer-uri",
        () -> keycloak.getAuthServerUrl() + "/realms/test");
}
```

Медленнее unit tests но даёт real integration coverage. Хорошо для критических security paths где mock коверов может skip important edge cases.

## Кэширование JWKS

Spring по default кэширует JWKS ключи в memory. При появлении JWT с новым kid не найденным в кэше — refreshes JWKS. Кэш живёт indefinitely пока не будет обновлён.

Проблема — если Keycloak делает key rotation, а Spring запрашивал JWKS давно, новые токены могут не валидироваться пока не произойдёт refresh. Возможен window ошибок 401 у пользователей.

Настройка explicit cache duration:
```java
@Bean
JwtDecoder jwtDecoder(@Value("${issuer}") String issuer) {
    return NimbusJwtDecoder.withIssuerLocation(issuer)
        .cache(Duration.ofMinutes(5))
        .build();
}
```

5 минут баланс между свежестью данных и загрузкой на Keycloak. Слишком часто (например 1 минута) создаёт unnecessary load. Слишком редко (например часы) создаёт длинные windows при rotation.

Alternative — restOperations с retry logic на JWKS для resilience при temporary Keycloak issues.

## Классификация ошибок

401 Unauthorized означает проблему с authentication. Возможные причины включают отсутствие токена — header Authorization не предоставлен. Токен истёк — exp claim в прошлом. Signature invalid — токен подписан не тем ключом или tampered. Issuer mismatch — iss claim не совпадает с ожидаемым. Audience mismatch — если проверка настроена, aud не содержит expected value.

403 Forbidden означает что authentication successful но нет прав. Токен валиден но не содержит требуемые roles/authorities. hasRole("admin") на endpoint, а в токене нет ROLE_admin. Полезное разграничение — 401 «предъявите valid credentials», 403 «credentials valid но недостаточно прав».

NoSuchMethodError на claim access это известная проблема при skew версий Spring Security компонентов. Реальный кейс taxreport21-java21-runtime-regressions в КНП — spring-security-jose и spring-security-core оказались разных версий в classpath, при вызове методов возникал NoSuchMethodError потому что method signature изменилась между версиями. Fix — явно закрепить версии через Spring Boot BOM что гарантирует consistency всех Spring компонентов.

JWKS unavailable ошибка при недоступности Keycloak. Все запросы падают с 401 потому что Spring не может валидировать подпись без публичных ключей. Health check Keycloak критически важен. Alerting на JWKS availability. Circuit breaker на JWKS fetch для fast fail при sustained downtime.

Разные issuers в multi-tenant setup требуют специального подхода. По default Spring поддерживает один issuer. Multi-tenant через AuthenticationManagerResolver позволяющий routing запросов к разным JwtDecoders на основе токена:
```java
@Bean
AuthenticationManagerResolver<HttpServletRequest> resolver() {
    return request -> {
        String issuer = extractIssuerFromToken(request);
        return authManagerForIssuer(issuer);
    };
}
```

Каждый tenant имеет свой Keycloak realm или даже отдельный Keycloak, Spring динамически выбирает правильный decoder на основе issuer claim в JWT.

## Метрики и логирование

Auth events logging помогает audit и debugging. Spring публикует events для authentication success, failure, logout. Регистрация listener позволяет реагировать:
```java
@Component
class AuthEventListener {
    @EventListener
    void onAuthSuccess(AuthenticationSuccessEvent event) {
        log.info("Login success: {}", event.getAuthentication().getName());
    }
    
    @EventListener
    void onAuthFailure(AbstractAuthenticationFailureEvent event) {
        log.warn("Login failed: {}", event.getException().getMessage());
    }
}
```

Structured logging с MDC context (correlation ID, user id) в auth events обеспечивает traceability. Alerting на unusual patterns — множественные failures от одного IP, login outside business hours для privileged accounts.

Critical rule — никогда не логировать tokens. Authorization header содержит sensitive material. Полный token в логах = complete compromise любого legitimate access. Стандартный logback pattern для маскирования:
```xml
<pattern>%replace(%msg){'Bearer [A-Za-z0-9._-]+', 'Bearer ***'}%n</pattern>
```

Regex заменяет любой Bearer token на маркер. Similar patterns для других sensitive data. Обязательный элемент production logging configuration.

Metrics через Actuator и Micrometer. Стандартные metrics включают request count, latencies, error rates разбитые по authenticated/unauthenticated. Custom metrics для auth-specific показателей — successful logins, failed logins, token refresh rate, JWKS cache hits/misses.

## Полная production конфигурация

Собранная воедино конфигурация Resource Server plus Client в микросервисе КНП:
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER:https://keycloak.isna/realms/knp}
          audiences:
            - isna-knp-integration
      client:
        registration:
          knp-service:
            client-id: isna-knp-integration
            client-secret: ${KEYCLOAK_SECRET}
            authorization-grant-type: client_credentials
        provider:
          keycloak:
            issuer-uri: ${KEYCLOAK_ISSUER}

logging:
  level:
    org.springframework.security: INFO
  pattern:
    console: "%d %-5level [%X{traceId:-},%X{spanId:-}] %logger - %replace(%msg){'Bearer [A-Za-z0-9._-]+', 'Bearer ***'}%n"
```

Java конфигурация с всеми custom bindings:
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    @Bean
    SecurityFilterChain filter(HttpSecurity http, 
                               JwtAuthenticationConverter conv) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**", 
                                "/actuator/info", 
                                "/actuator/prometheus").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> 
                o.jwt(jwt -> jwt.jwtAuthenticationConverter(conv)))
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
                .accessDeniedHandler((req, resp, e) -> resp.setStatus(403)))
            .headers(h -> h.frameOptions(f -> f.deny()));
        return http.build();
    }
    
    @Bean
    JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter c = new JwtAuthenticationConverter();
        c.setJwtGrantedAuthoritiesConverter(
            new KeycloakRealmAndClientRolesConverter("isna-knp-integration"));
        c.setPrincipalClaimName("preferred_username");
        return c;
    }
}
```

## Реальные кейсы КНП

Кейс taxreport21-java21-runtime-regressions — OAuth2 NoSuchMethodError при вызове JWT parsing methods. Root cause — skew версий spring-security-jose и spring-security-core в classpath. Fix — enforce version consistency через Spring Boot BOM. Ensure всех Spring dependencies собраны через BOM без manual version overrides.

Кейс knp-e2e-prod-taxrep21-smoke-local-run — bearer канал НЗ падает при миграции на Java 21 из-за incompatibility Spring Security с новой версией Java. Не проблема самого OAuth2, а general Spring Security compatibility при major Java migrations. Fix — обновить Spring Security до совместимой версии, retest все auth paths.

Кейс knp-n07-sent-documents-invisible — permission KNP_PERM_CREATE_NZ_N07 как client role в Keycloak, permission gate в коде fv.code IN permissionCodes. Если пользователю не назначена эта роль в Keycloak, N07 форма не показывается в отправленных документах. Урок — permission model должна быть end-to-end consistent между Keycloak configuration и application code, missing role в Keycloak даёт silent invisibility а не explicit error.

## Итоги

Spring Security OAuth2 Resource Server стандартный подход для микросервисов КНП после deprecation Keycloak adapters. Одна строка issuer-uri автоматически конфигурирует JWT validation через discovery и JWKS.

Custom JwtAuthenticationConverter необходим для правильного парсинга Keycloak-specific структуры ролей в realm_access и resource_access. Стандартный Spring подход к authorities не понимает эту структуру без customization.

Method Security через @PreAuthorize даёт fine-grained authorization на уровне отдельных methods. hasRole для realm roles, hasAuthority для permissions.

OAuth2 Client функциональность через OAuth2AuthorizedClientManager для machine-to-machine calls между сервисами. RestClient и Feign integration через interceptors автоматически добавляют Bearer token в исходящие запросы.

Пропагация user token в downstream calls когда нужен original user context. Trade-off между full context и dependency на original session.

Audience валидация через audiences свойство или AudienceValidator. Multi-tenant поддержка через AuthenticationManagerResolver для разных issuers.

Тестирование через MockMvc с jwt() fluent или @WithMockJwt для unit tests. TestContainers с KeycloakContainer для integration tests с real Keycloak.

Кэширование JWKS с explicit duration критично для баланса свежести и load. 5 минут разумный default.

Ошибки 401 vs 403 разграничивают authentication и authorization проблемы. NoSuchMethodError указывает на version skew. JWKS unavailability ломает все токены — health check и alerting Keycloak critical.

Никогда не логировать tokens — full compromise любого доступа. Regex маскирование в logback pattern стандартный элемент security configuration.

Итог блока Spring Security и OAuth2 и Keycloak — файлы 24-27 покрыли Spring Security основы, OAuth2/OIDC теорию, Keycloak как IAM, integration в production. Дальше начинается блок PostgreSQL, HikariCP, highload с файла 28 postgresql-internals.
