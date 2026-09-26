# 120. Spring Security + Keycloak: интеграция глубоко

## Зачем это знать

Ты пишешь Spring Boot приложение. Хочешь чтобы аутентификация была через Keycloak. Открываешь Stack Overflow — 200 ответов, половина устаревших. Одни советуют `keycloak-spring-boot-adapter`, другие — «adapter deprecated, используйте Spring Security». Пытаешься сделать по последним туториалам — работает базово, но не понятны детали. Почему `@PreAuthorize("hasRole('ADMIN')")` возвращает 403 хотя в токене явно `"realm_access": {"roles": ["ADMIN"]}`. Что такое `SecurityFilterChain` bean и почему `WebSecurityConfigurerAdapter` больше нет. Как правильно extractить роли Keycloak из JWT — по дефолту Spring их не видит. Как настроить CORS для SPA-frontend'а. Как сделать logout который реально терминирует Keycloak session, а не только локальную.

Разница между «работает с адаптером» и «понимаю Spring Security + Keycloak» — способность за минуту ответить: три типа Spring OAuth приложений и когда какой применяется. Что реально делает `SecurityFilterChain` — какие фильтры и в каком порядке. Как автоматически конфигурируется JWT decoder через `issuer-uri`. Что Spring НЕ проверяет по умолчанию (audience!). Как написать custom `JwtAuthenticationConverter` для Keycloak-специфичной extraction ролей. Разница между `@PreAuthorize` и `@Secured`. Как правильно тестировать security с `@WithMockUser` и mock JWT. Как отладить 401/403 в проде — где смотреть, что проверять.

Разберём: три типа Spring OAuth приложений (Client, Resource Server, комбо) — когда какой. Setup Resource Server для API. **Security Filter Chain детально** — каждый фильтр, что делает, в каком порядке. Автоматическая конфигурация через `issuer-uri`. Как Spring валидирует JWT (что проверяется, что НЕТ). Настройка audience validation. **Custom JwtAuthenticationConverter** для правильной extraction Keycloak ролей — обязательно. Method security (`@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, `@PostFilter`). Работа с current user — 4 способа. CORS для SPA. Setup OAuth 2.0 Client для server-side web app. Logout — front-channel и back-channel. Testing security — MockMvc + `@WithMockUser` + JWT mock + Testcontainers Keycloak. Diagnostics и troubleshooting в проде. Migration с deprecated Keycloak adapter.

OAuth/OIDC протокол — файл 117. JWT детально — 118. Keycloak как IDP — 119. Здесь фокус на Spring Security и её интеграции с Keycloak через standards.

## Три типа Spring OAuth приложений

Spring Security различает три типа приложений в OAuth-мире. Понимать который у тебя — фундаментальный первый шаг.

### 1. OAuth 2.0 Client

Приложение которое **инициирует login** пользователя через OAuth provider. Обычно **classic web-app с backend'ом** — сервер рендерит HTML (Thymeleaf, JSP, Freemarker), session-based.

**Что делает Spring**:
- Автоматически делает **Authorization Code flow**.
- Redirect пользователя на Keycloak → login → callback.
- Обмен code на access_token + refresh_token + id_token.
- Создаёт **HTTP session** после успешного логина.
- В контроллерах `@AuthenticationPrincipal OAuth2User` даёт доступ к user info.

**Use case**: классическое web-приложение где backend отвечает HTML, session-based. Admin panel на Thymeleaf. Внутренние enterprise apps.

**Dependency**: `spring-boot-starter-oauth2-client`.

### 2. OAuth 2.0 Resource Server

Приложение которое **принимает JWT** и проверяет его. Не делает login — только validation. Клиент (SPA, mobile, другой сервис) сам получает JWT из Keycloak и присылает.

**Что делает Spring**:
- Извлекает `Authorization: Bearer <JWT>` header из каждого запроса.
- Валидирует JWT (подпись через JWKS, claims).
- Создаёт `JwtAuthenticationToken` в SecurityContext.
- **Stateless** — не создаёт HTTP session.
- В контроллерах `@AuthenticationPrincipal Jwt jwt` даёт прямой доступ к claims.

**Use case**: REST API за SPA/mobile frontend'ом. Микросервисы. GraphQL API. **Наиболее распространённый сценарий современных приложений**.

**Dependency**: `spring-boot-starter-oauth2-resource-server`.

### 3. Комбо

Приложение одновременно и Client, и Resource Server. Backend рендерит HTML (для админ-панели) и предоставляет API (для мобильного приложения). Часть endpoints — session-based (browser), часть — JWT (mobile).

**Dependency**: обе `oauth2-client` + `oauth2-resource-server`. Требует ручной настройки чтобы разделить какой endpoint как обрабатывается.

### Как выбрать

Простое правило по типу приложения:

- **SPA (React/Angular/Vue) + backend API** → backend это **Resource Server**. SPA сама делает Authorization Code + PKCE в Keycloak.
- **Mobile + backend API** → backend это **Resource Server**. Mobile app делает OAuth сама через AppAuth SDK или подобное.
- **Классический server-side rendered web app** (Thymeleaf) → **Client**. Backend делает OAuth flow, session.
- **Микросервис принимающий JWT от gateway** → **Resource Server**.
- **API Gateway перед микросервисами** → часто **Client + Resource Server** (Client для login, Resource Server для проверки JWT от других сервисов).

Для большинства современных API — Resource Server. Разберём его детально в первую очередь.

## Setup Resource Server: API за JWT

Самый распространённый сценарий. API получает `Authorization: Bearer <JWT>` от клиента, проверяет, обрабатывает.

### Минимальная настройка

**Dependency** в `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

**Конфигурация** в `application.yml`:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/myapp
```

**Всё**. Spring Boot автоконфигурация делает:

1. При старте — HTTP GET `https://keycloak.example.com/realms/myapp/.well-known/openid-configuration`.
2. Извлекает `jwks_uri` (например `https://keycloak.example.com/realms/myapp/protocol/openid-connect/certs`).
3. HTTP GET JWKS — скачивает публичные ключи.
4. Кэширует ключи (по умолчанию 5 минут TTL, автоматический refresh при unknown `kid`).
5. Создаёт `NimbusJwtDecoder` bean с validators.
6. Настраивает `BearerTokenAuthenticationFilter` чтобы извлекать JWT из header.

По умолчанию **все endpoints требуют аутентификации**. Нужно добавить `SecurityFilterChain` bean чтобы разрешить public endpoints:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/actuator/health").permitAll()
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()  // CORS preflight
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .csrf(csrf -> csrf.disable())   // stateless API, CSRF не нужен
            .cors(Customizer.withDefaults());
        return http.build();
    }
}
```

Разбор параметров:

- **`authorizeHttpRequests(...)`** — правила авторизации URLs. `.permitAll()` — без аутентификации, `.authenticated()` — требует JWT, `.hasRole("ADMIN")` — требует конкретную роль.
- **`OPTIONS permitAll()`** — CORS preflight requests должны проходить без JWT. Иначе SPA не может сделать preflight, потом настоящий request.
- **`oauth2ResourceServer(...).jwt(...)`** — включить JWT-based auth. Без параметров — использует автоконфигурацию.
- **`sessionCreationPolicy(STATELESS)`** — не создавать HTTP session. Каждый запрос независим (JWT содержит всё что нужно).
- **`csrf.disable()`** — для stateless JWT API CSRF не актуален (нет cookies которые атакующий может использовать).
- **`cors()`** — включить CORS (для запросов от SPA с другого origin).

Всё. API готов принимать `Authorization: Bearer <JWT>`, автоматически валидировать, авторизовать endpoint access.

## Security Filter Chain: что реально происходит

Spring Security работает через **цепочку Servlet Filters** вставленных в HTTP request lifecycle. Понимание порядка критично для отладки — если что-то не работает, важно знать где в цепи проблема.

### Полная цепочка

При каждом HTTP запросе:

```
HTTP Request входит в приложение
     │
     ▼
