# 27. Spring Security + OAuth2 + Keycloak в проде

Как правильно настроить Spring Security как OAuth2 Resource Server с Keycloak. Прод-грейд.

---

## 1. Роли Spring Boot приложения в OAuth2

Spring Boot микросервис может быть:

1. **OAuth2 Resource Server** — принимает Bearer-токены, проверяет, авторизует. **Самый частый случай** для API.
2. **OAuth2 Client** — сам инициирует OAuth2 flow (для UI backend).
3. **Login клиент** — обычная web-app с OIDC login.

Разные стартеры:
```gradle
// Resource Server (для API)
implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'

// Client (для инициации flow / Login)
implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
```

В ИСНА большинство микросервисов = **Resource Server**.

---

## 2. Настройка Resource Server (JWT)

### 2.1 Минимальная

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.isna/realms/knp
```

Spring:
1. Читает `.well-known/openid-configuration` от `issuer-uri`.
2. Из discovery берёт `jwks_uri`.
3. Настраивает `JwtDecoder` (NimbusJwtDecoder).
4. Регистрирует `BearerTokenAuthenticationFilter` в цепочке.

Всё. Каждый входящий запрос:
1. Извлекает `Authorization: Bearer ...`.
2. Проверяет JWT (подпись, exp, iss, aud).
3. Валиден → создаёт `JwtAuthenticationToken`, кладёт в SecurityContext.
4. Не валиден → 401.

### 2.2 Минимальная security config

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

Готово! Все endpoints требуют валидный JWT.

---

## 3. Проблема: Keycloak roles в JWT

Keycloak кладёт роли специфично:
```json
{
    "sub": "berik",
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

Spring по умолчанию читает `scope` claim → создаёт `SCOPE_read`, `SCOPE_write`. Keycloak-роли не подхватываются.

### 3.1 Кастомный JwtAuthenticationConverter

```java
@Bean
JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Collection<GrantedAuthority> auths = new ArrayList<>();

        // realm roles
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess != null) {
            List<String> roles = (List<String>) realmAccess.get("roles");
            if (roles != null) {
                roles.forEach(r -> auths.add(new SimpleGrantedAuthority("ROLE_" + r)));
            }
        }

        // client roles for this app
        Map<String, Object> resourceAccess = jwt.getClaim("resource_access");
        if (resourceAccess != null) {
            Map<String, Object> clientAccess = (Map<String, Object>) resourceAccess.get("isna-knp-integration");
            if (clientAccess != null) {
                List<String> roles = (List<String>) clientAccess.get("roles");
                if (roles != null) {
                    roles.forEach(r -> auths.add(new SimpleGrantedAuthority(r)));
                }
            }
        }

        return auths;
    });
    return converter;
}
```

Теперь:
- `realm_access.roles` = `["admin"]` → authority `ROLE_admin` (можно `hasRole("admin")`).
- `resource_access.isna-knp-integration.roles` = `["CREATE_FNO"]` → authority `CREATE_FNO` (можно `hasAuthority("CREATE_FNO")`).

### 3.2 Подключить converter

```java
@Bean
SecurityFilterChain filter(HttpSecurity http, JwtAuthenticationConverter conv) throws Exception {
    http
        // ...
        .oauth2ResourceServer(o -> o.jwt(jwt -> jwt.jwtAuthenticationConverter(conv)));
    return http.build();
}
```

---

## 4. Использование

### 4.1 В контроллере

```java
@RestController
@RequestMapping("/api/fno")
class FnoController {

    @PostMapping
    @PreAuthorize("hasAuthority('CREATE_FNO')")
    public Fno create(@RequestBody FnoDto dto, @AuthenticationPrincipal Jwt jwt) {
        String username = jwt.getSubject();
        return svc.create(dto, username);
    }

    @GetMapping("/{id}")
    @PreAuthorize("hasRole('user')")
    public Fno get(@PathVariable Long id) {
        return svc.get(id);
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('admin')")
    public void delete(@PathVariable Long id) {
        svc.delete(id);
    }
}
```

### 4.2 В сервисе

```java
@Service
class MyService {

    void doWork() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        Jwt jwt = (Jwt) auth.getPrincipal();
        String userId = jwt.getSubject();
        String email = jwt.getClaimAsString("email");
        // ...
    }
}
```

---

## 5. Настройка проверок JWT

Дополнительно:
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.isna/realms/knp
          audiences:
            - isna-knp-integration           # проверка aud
```

Или программно:
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

`createDefault()` уже проверяет `exp`, `nbf`, `iss` (если задано).

---

## 6. OAuth2 Client — вызовы других сервисов

Микросервис `isna-knp-integration` хочет позвать `isna-knp-user`. Нужен токен.

### 6.1 Client Credentials

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          isnaknpuser-client:
            client-id: isna-knp-integration-service
            client-secret: ${KEYCLOAK_SECRET}
            authorization-grant-type: client_credentials
            scope: openid
        provider:
          keycloak:
            issuer-uri: https://keycloak.isna/realms/knp
```

### 6.2 Использование через RestClient (Boot 3.2+)

```java
@Bean
OAuth2AuthorizedClientManager auth2ClientManager(
        ClientRegistrationRepository clientRegistrationRepository,
        OAuth2AuthorizedClientRepository authorizedClientRepository) {

    OAuth2AuthorizedClientProvider authorizedClientProvider =
        OAuth2AuthorizedClientProviderBuilder.builder()
            .clientCredentials()
            .build();

    DefaultOAuth2AuthorizedClientManager manager =
        new DefaultOAuth2AuthorizedClientManager(
            clientRegistrationRepository, authorizedClientRepository);
    manager.setAuthorizedClientProvider(authorizedClientProvider);
    return manager;
}

@Bean
RestClient restClient(OAuth2AuthorizedClientManager manager) {
    return RestClient.builder()
        .requestInterceptor((req, body, exec) -> {
            OAuth2AuthorizeRequest authReq = OAuth2AuthorizeRequest
                .withClientRegistrationId("isnaknpuser-client")
                .principal("system")
                .build();
            OAuth2AuthorizedClient client = manager.authorize(authReq);
            req.getHeaders().setBearerAuth(client.getAccessToken().getTokenValue());
            return exec.execute(req, body);
        })
        .build();
}
```

Теперь `restClient` автоматом получает токен + добавляет в header.

### 6.3 Feign + OAuth2

Для Feign — RequestInterceptor:
```java
@Component
class OAuth2FeignRequestInterceptor implements RequestInterceptor {

    @Autowired OAuth2AuthorizedClientManager manager;

    public void apply(RequestTemplate template) {
        OAuth2AuthorizeRequest req = OAuth2AuthorizeRequest
            .withClientRegistrationId("isnaknpuser-client")
            .principal("system")
            .build();
        OAuth2AuthorizedClient client = manager.authorize(req);
        template.header("Authorization", "Bearer " + client.getAccessToken().getTokenValue());
    }
}
```

Регистрируется как `RequestInterceptor` в Feign config.

### 6.4 Пропагация user token

Если сервис вызывается от имени пользователя — переиспользовать его токен:

```java
Jwt jwt = (Jwt) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
String userToken = jwt.getTokenValue();

// добавить в исходящий запрос
```

Пропагация — важно для аудита: downstream видит того же пользователя.

---

## 7. Тестирование

### 7.1 С mock JWT

```java
@Test
@WithMockJwt(subject = "berik", authorities = "CREATE_FNO")
void authorized() {
    // ...
}
```

MockMvc:
```java
mockMvc.perform(post("/api/fno")
    .with(jwt().authorities(new SimpleGrantedAuthority("CREATE_FNO")))
    .content("{...}"))
    .andExpect(status().isOk());
```

### 7.2 Real Keycloak в тестах

`Testcontainers` для Keycloak:
```java
@Container
static KeycloakContainer keycloak = new KeycloakContainer("quay.io/keycloak/keycloak:24.0")
    .withRealmImportFile("test-realm.json");

@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    r.add("spring.security.oauth2.resourceserver.jwt.issuer-uri",
        () -> keycloak.getAuthServerUrl() + "/realms/test");
}
```

Медленнее, но реалистичней.

---

## 8. Кэширование JWKS

Spring по умолчанию кэширует ключи в памяти. Если Keycloak ротирует ключ, а Spring запрашивал JWKS час назад — новые токены упадут.

Настройка:
```java
@Bean
JwtDecoder jwtDecoder(@Value("${issuer}") String issuer) {
    return NimbusJwtDecoder.withIssuerLocation(issuer)
        .cache(Duration.ofMinutes(5))         // сколько кэшировать
        .build();
}
```

Или использовать `restOperations` с retry на JWKS.

---

## 9. Grafik ошибок

### 9.1 401 Unauthorized

- Токен отсутствует.
- Токен истёк.
- Подпись не валидна (не тот ключ).
- `iss` не совпадает.
- `aud` не совпадает (если проверка включена).

### 9.2 403 Forbidden

- Токен валиден, но нет нужной роли/authority.
- `hasRole("admin")` — а в JWT нет.

### 9.3 «NoSuchMethodError» на toString / claim access

Реальный ИСНА-кейс `taxreport21-java21-runtime-regressions`: spring-security-jose и spring-security-core разные версии → skew → NoSuchMethodError.

Фикс: явно закрепить версии через Spring Boot BOM.

### 9.4 JWKS не доступен

Keycloak down / network issue → все запросы падают. Health-check Keycloak критичен.

### 9.5 Разные issuer в токенах

Если несколько realm — надо multi-tenant setup:
```java
@Bean
AuthenticationManagerResolver<HttpServletRequest> authenticationManagerResolver() {
    return request -> {
        String issuer = extractIssuerFromToken(request);
        return authManagerForIssuer(issuer);
    };
}
```

---

## 10. Метрики и логирование

### 10.1 Логировать auth-события

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

### 10.2 Не логировать токены

Никогда не логируй `Authorization` header или `token`. Утечка = компромисс всех пользователей.

Отфильтровать в logback:
```xml
<pattern>%replace(%msg){'Bearer [A-Za-z0-9._-]+', 'Bearer ***'}%n</pattern>
```

---

## 11. Полная прод-конфигурация

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER:https://keycloak.isna/realms/knp}
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
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filter(HttpSecurity http, JwtAuthenticationConverter conv) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**", "/actuator/info", "/actuator/prometheus").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(jwt -> jwt.jwtAuthenticationConverter(conv)))
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
                .accessDeniedHandler((req, resp, e) -> resp.setStatus(403)))
            .headers(h -> h.frameOptions(f -> f.deny()));

        return http.build();
    }

    @Bean
    JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter c = new JwtAuthenticationConverter();
        c.setJwtGrantedAuthoritiesConverter(new KeycloakRealmAndClientRolesConverter("isna-knp-integration"));
        c.setPrincipalClaimName("preferred_username");
        return c;
    }
}
```

---

## 12. Реальные кейсы ИСНА

- `taxreport21-java21-runtime-regressions`: OAuth2 NoSuchMethodError = spring-security-jose vs core version skew. Фикс = BOM.
- `knp-e2e-prod-taxrep21-smoke-local-run`: bearer-канал НЗ падает при миграции на Java 21 (spring-security несовместимость).
- `knp-n07-sent-documents-invisible`: `KNP_PERM_CREATE_NZ_N07` — client-role в Keycloak, permission-гейт `fv.code IN permissionCodes`. Если не выдан юзеру → N07 не видна в отправленных.

---

## 13. Собесные вопросы

1. **Как настроить Spring Boot как Resource Server?** — `spring-boot-starter-oauth2-resource-server` + `spring.security.oauth2.resourceserver.jwt.issuer-uri`.
2. **Как проверить JWT?** — Spring автоматически через discovery + JWKS.
3. **Почему Keycloak-роли не работают в `hasRole`?** — Они в `realm_access.roles` / `resource_access.<client>.roles`, а не в `scope`; нужен кастомный `JwtAuthenticationConverter`.
4. **Как выдать роль в токен?** — Realm Role или Client Role → добавляется маппером в JWT.
5. **Как переиспользовать user token в downstream-запросе?** — Извлечь `jwt.getTokenValue()` из SecurityContext, добавить в исходящий header.
6. **Как получить token для M2M?** — Client Credentials через `OAuth2AuthorizedClientManager`.
7. **Как настроить audiences?** — `spring.security.oauth2.resourceserver.jwt.audiences` или `AudienceValidator`.
8. **Что произойдёт если Keycloak недоступен?** — Все токены upstream не проверятся (JWKS упадёт) → 401 везде.
9. **Как кэшировать JWKS?** — `NimbusJwtDecoder.withIssuerLocation(...).cache(Duration.ofMinutes(5))`.
10. **Разница `hasRole("admin")` и `hasAuthority("admin")`?** — hasRole автопрефиксует ROLE_.
11. **Как мокать JWT в тесте?** — `@WithMockJwt` или `.with(jwt())` в MockMvc.
12. **Adapters Keycloak или Spring Security?** — Adapters deprecated, использовать Spring Security OAuth2 Resource Server.

---

## Итог

- **Resource Server (JWT)** — типичный микросервис.
- **Discovery + JWKS** — auto-конфиг.
- **`JwtAuthenticationConverter`** — маппинг Keycloak-ролей на Spring authorities.
- **`@PreAuthorize`** для тонкой авторизации методов.
- **`OAuth2AuthorizedClientManager`** для Client Credentials M2M.
- **Feign RequestInterceptor** для проброса токенов.
- **Кэшировать JWKS** осознанно.
- **Никогда не логировать токены**.

---

## Итог блока Spring Security / OAuth2 / Keycloak

- 24 — Spring Security основы (фильтры, UserDetails, Method Security).
- 25 — OAuth2 / OIDC теория (роли, grants, JWT, PKCE, discovery).
- 26 — Keycloak (realm, client, roles, mappers, endpoints).
- 27 — Spring Security + Keycloak в проде (Resource Server, JwtAuthenticationConverter, M2M).

Следующий блок — PostgreSQL / HikariCP / highload / Load Balancer (файлы 28-31). Начинаю с `28-postgresql-internals.md`.