┌─────────────────────────────────────────┐
│ 1. DisableEncodeUrlFilter                │  disable URL rewriting
├─────────────────────────────────────────┤
│ 2. WebAsyncManagerIntegrationFilter      │  integration с async
├─────────────────────────────────────────┤
│ 3. SecurityContextHolderFilter           │  читает SecurityContext (обычно пустой)
├─────────────────────────────────────────┤
│ 4. HeaderWriterFilter                    │  добавляет security headers в response
├─────────────────────────────────────────┤
│ 5. CorsFilter (если .cors())             │  ⭐ CORS handling
├─────────────────────────────────────────┤
│ 6. CsrfFilter (если не disabled)         │  CSRF token validation
├─────────────────────────────────────────┤
│ 7. LogoutFilter                          │  обработка /logout
├─────────────────────────────────────────┤
│ 8. BearerTokenAuthenticationFilter       │  ⭐⭐⭐ извлекает JWT, валидирует
│    (для Resource Server)                 │      создаёт Authentication
├─────────────────────────────────────────┤
│ 9. RequestCacheAwareFilter               │  восстановление saved request
├─────────────────────────────────────────┤
│ 10. SecurityContextHolderAwareRequestFilter  │  обёртка HttpServletRequest
├─────────────────────────────────────────┤
│ 11. AnonymousAuthenticationFilter        │  если не аутентифицирован — anonymous
├─────────────────────────────────────────┤
│ 12. SessionManagementFilter              │  session management
├─────────────────────────────────────────┤
│ 13. ExceptionTranslationFilter           │  ⭐ ловит security exceptions
│                                          │      → 401 (unauthenticated) 
│                                          │      → 403 (unauthorized)
├─────────────────────────────────────────┤
│ 14. AuthorizationFilter                  │  ⭐⭐ проверяет authorizeHttpRequests rules
└─────────────────────────────────────────┘
     │
     ▼
Controller (если авторизован)
```

Ключевые фильтры выделены. Разберём их подробнее.

### BearerTokenAuthenticationFilter (сердце Resource Server)

Что делает пошагово:

1. Извлекает `Authorization` header.
2. Проверяет что начинается с `Bearer ` (case-sensitive).
3. Достаёт token (всё что после `Bearer `).
4. Вызывает `JwtDecoder.decode(token)`:
   - Парсит JWT (3 части через `.`).
   - Base64URL-декодирует header, парсит JSON.
   - Находит ключ в JWKS по `kid`.
   - Проверяет подпись header'а + payload'а RSA/HMAC-verify.
   - Валидирует claims: `exp`, `iat`, `nbf`, `iss` (соответствует `issuer-uri`).
5. Успех → создаёт **`JwtAuthenticationToken`** (implements `Authentication`).
   - Извлекает `sub` claim → используется как `Authentication.getName()`.
   - Извлекает scopes (`SCOPE_...`) → используется как GrantedAuthorities.
6. Сохраняет `JwtAuthenticationToken` в `SecurityContextHolder` (ThreadLocal current thread).

Failure на любом шаге → `AuthenticationException` → идёт дальше по цепи → `ExceptionTranslationFilter` конвертирует в **401 Unauthorized**.

### AuthorizationFilter (проверка прав)

После аутентификации проверяет: имеет ли пользователь права на этот endpoint (согласно `authorizeHttpRequests(...)` rules).

Псевдокод:

```
requested_url = request.getURI()
authentication = SecurityContextHolder.getAuthentication()

for rule in authorizeHttpRequests_rules:
    if rule.matches(requested_url):
        if rule.check(authentication):
            proceed()  // разрешить
        else:
            throw AccessDeniedException()  // 403
        break
```

Если правило `.authenticated()` и authentication == null (нет валидного JWT) → 401.
Если правило `.hasRole("ADMIN")` и user не имеет `ROLE_ADMIN` authority → 403.

Разница критична для debugging:
- **401 Unauthorized** = не аутентифицирован (нет / плохой JWT).
- **403 Forbidden** = аутентифицирован, но нет прав.

### ExceptionTranslationFilter

Ловит security exceptions от последующих фильтров или контроллеров:

- **`AuthenticationException`** → 401 Unauthorized (+ `WWW-Authenticate: Bearer` header).
- **`AccessDeniedException`** → 403 Forbidden.

Стандартные HTTP-ответы, без stacktrace в теле (безопасность).

### Просмотреть цепочку в debug

Включить debug logging:

```yaml
logging:
  level:
    org.springframework.security: DEBUG
    org.springframework.security.web.FilterChainProxy: TRACE
```

В логах видно каждый filter при каждом запросе:

```
DEBUG FilterChainProxy - Securing GET /api/orders
TRACE FilterChainProxy - Invoking DisableEncodeUrlFilter (1/14)
TRACE FilterChainProxy - Invoking WebAsyncManagerIntegrationFilter (2/14)
...
TRACE FilterChainProxy - Invoking BearerTokenAuthenticationFilter (8/14)
DEBUG BearerTokenAuthenticationFilter - Set SecurityContextHolder to JwtAuthenticationToken
...
TRACE FilterChainProxy - Invoking AuthorizationFilter (14/14)
DEBUG AuthorizationFilter - Authorized filter invocation
```

Полезно для troubleshooting — видно где именно ломается.

## Автоматическая конфигурация: что делает `issuer-uri`

Одна строка `issuer-uri: https://keycloak.example.com/realms/myapp` даёт много магии. Разберём что происходит.

### При старте приложения

1. **HTTP GET discovery endpoint**: `<issuer-uri>/.well-known/openid-configuration`.
2. Парсит JSON ответ, извлекает:
   - `jwks_uri` — URL публичных ключей.
   - `issuer` — canonical issuer URI (для валидации).
   - Прочие URLs (token, authorize, userinfo) — не используются для Resource Server.
3. Создаёт **`NimbusJwtDecoder`** bean:
   - JWK Source из `jwks_uri` (Nimbus библиотека).
   - Validators: signature + `exp` + `iat` + `iss` (equals `issuer-uri`).
4. Регистрирует `BearerTokenAuthenticationFilter` в security filter chain.

### JWKS caching

По умолчанию:
- **Cache TTL**: 5 минут.
- **Cache refresh при unknown `kid`**: принудительный refresh JWKS (максимум раз в 5 минут для предотвращения DoS).
- **Backup ключей**: если Keycloak недоступен в момент refresh — используются старые ключи (пока валидны).

Практический эффект: при **key rotation** в Keycloak (обычно раз в 1-2 года):

1. Keycloak начинает подписывать новые JWT новым ключом (`kid: def456`).
2. Старые JWT (подписанные `kid: abc123`) продолжают быть валидны — старый ключ ещё в JWKS.
3. Spring получает первый JWT с `kid: def456` — не находит в кэше → принудительный refresh JWKS → находит новый ключ → проверяет успешно.
4. Дальше — оба ключа в кэше, оба типа JWT валидируются.
5. Через время старые JWT истекут, Keycloak уберёт старый ключ из JWKS, при следующем refresh Spring обновит.

Клиент никогда не видит проблем — rotation прозрачен.

### Что если Keycloak недоступен при старте

Проблема: `issuer-uri` требует HTTP GET к Keycloak при старте Spring. Если Keycloak недоступен — Spring не стартует, `BeanCreationException`.

**Fix для startup resilience**:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://keycloak.example.com/realms/myapp/protocol/openid-connect/certs
          issuer-uri: https://keycloak.example.com/realms/myapp
```

С `jwk-set-uri` явно — Spring не делает discovery при старте, только при первом JWT. Но также не проверяет соответствие `iss` claim с `issuer-uri` автоматически. Нужно вручную:

```java
@Bean
JwtDecoder jwtDecoder(@Value("${jwt.issuer-uri}") String issuer,
                     @Value("${jwt.jwk-set-uri}") String jwkSetUri) {
    NimbusJwtDecoder decoder = NimbusJwtDecoder.withJwkSetUri(jwkSetUri).build();
    decoder.setJwtValidator(JwtValidators.createDefaultWithIssuer(issuer));
    return decoder;
}
```

Или использовать `spring.cloud.discovery` для service discovery.

## Как Spring валидирует JWT: детально

По умолчанию `JwtDecoder` проверяет:

**1. Формат** — 3 части, base64URL-декодируется, валидный JSON.

**2. Header**:
- `alg` — из configured (обычно RS256 по умолчанию, извлекается из JWKS).
- `kid` — известен в JWKS.

**3. Подпись** — RSA-verify с публичным ключом из JWKS.

**4. Claims**:
- **`iss`** — точно совпадает с `issuer-uri` (строковое сравнение).
- **`exp`** — не в прошлом (с clock skew 30 секунд).
- **`iat`** — не в будущем (с clock skew).
- **`nbf`** (если есть) — уже прошёл.

### Что НЕ проверяется по умолчанию

**`aud`** — audience! Spring **не проверяет** что токен для этого приложения. Значит любой валидный JWT от того же Keycloak realm может пройти проверку, даже если выдан для другого клиента.

**Реальная проблема**. У Keycloak по умолчанию `aud: account`. Твой API `orders-api` получает эти токены — если не проверяешь aud, работает. Но токены выданные для **любого другого клиента** того же realm тоже проходят (у них тоже `aud: account`). Возможна privilege escalation — токен `reports-api` работает в `orders-api`.

### Настройка audience validation

Ручная настройка обязательна:

```java
@Configuration
public class JwtConfig {
    
    @Value("${spring.security.oauth2.resourceserver.jwt.issuer-uri}")
    private String issuerUri;
    
    @Value("${app.expected-audience:orders-api}")
    private String expectedAudience;
    
    @Bean
    JwtDecoder jwtDecoder() {
        NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuerUri);
        
        // Дефолтные validators (iss, exp, iat) + custom для aud
        OAuth2TokenValidator<Jwt> defaultValidators = 
            JwtValidators.createDefaultWithIssuer(issuerUri);
        
        OAuth2TokenValidator<Jwt> audienceValidator = token -> {
            List<String> audiences = token.getAudience();
            if (audiences != null && audiences.contains(expectedAudience)) {
                return OAuth2TokenValidatorResult.success();
            }
            return OAuth2TokenValidatorResult.failure(
                new OAuth2Error("invalid_token", 
                    "Missing required audience: " + expectedAudience, null));
        };
        
        decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
            defaultValidators, audienceValidator));
        
        return decoder;
    }
}
```

Также нужно в Keycloak настроить **Audience Mapper** в клиенте — добавить `aud: orders-api` в токены выданные для этого клиента. Иначе валидация будет всегда фейлиться.

## Custom JwtAuthenticationConverter: правильная extraction ролей Keycloak

**Проблема по умолчанию**. Spring Security ожидает роли в claim `scope` (стандарт OAuth) или как `authorities`. Keycloak кладёт роли **по-другому**:

- **Realm roles** → `realm_access.roles: ["ADMIN", "USER"]`
- **Client roles** → `resource_access.<client_id>.roles: ["orders:read"]`

Без custom converter Spring **не видит** этих ролей. `@PreAuthorize("hasRole('ADMIN')")` не сработает даже если в токене явно `"realm_access": {"roles": ["ADMIN"]}`.

### Реализация custom converter

```java
@Configuration
public class SecurityConfig {
    
    @Value("${app.keycloak.client-id:orders-api}")
    private String clientId;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(keycloakConverter()))
            );
        return http.build();
    }
    
    private JwtAuthenticationConverter keycloakConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(this::extractKeycloakAuthorities);
        return converter;
    }
    
    @SuppressWarnings("unchecked")
    private Collection<GrantedAuthority> extractKeycloakAuthorities(Jwt jwt) {
        Collection<GrantedAuthority> authorities = new ArrayList<>();
        
        // 1. Realm roles → ROLE_XXX (для hasRole)
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess != null) {
            List<String> realmRoles = (List<String>) realmAccess.get("roles");
            if (realmRoles != null) {
                realmRoles.forEach(role -> 
                    authorities.add(new SimpleGrantedAuthority("ROLE_" + role))
                );
            }
        }
        
        // 2. Client roles → ROLE_XXX (для нашего конкретного client)
        Map<String, Object> resourceAccess = jwt.getClaim("resource_access");
        if (resourceAccess != null) {
            Map<String, Object> clientAccess = 
                (Map<String, Object>) resourceAccess.get(clientId);
            if (clientAccess != null) {
                List<String> clientRoles = (List<String>) clientAccess.get("roles");
                if (clientRoles != null) {
                    clientRoles.forEach(role -> 
                        authorities.add(new SimpleGrantedAuthority("ROLE_" + role))
                    );
                }
            }
        }
        
        // 3. Scopes → SCOPE_XXX (для hasAuthority)
        String scope = jwt.getClaimAsString("scope");
        if (scope != null && !scope.isBlank()) {
            Arrays.stream(scope.split(" "))
                .forEach(s -> authorities.add(new SimpleGrantedAuthority("SCOPE_" + s)));
        }
        
        return authorities;
    }
}
```

Теперь работает:
- `@PreAuthorize("hasRole('ADMIN')")` — проверяет наличие `ROLE_ADMIN` authority.
- `@PreAuthorize("hasAuthority('SCOPE_orders:read')")` — проверяет scope.

### Convention Spring Security

Важно понимать префиксы:

- **`ROLE_` prefix** — используется для ролей. `hasRole('ADMIN')` эквивалентно `hasAuthority('ROLE_ADMIN')`. Spring автоматически добавляет `ROLE_` при `hasRole()`.
- **`SCOPE_` prefix** — используется для OAuth scopes. `hasAuthority('SCOPE_orders:read')`.

Наш converter добавляет `ROLE_` prefix при mapping ролей — это позволяет использовать удобный `hasRole('ADMIN')` вместо `hasAuthority('ROLE_ADMIN')`.

## Method Security: @PreAuthorize и другие

Endpoint-level authorization через `authorizeHttpRequests(...)` — грубо (по URL patterns). Method-level authorization — точнее (per method). Обычно используются оба.

### Включение

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig { ... }
```

Начиная с Spring Security 6 — `@EnableMethodSecurity` (раньше `@EnableGlobalMethodSecurity`).

### @PreAuthorize — проверка ДО вызова метода

Самый распространённый. Проверка выполняется перед входом в метод:

```java
@Service
public class OrderService {
    
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long id) {
        // Выполнится только если user имеет ROLE_ADMIN
    }
    
    @PreAuthorize("hasAuthority('SCOPE_orders:write')")
    public Order createOrder(OrderRequest req) {
        // Требует scope orders:write в токене
    }
    
    // SpEL expressions с параметрами метода и Authentication
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.name")
    public User getUser(Long userId) {
        // ADMIN может любого пользователя, обычный — только себя
    }
    
    // Комбинация условий
    @PreAuthorize("hasRole('MANAGER') and hasAuthority('SCOPE_reports:read')")
    public Report generateReport() { ... }
    
    // Обращение к beans через @beanName
    @PreAuthorize("@customAuth.canAccessOrder(#orderId, authentication)")
    public Order getOrder(Long orderId) { ... }
}
```

При fail — `AccessDeniedException` → 403 Forbidden.

### @PostAuthorize — проверка ПОСЛЕ вызова

Метод выполнится, но результат не отдастся клиенту если expression false:

```java
@PostAuthorize("returnObject.owner == authentication.name or hasRole('ADMIN')")
public Document getDocument(Long id) {
    return documentRepo.findById(id).orElseThrow();
    // Документ вернётся только если owner совпадает с current user 
    // или user — ADMIN
}
```

`returnObject` — специальная переменная, ссылается на возвращаемое значение.

Полезно когда permission зависит от загруженного объекта — до загрузки не знаем можно ли user'у видеть.

Минус — метод **уже выполнился**, если тяжёлый (SQL, external calls) — работа сделана впустую. `@PreAuthorize` предпочтительнее если permission можно проверить до.

### @PreFilter / @PostFilter — фильтрация коллекций

Фильтрует элементы **до** или **после** вызова:

```java
@PostFilter("filterObject.owner == authentication.name")
public List<Document> listAll() {
    return documentRepo.findAll();
    // Вернётся только те документы, где owner == current user
}
```

`filterObject` — специальная переменная, ссылается на текущий элемент коллекции при итерации.

Работает через iteration + remove. **Медленно на больших коллекциях** — O(n) с SpEL evaluation каждого элемента. Лучше фильтровать в SQL:

```java
public List<Document> listMine(String username) {
    return documentRepo.findByOwner(username);
}
```

`@PostFilter` — для small collections или прототипов.

### @Secured — legacy

```java
@Secured("ROLE_ADMIN")
public void adminOperation() { }
```

Устаревающий, только с role names, без SpEL. Использовать `@PreAuthorize` вместо.

### Method security НЕ работает через self-invocation

Классический питфол — Spring proxy detection:

```java
@Service
class OrderService {
    
    public void placeOrder() {
        // Проверка НЕ применится — self-invocation
        deleteOrder(1L);
    }
    
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long id) { }
}
```

Прямой вызов `deleteOrder(1L)` через `this` минует Spring proxy → `@PreAuthorize` игнорируется.

Fix: вызывать через другой bean, или self-injection, или AspectJ. См. файл 85 про Facade/Proxy.

## Работа с current user

Получить текущего аутентифицированного пользователя — несколько способов.

### 1. SecurityContextHolder (везде в коде)

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();   // subject из JWT
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
```

Работает в любом коде — сервисах, репозиториях, utilities. ThreadLocal-based.

### 2. Principal параметр (Servlet API)

```java
@GetMapping("/me")
public String currentUser(Principal principal) {
    return principal.getName();   // = sub из JWT
}
```

Стандартный Servlet API. Работает без Spring Security.

### 3. Authentication параметр (Spring Security)

```java
@GetMapping("/me")
public UserInfo currentUser(Authentication auth) {
    Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
    boolean isAdmin = authorities.stream()
        .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
    return new UserInfo(auth.getName(), isAdmin);
}
```

Больше информации чем Principal.

### 4. @AuthenticationPrincipal Jwt — типизированный доступ

```java
@GetMapping("/me")
public UserInfo currentUser(@AuthenticationPrincipal Jwt jwt) {
    String userId = jwt.getSubject();
    String email = jwt.getClaimAsString("email");
    String name = jwt.getClaimAsString("name");
    List<String> roles = jwt.getClaimAsStringList("realm_access.roles");
    
    // Прямой доступ ко всем claims JWT
    Map<String, Object> customClaims = jwt.getClaims();
    
    return new UserInfo(userId, email, name, roles);
}
```

**Рекомендуется** — прямой доступ к payload JWT, типизированно, чисто.

### 5. Custom UserPrincipal для business-специфичного user

```java
public class AppUser {
    private final String userId;
    private final String email;
    private final Set<String> roles;
    private final String tenantId;
    private final String department;
    
    public AppUser(Jwt jwt) {
        this.userId = jwt.getSubject();
        this.email = jwt.getClaimAsString("email");
        this.tenantId = jwt.getClaimAsString("tenant_id");
        this.department = jwt.getClaimAsString("department");
        
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        this.roles = realmAccess != null 
            ? new HashSet<>((List<String>) realmAccess.get("roles"))
            : Collections.emptySet();
    }
    
    // getters
    public boolean isAdmin() { return roles.contains("ADMIN"); }
    public boolean canAccessTenant(String targetTenant) { 
        return isAdmin() || tenantId.equals(targetTenant); 
    }
}

// Custom converter создающий AppUser
@Configuration
class SecurityConfig {
    @Bean
    JwtAuthenticationConverter jwtAuthenticationConverter() {
        return new JwtAuthenticationConverter() {
            @Override
            protected AbstractAuthenticationToken convert(Jwt jwt) {
                AppUser appUser = new AppUser(jwt);
                Collection<GrantedAuthority> authorities = extractAuthorities(jwt);
                return new JwtAuthenticationToken(jwt, authorities, appUser.getUserId());
            }
        };
    }
}

// В контроллере
@GetMapping("/orders")
public List<Order> getOrders(@AuthenticationPrincipal AppUser user) {
    if (user.isAdmin()) {
        return orderService.findAll();
    }
    return orderService.findByTenant(user.getTenantId());
}
```

Чище чем работать с `Jwt` напрямую в каждом контроллере — business-логика в одном месте.

## CORS для SPA frontend

SPA (React) на `https://myspa.com` делает запросы к API на `https://api.myapp.com`. **Разные origin'ы** → browser блокирует cross-origin requests без CORS headers.

### Настройка CORS

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    
    // Кто может делать запросы (не *)
    config.setAllowedOrigins(List.of(
        "https://myspa.com",
        "https://admin.myapp.com"
    ));
    
    // Какие HTTP методы
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"));
    
    // Какие headers клиент может отправлять
    config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
    
    // Разрешить credentials (cookies, Authorization header)
    config.setAllowCredentials(true);
    
    // Cache preflight response (сколько браузер может не спрашивать снова)
    config.setMaxAge(3600L);
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

В `SecurityFilterChain`:

```java
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

### Preflight OPTIONS

Browser автоматически шлёт `OPTIONS` перед `POST/PUT/DELETE/PATCH` для проверки CORS. Spring Security по умолчанию **блокирует** OPTIONS (требует auth). Fix — allow OPTIONS в security config:

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
    .anyRequest().authenticated()
);
```

Или полагаться на то что `http.cors(...)` вставляет CorsFilter перед security filters — тогда CORS обрабатывается до проверки auth, OPTIONS автоматически проходят.

### Тонкость `setAllowCredentials(true)` + `setAllowedOrigins("*")`

**Не работает вместе** — spec запрещает. Если нужны credentials (cookies, Auth header) — origins должны быть конкретными URLs, не wildcard.

Для dev с меняющимися origins — `setAllowedOriginPatterns("http://localhost:*")` — patterns вместо exact match.

## Setup OAuth 2.0 Client: server-side web app

Для классических server-side web apps (Thymeleaf, JSP, Freemarker) с backend'ом делающим OAuth flow.

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

### Конфигурация

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: my-web-app
            client-secret: my-secret
            scope: openid,profile,email
            authorization-grant-type: authorization_code
            redirect-uri: '{baseUrl}/login/oauth2/code/{registrationId}'
        provider:
          keycloak:
            issuer-uri: https://keycloak.example.com/realms/myapp
            user-name-attribute: preferred_username
```

`registration.keycloak` — клиент. `provider.keycloak` — сам Keycloak (через discovery).

`{baseUrl}` и `{registrationId}` — placeholders Spring. Автоматически заменяются на URL приложения и `keycloak` соответственно.

### SecurityFilterChain

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/", "/public/**", "/webjars/**").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2Login(Customizer.withDefaults())
        .logout(logout -> logout
            .logoutSuccessUrl("/")
            .invalidateHttpSession(true)
        );
    return http.build();
}
```

### Что происходит при первом посещении защищённого URL

1. User заходит на `/dashboard`. Spring Security видит — не аутентифицирован.
2. **Redirect на `/oauth2/authorization/keycloak`** — endpoint Spring Security который инициирует OAuth flow.
3. Spring Security делает redirect на Keycloak `/protocol/openid-connect/auth` с параметрами:
   - `client_id=my-web-app`
   - `redirect_uri=https://myapp.com/login/oauth2/code/keycloak`
   - `scope=openid profile email`
   - `state=RANDOM` (для CSRF защиты)
4. User логинится в Keycloak.
5. Keycloak делает redirect на `redirect-uri` с параметром `code=xxx`.
6. Spring Security обрабатывает callback:
   - Обменивает `code` на `access_token` + `id_token` через token endpoint Keycloak (backend call).
   - Валидирует id_token.
   - Создаёт `OAuth2AuthenticationToken` в SecurityContext.
   - Создаёт **HTTP session** (в отличие от stateless Resource Server).
   - Redirect на originally requested URL (`/dashboard`).
7. User видит защищённую страницу.

Дальнейшие запросы — cookies session (`JSESSIONID`), Spring читает из session `OAuth2AuthenticationToken`.

### В контроллере

```java
@GetMapping("/dashboard")
public String dashboard(@AuthenticationPrincipal OAuth2User user, Model model) {
    model.addAttribute("username", user.getAttribute("preferred_username"));
    model.addAttribute("email", user.getAttribute("email"));
    model.addAttribute("name", user.getAttribute("name"));
    return "dashboard";
}
```

`OAuth2User` содержит все claims из id_token + userinfo response.

## Logout: front-channel и back-channel

Logout в OAuth сложнее чем в session-based приложениях.

### Front-channel logout (стандарт для web app)

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.logout(logout -> logout
        .logoutSuccessHandler(oidcLogoutSuccessHandler())
    );
    return http.build();
}

@Bean
public LogoutSuccessHandler oidcLogoutSuccessHandler() {
    OidcClientInitiatedLogoutSuccessHandler handler = 
        new OidcClientInitiatedLogoutSuccessHandler(clientRegistrationRepository);
    handler.setPostLogoutRedirectUri("http://localhost:8080/");
    return handler;
}
```

Поведение при `POST /logout`:
1. Spring Security invalidates HTTP session — user не аутентифицирован в приложении.
2. Redirect на Keycloak logout endpoint с параметрами:
   - `id_token_hint=<user's id_token>`.
   - `post_logout_redirect_uri=http://localhost:8080/`.
3. Keycloak терминирует **SSO session** (все связанные клиенты).
4. Redirect обратно на приложение (post_logout_redirect_uri).

Результат — user разлогинен и в приложении, и в Keycloak. При следующем заходе — полный login flow снова.

### Back-channel logout (Spring 6.2+)

Keycloak уведомляет клиентов о logout **напрямую**, без browser. Полезно когда:
- User logout из одного приложения → надо разлогинить во всех.
- Admin force-logout user'а через Admin Console.

Настройка в Keycloak client: **Backchannel Logout URL**: `https://myapp.com/logout/back-channel/openid-connect`.

Настройка в Spring Security 6.2+:

```java
http.oidcLogout(oidc -> oidc.backChannel(Customizer.withDefaults()));
```

Spring Security обрабатывает Keycloak notification, инвалидирует локальную session пользователя.

### Logout в Resource Server (JWT stateless)

Классического «logout» нет — JWT stateless, сервер не хранит session.

Опции:
- **Client удаляет токены локально** — просто, но украденный токен всё ещё валиден до exp.
- **Revoke refresh_token в Keycloak** — клиент вызывает `/protocol/openid-connect/logout` endpoint с `refresh_token`. Keycloak revoke — refresh перестаёт работать. Access token — валиден до exp (обычно короткое TTL).
- **Blacklist access_token** — стore в Redis (см. файл 118).

## Testing security

Тесты security с реальным Keycloak — сложно, медленно. Используем mocks.

### MockMvc + @WithMockUser

Простые тесты без реального JWT:

```java
@SpringBootTest
@AutoConfigureMockMvc
public class OrderControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanDelete() throws Exception {
        mockMvc.perform(delete("/api/orders/123"))
            .andExpect(status().isNoContent());
    }
    
    @Test
    @WithMockUser(roles = "USER")
    void userCannotDelete() throws Exception {
        mockMvc.perform(delete("/api/orders/123"))
            .andExpect(status().isForbidden());
    }
    
    @Test
    void anonymousCannotAccess() throws Exception {
        mockMvc.perform(get("/api/orders"))
            .andExpect(status().isUnauthorized());
    }
}
```

`@WithMockUser(roles = "ADMIN")` — создаёт mock authentication с `ROLE_ADMIN`. Не реальный JWT, но достаточно для тестов authorization logic.

### JWT mock для более реалистичных тестов

```java
@Test
void withRealJwtStructure() throws Exception {
    Jwt jwt = Jwt.withTokenValue("mock-token")
        .header("alg", "RS256")
        .claim("sub", "user-123")
        .claim("preferred_username", "ivan")
        .claim("email", "ivan@example.com")
        .claim("realm_access", Map.of("roles", List.of("ADMIN")))
        .claim("scope", "openid profile")
        .build();
    
    mockMvc.perform(get("/api/orders")
            .with(jwt().jwt(jwt).authorities(
                new SimpleGrantedAuthority("ROLE_ADMIN"),
                new SimpleGrantedAuthority("SCOPE_openid")
            )))
        .andExpect(status().isOk());
}
```

`jwt()` — post-processor из `SecurityMockMvcRequestPostProcessors`. Создаёт JWT в SecurityContext без реального Keycloak.

### Testcontainers Keycloak

Для integration тестов с реальным Keycloak:

```java
@Testcontainers
@SpringBootTest
public class KeycloakIntegrationTest {
    
    @Container
    static KeycloakContainer keycloak = new KeycloakContainer()
        .withRealmImportFile("test-realm.json");
    
    @DynamicPropertySource
    static void registerKeycloakProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.security.oauth2.resourceserver.jwt.issuer-uri",
            () -> keycloak.getAuthServerUrl() + "/realms/test");
    }
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void realJwtFromKeycloak() throws Exception {
        // Получить реальный JWT из Keycloak
        String token = getAccessToken("test-user", "password");
        
        mockMvc.perform(get("/api/orders")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }
    
    private String getAccessToken(String username, String password) {
        // HTTP POST к Keycloak /token endpoint
        // ...
    }
}
```

Медленнее (запуск Keycloak container ~30 сек первый раз), но покрывает реальную интеграцию — realm settings, mappers, JWKS.

Обычная практика: **90% тестов** — mock JWT (быстро), **10% integration** — Testcontainers Keycloak (полное покрытие).

## Диагностика в проде: 401 vs 403

**Симптом «401 Unauthorized»** — не аутентифицирован. Проверить:

1. **Header формат**: точно `Authorization: Bearer <token>` (не `bearer`, не `<token>` без Bearer, регистр важен).
2. **Token валиден**: декодировать на jwt.io, проверить `exp` не в прошлом, `iat` не в будущем.
3. **Signature**: правильный ключ? JWKS Keycloak доступен из приложения? Firewall не блокирует?
4. **`iss` claim**: точно совпадает с `issuer-uri` в конфиге приложения? (Особенно `http` vs `https`, trailing slash).
5. **`aud` claim** (если настроен audience validator): содержит expected audience?
6. **Логи Spring Security**: включить `logging.level.org.springframework.security=DEBUG`, увидеть какой шаг failed.

**Симптом «403 Forbidden»** — аутентификация прошла, но авторизация нет. Проверить:

1. **Какие authorities реально в SecurityContext**:
   ```java
   @GetMapping("/debug/authorities")
   public Collection<? extends GrantedAuthority> authorities(Authentication auth) {
       return auth.getAuthorities();
   }
   ```
2. **Роли Keycloak доходят до Spring**? Правильный `JwtAuthenticationConverter` настроен? Client роли извлекаются для правильного `clientId`?
3. **@PreAuthorize expression правильный синтаксис**? `hasRole('ADMIN')` требует authority `ROLE_ADMIN` (с префиксом).
4. **URL patterns в `authorizeHttpRequests` правильные matchers**? Порядок matter — первый match применяется.

**Симптом «работает локально, не работает в prod»** — обычно:

1. **`issuer-uri` не тот** — dev vs prod Keycloak.
2. **HTTPS mismatch** — Keycloak на `https://`, `iss` в токене с `https://`, а Spring считает разные строки чем `http://`.
3. **Prod Keycloak за прокси** — `iss` может быть внутренний URL, недоступный из приложения. Настроить `KC_HOSTNAME`, `KC_HOSTNAME_URL` в Keycloak.
4. **Firewall**: приложение не может достучаться до Keycloak JWKS endpoint. Test: `curl -v https://keycloak/.../certs` из pod'а.
5. **Clock skew** — время серверов сильно расходится → exp / iat ошибки. Настроить NTP.

### Debug logging

```yaml
logging:
  level:
    org.springframework.security: DEBUG
    org.springframework.security.oauth2: DEBUG
    org.springframework.security.web.FilterChainProxy: TRACE
    org.springframework.web: DEBUG
```

В логах видно каждый шаг: filter chain, JWT parsing, validation errors, authority extraction, authorization decisions.

### Diagnostic endpoint

Полезно иметь endpoint отдающий текущее security state:

```java
@RestController
public class DebugController {
    
    @GetMapping("/debug/whoami")
    @PreAuthorize("isAuthenticated()")
    public Map<String, Object> whoami(@AuthenticationPrincipal Jwt jwt) {
        return Map.of(
            "subject", jwt.getSubject(),
            "issuer", jwt.getIssuer().toString(),
            "audience", jwt.getAudience(),
            "expires", jwt.getExpiresAt(),
            "issued", jwt.getIssuedAt(),
            "claims", jwt.getClaims(),
            "authorities", SecurityContextHolder.getContext()
                .getAuthentication().getAuthorities()
        );
    }
}
```

В prod закрыть за admin ролью или отключить (не показывать в public).

## Migration с deprecated Keycloak adapter

Раньше был официальный `keycloak-spring-boot-adapter`. С Keycloak 21+ и Spring Boot 3+ — **deprecated**. Правильный путь — Spring Security native OAuth support.

### Что менять

**Dependencies** — убрать:
```xml
<!-- OLD -->
<dependency>
    <groupId>org.keycloak</groupId>
    <artifactId>keycloak-spring-boot-starter</artifactId>
</dependency>
```

Добавить:
```xml
<!-- NEW -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

**Конфигурация** — убрать:
```yaml
# OLD
keycloak:
  auth-server-url: https://keycloak.example.com
  realm: myapp
  resource: my-client
  bearer-only: true
```

Добавить:
```yaml
# NEW
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/myapp
```

**Security config** — убрать `extends KeycloakWebSecurityConfigurerAdapter`, использовать современный `SecurityFilterChain` bean.

**Extraction ролей** — раньше Keycloak adapter автоматически понимал `realm_access.roles`. Теперь надо custom `JwtAuthenticationConverter` (см. выше).

### Плюсы миграции

- **Стандарт** — OAuth spec, не vendor-specific адаптер.
- **Активно поддерживается** — Spring Security обновляется каждый релиз. Keycloak adapter — не развивается.
- **Работает с любым OIDC-совместимым IDP** — Auth0, Okta, Google, Azure AD, не только Keycloak.
- **Меньше зависимостей** — не тащит Keycloak-специфичный код.
- **Лучше в K8s / cloud native** — стандартные подходы работают предсказуемо.

## Заключение

**Три типа Spring OAuth приложений**:
- **OAuth 2.0 Client** — инициирует login (Authorization Code flow), session-based, для server-side web apps.
- **Resource Server** — принимает JWT, stateless, для API (SPA/mobile frontend).
- **Комбо** — оба вместе для приложений с mixed workload.

Современные API — обычно **Resource Server**.

**Setup Resource Server** — один starter (`spring-boot-starter-oauth2-resource-server`) + одна строка `spring.security.oauth2.resourceserver.jwt.issuer-uri`. Spring автоматически: читает discovery, скачивает JWKS с кэшированием (5 мин TTL + refresh на unknown kid), настраивает `JwtDecoder` с validators (signature + iss + exp + iat).

**Security Filter Chain** — цепочка Servlet filters в конкретном порядке. Ключевые:
- **CorsFilter** — CORS handling.
- **BearerTokenAuthenticationFilter** — извлекает JWT из `Authorization: Bearer`, валидирует, создаёт `JwtAuthenticationToken` в SecurityContext.
- **ExceptionTranslationFilter** — переводит security exceptions в 401/403.
- **AuthorizationFilter** — проверяет `authorizeHttpRequests` rules.

**Что Spring НЕ проверяет по умолчанию** — `aud` (audience)! Обязательно настроить в prod через custom `JwtValidator`. Иначе токен для другого клиента того же realm пройдёт validation.

**Custom JwtAuthenticationConverter обязателен для Keycloak** — стандартный не понимает `realm_access.roles` и `resource_access.<client>.roles`. Реализация: извлекать realm roles + client roles + scopes, mapping в GrantedAuthority с префиксом `ROLE_` (для hasRole) и `SCOPE_` (для hasAuthority).

**Method Security** через `@EnableMethodSecurity` + `@PreAuthorize` (до вызова) / `@PostAuthorize` (после, с returnObject) / `@PreFilter` / `@PostFilter` (фильтрация коллекций). SpEL expressions с доступом к параметрам метода и Authentication. `@Secured` — legacy.

**Не работает через self-invocation** — Spring proxy detection. Fix — self-injection, разделение на 2 bean, или AspectJ.

**Current user** — 5 способов: `SecurityContextHolder`, `Principal`, `Authentication`, **`@AuthenticationPrincipal Jwt jwt`** (рекомендуется — типизированный доступ), custom `AppUser` через custom converter (для business-специфичных полей).

**CORS для SPA** — настроить `CorsConfigurationSource` (allowedOrigins конкретные, не `*`; allowCredentials true), включить в `HttpSecurity.cors()`. Preflight OPTIONS обычно permitAll или полагаться на CorsFilter перед security filters.

**OAuth 2.0 Client** для server-side web app — starter `oauth2-client`, конфиг с client-id/secret/scope/redirect-uri. Автоматический Authorization Code flow, session-based, `@AuthenticationPrincipal OAuth2User` в контроллерах.

**Logout**: 
- Front-channel через `OidcClientInitiatedLogoutSuccessHandler` — приложение redirect'ит на Keycloak logout, тот терминирует SSO session, redirect обратно.
- Back-channel (Spring 6.2+) — Keycloak уведомляет приложения о logout через backchannel URL, приложения инвалидируют local sessions.
- В Resource Server (JWT) — client удаляет токены + revoke refresh_token в Keycloak.

**Testing**: `@WithMockUser(roles=...)` для простых тестов (без реального JWT), `jwt()` postprocessor для реалистичных JWT mock, Testcontainers Keycloak для integration tests. Практика: 90% mock, 10% integration.

**Диагностика в проде**:
- **401** → JWT валиден? iss совпадает? JWKS доступен? Header формат?
- **403** → правильные authorities? Custom converter работает? Client роли для правильного clientId?
- **Debug logging** `org.springframework.security=DEBUG` показывает каждый шаг.
- **Diagnostic endpoint** `/debug/whoami` для быстрой проверки claims + authorities.
- **Работает локально не работает в prod** — обычно issuer-uri mismatch, HTTPS vs HTTP, clock skew, firewall.

**Migration с Keycloak adapter**: убрать `keycloak-spring-boot-adapter`, добавить `spring-boot-starter-oauth2-resource-server`. Заменить конфиг на `issuer-uri`. Custom `JwtAuthenticationConverter` для ролей. Плюсы — стандарт, active support, работает с любым OIDC IDP, не только Keycloak.

OAuth/OIDC протокол — файл 117. JWT детально — 118. Keycloak как IDP — 119. Здесь была глубина по Spring Security + Keycloak integration через standards: три типа приложений, filter chain internals, custom converter обязательность, method security, current user access, CORS, logout варианты, testing patterns, real-world diagnostics.
