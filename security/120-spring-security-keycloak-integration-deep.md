# 120. Spring Security + Keycloak: интеграция глубоко

## Зачем это знать

Ты пишешь Spring Boot приложение — API или веб-сервис. Хочешь чтобы аутентификация была через Keycloak. Как? Раньше был официальный `keycloak-spring-boot-adapter` — deprecated с Spring Boot 3 / Keycloak 21+. Сейчас правильный способ — **Spring Security 6+ OAuth2 support** (полностью встроен, без Keycloak adapter'а). Работает через стандарты OAuth/OIDC — Keycloak воспринимается просто как OIDC-совместимый provider, ничего специфичного.

Разница между «добавил `keycloak-spring-boot` в pom.xml и работает» и «понимаю Spring Security + Keycloak» — способность за минуту ответить: какие типы приложений бывают (OAuth Client vs Resource Server vs комбо) и когда какой. Как выглядит security filter chain и где Spring вставляет JWT validation. Как правильно извлечь роли Keycloak из JWT (по дефолту Spring НЕ понимает `realm_access.roles`). Как настроить method security через `@PreAuthorize`. Как сделать logout который реально терминирует Keycloak session. Как отладить когда `403 Forbidden` — где смотреть, что проверять.

Разберём: три типа Spring приложений с OAuth (Client, Resource Server, комбо). Setup для каждого. Security Filter Chain — детально. Автоматическое конфигурирование через `issuer-uri`. Как Spring валидирует JWT (JWKS caching, claims validation). Custom `JwtAuthenticationConverter` для правильной extraction ролей Keycloak. Method security (`@PreAuthorize`, `@PostAuthorize`, `@Secured`). Работа с current user (`Authentication`, `Principal`, `@AuthenticationPrincipal Jwt`). CORS настройка для SPA-frontend. Logout — front-channel и back-channel. Session management в Client приложениях. Testing security (MockMvc + JWT mock). Diagnostics и troubleshooting в проде. Migration с deprecated Keycloak adapter на Spring Security native.

OAuth/OIDC протокол — файл 117. JWT детально — 118. Keycloak как IDP — 119. Здесь фокус на Spring Security и её интеграции с Keycloak.

## Три типа Spring OAuth приложений

Spring Security различает три типа приложений в OAuth-мире:

**1. OAuth 2.0 Client (`spring-boot-starter-oauth2-client`)** — приложение которое **инициирует login** пользователя через OAuth provider. Обычно web-app с backend'ом (Server-Side Rendering — Thymeleaf, JSP). Spring делает Authorization Code flow автоматически: redirect на Keycloak, callback handling, session management.

Use case: классическое web-приложение где backend рендерит HTML, session-based.

**2. OAuth 2.0 Resource Server (`spring-boot-starter-oauth2-resource-server`)** — API который **принимает JWT** и проверяет его. Не делает login, только validation. Клиент (SPA / mobile / другой сервис) сам получает JWT из Keycloak и присылает.

Use case: REST API за SPA/mobile frontend'ом. Микросервисы. GraphQL API.

**3. Комбо** — и Client, и Resource Server в одном. Backend рендерит HTML и предоставляет API. Часть endpoints — для session-based (browser), часть — для JWT (mobile).

**Как выбрать**:

- SPA (React/Angular/Vue) + backend API → backend это **Resource Server**. SPA делает Authorization Code + PKCE сама.
- Mobile + backend API → **Resource Server**. Mobile app делает OAuth сама через AppAuth SDK или подобное.
- Классическое server-side rendered web app (Thymeleaf) → **Client**. Backend делает OAuth flow, session.
- Microservice принимающий JWT от gateway → **Resource Server**.
- Gateway перед microservices → **Client + Resource Server**.

Для большинства современных API — Resource Server. Разберём его детально.

## Setup: Resource Server (API за JWT)

Самый распространённый сценарий. API получает `Authorization: Bearer <JWT>` от клиента (SPA/mobile), проверяет, обрабатывает.

**Dependency** (`pom.xml`):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

**Конфигурация** (`application.yml`) — минимум:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/myapp
```

**Всё**. Spring Boot автоконфигурация:

1. При старте — читает discovery endpoint `https://keycloak.example.com/realms/myapp/.well-known/openid-configuration`.
2. Извлекает `jwks_uri` — URL публичных ключей.
3. Скачивает JWKS, кэширует.
4. Настраивает `JwtDecoder` который проверяет подпись + `iss` + `exp`.

По умолчанию все endpoints требуют аутентификации. Нужно исключить public — добавить `SecurityFilterChain`:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/actuator/health").permitAll()
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

- `sessionCreationPolicy(STATELESS)` — не создавать сессии. Каждый запрос независим (JWT содержит всё).
- `csrf.disable()` — для stateless JWT API CSRF не актуален (нет cookies).
- `cors()` — включить CORS (для запросов от SPA с другого origin).

Всё. API готов принимать `Authorization: Bearer <JWT>`, автоматически валидировать, авторизовать.

## Security Filter Chain: что реально происходит

Spring Security работает через **цепочку фильтров** (Servlet Filter chain) вставленных в HTTP request lifecycle. Понимать порядок критично для отладки.

При запросе:

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────┐
│  SecurityContextHolderFilter        │  ← читает SecurityContext из ThreadLocal (обычно пустой на входе)
├─────────────────────────────────────┤
│  CorsFilter                          │  ← CORS handling
├─────────────────────────────────────┤
│  CsrfFilter (if enabled)             │
├─────────────────────────────────────┤
│  BearerTokenAuthenticationFilter    │  ← ⭐ извлекает JWT из Authorization header
│  (для Resource Server)              │     проверяет, создаёт Authentication
├─────────────────────────────────────┤
│  ExceptionTranslationFilter         │  ← ловит security exceptions, отдаёт 401/403
├─────────────────────────────────────┤
│  AuthorizationFilter                │  ← проверяет authorizeHttpRequests rules
└─────────────────────────────────────┘
     │
     ▼
  Controller (если авторизован)
```

**BearerTokenAuthenticationFilter** — сердце Resource Server:

1. Извлекает `Authorization` header.
2. Проверяет что начинается с `Bearer `.
3. Достаёт token (всё после `Bearer `).
4. Вызывает `JwtDecoder.decode(token)`:
   - Парсит JWT (3 части).
   - Находит ключ в JWKS по `kid`.
   - Проверяет подпись.
   - Валидирует claims: `exp`, `iat`, `nbf`, `iss`, `aud`.
5. Успех → создаёт `JwtAuthenticationToken` (implements `Authentication`).
6. Сохраняет в `SecurityContextHolder` (ThreadLocal).

Failure на любом шаге → `AuthenticationException` → `ExceptionTranslationFilter` конвертирует в `401 Unauthorized`.

**AuthorizationFilter** после аутентификации проверяет: имеет ли пользователь права на этот endpoint (согласно `authorizeHttpRequests(...)` rules). Если нет — `403 Forbidden`.

**Просмотреть цепочку**:

```java
@Autowired
private FilterChainProxy filterChain;

// В методе (например test)
for (SecurityFilterChain chain : filterChain.getFilterChains()) {
    System.out.println("Chain: " + chain);
    for (Filter filter : chain.getFilters()) {
        System.out.println("  Filter: " + filter.getClass().getSimpleName());
    }
}
```

Или логи `org.springframework.security.web.FilterChainProxy` на DEBUG уровне.

## Автоматическая конфигурация: что делает `issuer-uri`

Одна строка `issuer-uri` даёт много магии.

При старте Spring Boot:

1. **Читает discovery endpoint**: `GET {issuer-uri}/.well-known/openid-configuration`.
2. Парсит JSON, извлекает:
   - `jwks_uri` — URL публичных ключей.
   - `token_endpoint` — token endpoint.
   - `authorization_endpoint` — auth endpoint.
   - `userinfo_endpoint`.
   - Прочие URLs.
3. Создаёт **`NimbusJwtDecoder`** с настройками:
   - JWK Source из `jwks_uri` (с автоматическим кэшированием).
   - Validators: signature + `exp` + `iat` + `iss` (должен == `issuer-uri`).
4. Настраивает `BearerTokenAuthenticationFilter` с этим decoder.

Все настройки автоматические. Можно переопределить если нужно.

**JWKS caching**. По умолчанию:
- Cache TTL: 5 минут.
- При unknown `kid` в JWT — принудительный refresh (максимум раз в 5 минут).
- Backup ключей на случай если Keycloak временно недоступен.

Для key rotation в Keycloak (обычно раз в год) — Spring сам скачает новые ключи при первом токене с новым `kid`.

## Как Spring валидирует JWT: детально

По умолчанию `JwtDecoder` проверяет:

**1. Формат** — 3 части, base64URL.

**2. Header**:
- `alg` — из configured (обычно RS256).
- `kid` — известен в JWKS.

**3. Подпись** — RSA-verify с публичным ключом из JWKS.

**4. Claims**:
- **`iss`** — совпадает с `issuer-uri` (точное сравнение строк).
- **`exp`** — не в прошлом (с clock skew 30 сек).
- **`iat`** — не в будущем.
- **`nbf`** (если есть) — уже прошёл.

**Что НЕ проверяется по умолчанию**:
- **`aud`** — audience! Spring НЕ проверяет что токен для этого приложения. Надо явно настроить.

**Настройка audience validation**:

```java
@Configuration
public class JwtConfig {
    
    @Value("${spring.security.oauth2.resourceserver.jwt.issuer-uri}")
    private String issuerUri;
    
    @Value("${app.expected-audience}")
    private String expectedAudience;
    
    @Bean
    JwtDecoder jwtDecoder() {
        NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuerUri);
        
        // Дефолтные validators (iss, exp, iat) + custom для aud
        OAuth2TokenValidator<Jwt> defaultValidators = 
            JwtValidators.createDefaultWithIssuer(issuerUri);
        OAuth2TokenValidator<Jwt> audienceValidator = 
            token -> {
                List<String> audiences = token.getAudience();
                if (audiences.contains(expectedAudience)) {
                    return OAuth2TokenValidatorResult.success();
                }
                return OAuth2TokenValidatorResult.failure(
                    new OAuth2Error("invalid_token", "Missing required audience", null));
            };
        
        decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
            defaultValidators, audienceValidator));
        
        return decoder;
    }
}
```

Обязательно проверять `aud` в prod. Иначе токен выданный для другого клиента может быть использован против вашего API.

## Custom JwtAuthenticationConverter: правильная extraction ролей Keycloak

**Проблема по умолчанию**. Spring Security ожидает роли в claim `scope` (стандарт OAuth) или `authorities`. Keycloak кладёт роли по-другому:
- Realm roles → `realm_access.roles: ["ADMIN", "USER"]`
- Client roles → `resource_access.<client>.roles: ["orders:read"]`

Без custom converter — Spring не видит ролей. `@PreAuthorize("hasRole('ADMIN')")` не работает.

**Custom converter**:

```java
@Configuration
public class SecurityConfig {
    
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
    
    private Collection<GrantedAuthority> extractKeycloakAuthorities(Jwt jwt) {
        Collection<GrantedAuthority> authorities = new ArrayList<>();
        
        // Realm roles
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess != null) {
            List<String> roles = (List<String>) realmAccess.get("roles");
            if (roles != null) {
                for (String role : roles) {
                    authorities.add(new SimpleGrantedAuthority("ROLE_" + role));
                }
            }
        }
        
        // Client roles (для конкретного client)
        Map<String, Object> resourceAccess = jwt.getClaim("resource_access");
        if (resourceAccess != null) {
            Map<String, Object> clientAccess = 
                (Map<String, Object>) resourceAccess.get("my-api");
            if (clientAccess != null) {
                List<String> clientRoles = (List<String>) clientAccess.get("roles");
                if (clientRoles != null) {
                    for (String role : clientRoles) {
                        authorities.add(new SimpleGrantedAuthority("ROLE_" + role));
                    }
                }
            }
        }
        
        // Scopes (стандартный OAuth)
        String scope = jwt.getClaimAsString("scope");
        if (scope != null) {
            for (String s : scope.split(" ")) {
                authorities.add(new SimpleGrantedAuthority("SCOPE_" + s));
            }
        }
        
        return authorities;
    }
}
```

Теперь `@PreAuthorize("hasRole('ADMIN')")` работает (проверяет наличие `ROLE_ADMIN` authority).

**Convention Spring Security**:
- **Role** → `ROLE_` prefix. `hasRole('ADMIN')` эквивалентно `hasAuthority('ROLE_ADMIN')`.
- **Scope** → `SCOPE_` prefix. `hasAuthority('SCOPE_orders:read')`.

Соблюдение convention упрощает работу с встроенными expression в `@PreAuthorize`.

## Method Security: @PreAuthorize и другие

Endpoint-level authorization через `authorizeHttpRequests(...)` в SecurityFilterChain — грубо (по URL patterns). Method-level authorization — точнее (per method).

**Включить**:

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig { ... }
```

**@PreAuthorize** — проверка перед вызовом метода:

```java
@Service
public class OrderService {
    
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long id) {
        // Только ADMIN может вызвать
    }
    
    @PreAuthorize("hasAuthority('SCOPE_orders:write')")
    public Order createOrder(OrderRequest req) {
        // Требует scope orders:write в токене
    }
    
    // SpEL с параметрами метода и Authentication:
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.name")
    public User getUser(Long userId) {
        // ADMIN может любого, обычный user — только себя
    }
    
    // Комбинация
    @PreAuthorize("hasRole('MANAGER') and hasAuthority('SCOPE_reports:read')")
    public Report generateReport() { ... }
}
```

**@PostAuthorize** — проверка **после** вызова, с доступом к returnObject:

```java
@PostAuthorize("returnObject.owner == authentication.name or hasRole('ADMIN')")
public Document getDocument(Long id) {
    return documentRepo.findById(id).orElseThrow();
    // Документ вернётся клиенту только если owner совпадает с current user или user — ADMIN
}
```

Метод выполнится, но результат не отдастся. Полезно когда permission зависит от загруженного объекта.

**@PreFilter / @PostFilter** — фильтрация коллекций:

```java
@PostFilter("filterObject.owner == authentication.name")
public List<Document> listAll() {
    // Вернёт только те документы, где owner == current user
}
```

Работает через iteration + remove. Медленно на больших коллекциях (O(n) с SpEL evaluation). Лучше фильтровать в SQL.

**@Secured** — легаси, только с role names:

```java
@Secured("ROLE_ADMIN")
public void adminOperation() { }
```

Устаревающий, использовать `@PreAuthorize`.

## Работа с current user

Получить текущего аутентифицированного пользователя в контроллере — несколько способов.

**1. Через SecurityContext (везде в коде)**:

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();   // subject из JWT
```

**2. `Principal` параметр** (Servlet API):

```java
@GetMapping("/me")
public String currentUser(Principal principal) {
    return principal.getName();   // = sub из JWT
}
```

**3. `Authentication` параметр** (Spring Security):

```java
@GetMapping("/me")
public UserInfo currentUser(Authentication auth) {
    Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
    // ...
}
```

**4. `@AuthenticationPrincipal Jwt`** — типизированный доступ к JWT:

```java
@GetMapping("/me")
public UserInfo currentUser(@AuthenticationPrincipal Jwt jwt) {
    String userId = jwt.getSubject();
    String email = jwt.getClaimAsString("email");
    List<String> roles = jwt.getClaimAsStringList("realm_access.roles");
    // Прямой доступ ко всем claims JWT
    return new UserInfo(userId, email, roles);
}
```

**Рекомендуется** — `@AuthenticationPrincipal Jwt`. Прямой доступ к payload, типизировано, чисто.

**Custom UserPrincipal** для business-специфичного user object:

```java
public class AppUser {
    private final String userId;
    private final String email;
    private final Set<String> roles;
    private final String tenantId;
    
    public AppUser(Jwt jwt) {
        this.userId = jwt.getSubject();
        this.email = jwt.getClaimAsString("email");
        this.tenantId = jwt.getClaimAsString("tenant_id");
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        this.roles = realmAccess != null ? 
            new HashSet<>((List<String>) realmAccess.get("roles")) : 
            Collections.emptySet();
    }
    
    // getters
}

// Custom converter создающий AppUser
@Bean
JwtAuthenticationConverter jwtAuthenticationConverter() {
    return new JwtAuthenticationConverter() {
        @Override
        protected AbstractAuthenticationToken convert(Jwt jwt) {
            AppUser appUser = new AppUser(jwt);
            return new AppUserAuthenticationToken(appUser, extractAuthorities(jwt));
        }
    };
}

// В контроллере
@GetMapping("/orders")
public List<Order> getOrders(@AuthenticationPrincipal AppUser user) {
    return orderService.findByTenant(user.getTenantId());
}
```

Чище чем работать с `Jwt` напрямую в каждом контроллере.

## CORS для SPA frontend

SPA (React/Angular) на `https://myspa.com` делает запросы к API на `https://api.myapp.com`. Browser блокирует cross-origin requests без CORS headers.

**Включить CORS**:

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://myspa.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
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

**Preflight OPTIONS** — browser автоматически шлёт `OPTIONS` перед `POST/PUT/DELETE` для проверки CORS. Spring Security по умолчанию **блокирует** OPTIONS (требует auth). Fix — allow OPTIONS в security config:

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
    .anyRequest().authenticated()
);
```

Или ставить CORS filter **перед** security filters (что автоматически делает `http.cors(...)` — filter вставляется в правильное место).

## Setup: OAuth 2.0 Client (server-side web app)

Для классических web-app с backend'ом делающим OAuth flow.

**Dependency**:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

**Конфигурация**:

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

**SecurityFilterChain**:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/", "/public/**").permitAll()
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

**Что происходит при первом посещении защищённого URL**:

1. User заходит на `/dashboard`. Spring Security видит — не аутентифицирован.
2. **Redirect на `/oauth2/authorization/keycloak`** — endpoint Spring Security который инициирует OAuth flow.
3. Spring Security делает redirect на Keycloak `/protocol/openid-connect/auth` с параметрами (client_id, redirect_uri, scope, state).
4. User логинится в Keycloak.
5. Keycloak делает redirect на `redirect-uri` (`/login/oauth2/code/keycloak`).
6. Spring Security обрабатывает callback:
   - Обменивает `code` на `access_token` + `id_token` через token endpoint Keycloak.
   - Валидирует id_token.
   - Создаёт `OAuth2AuthenticationToken` в SecurityContext.
   - Создаёт **HTTP session** (в отличие от stateless Resource Server).
   - Redirect на originally requested URL (`/dashboard`).
7. User видит защищённую страницу.

Дальнейшие запросы — cookies session (`JSESSIONID`), Spring читает из session `OAuth2AuthenticationToken`.

**В контроллере**:

```java
@GetMapping("/dashboard")
public String dashboard(@AuthenticationPrincipal OAuth2User user, Model model) {
    model.addAttribute("username", user.getAttribute("preferred_username"));
    model.addAttribute("email", user.getAttribute("email"));
    return "dashboard";
}
```

## Logout: front-channel и back-channel

**Front-channel logout** (стандарт, простой):

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
1. Spring Security invalidates HTTP session.
2. Redirect на `keycloak/logout` endpoint с `id_token_hint` и `post_logout_redirect_uri`.
3. Keycloak терминирует SSO session.
4. Redirect обратно на приложение.

Результат — user разлогинен и в приложении, и в Keycloak. При следующем заходе — снова полный login flow.

**Back-channel logout** — Keycloak уведомляет клиентов о logout напрямую (без browser). Полезно когда:
- User logout из одного приложения, надо разлогинить во всех.
- Admin force-logout user'а через Admin console.

Настройка в Keycloak client: `Backchannel Logout URL: https://myapp.com/logout/back-channel/openid-connect`.

Настройка в Spring Security 6.2+:

```java
http.oidcLogout(oidc -> oidc.backChannel(Customizer.withDefaults()));
```

Spring Security обрабатывает Keycloak notification, инвалидирует локальную session пользователя.

## Testing security

Тесты security с реальным Keycloak — сложно. Используем mocks.

**MockMvc + @WithMockUser** для простых case:

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

**JWT mock** для более реалистичных тестов:

```java
@Test
void withRealJwt() throws Exception {
    Jwt jwt = Jwt.withTokenValue("mock-token")
        .header("alg", "RS256")
        .claim("sub", "user-123")
        .claim("preferred_username", "ivan")
        .claim("realm_access", Map.of("roles", List.of("ADMIN")))
        .build();
    
    mockMvc.perform(get("/api/orders")
            .with(jwt().jwt(jwt).authorities(new SimpleGrantedAuthority("ROLE_ADMIN"))))
        .andExpect(status().isOk());
}
```

**Integration test с реальным Keycloak** — через **Testcontainers**:

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
    
    // Tests с реальным Keycloak в контейнере
}
```

Медленнее (запуск контейнера ~30 сек), но покрывает реальную интеграцию.

## Диагностика в проде

**Симптом «401 Unauthorized»**. Проверить:

1. **Header формат**: `Authorization: Bearer <token>` (не `bearer`, не `<token>` без Bearer).
2. **Token валиден**: декодировать на jwt.io, проверить exp не в прошлом.
3. **Signature**: правильный ключ (JWKS Keycloak доступен из приложения?).
4. **iss claim**: совпадает с `issuer-uri` в конфиге.
5. **Логи Spring Security**: включить `logging.level.org.springframework.security=DEBUG`, увидеть какой шаг falied.

**Симптом «403 Forbidden»**. Аутентификация прошла, но авторизация нет. Проверить:

1. Какие authorities реально в SecurityContext (log в контроллере или debug).
2. Роли Keycloak доходят до Spring? — правильный `JwtAuthenticationConverter`?
3. `@PreAuthorize` expression — правильный синтаксис? `hasRole('ADMIN')` требует authority `ROLE_ADMIN`.
4. URL patterns в `authorizeHttpRequests` — правильные matchers?

**Симптом «работает локально, не работает в prod»**. Обычно:

1. **`issuer-uri` не тот** — dev vs prod Keycloak.
2. **HTTPS mismatch** — Keycloak на `https://`, `iss` в токене с `https://`, а Spring считает разные строки чем `http://`. Проверить exact match.
3. **Prod Keycloak за прокси** — `iss` может быть внутренний URL, недоступный из приложения для discovery. Настроить `KC_HOSTNAME`, `KC_HOSTNAME_URL` в Keycloak.
4. **Firewall**: приложение не может достучаться до Keycloak JWKS endpoint. Проверить сетевую доступность.
5. **Clock skew** — время серверов сильно расходится → exp / iat ошибки. Настроить NTP.

**Enable debug logs**:

```yaml
logging:
  level:
    org.springframework.security: DEBUG
    org.springframework.security.oauth2: DEBUG
    org.springframework.web: DEBUG
```

В логах видно каждый шаг: filter chain, JWT parsing, validation errors, authority extraction.

**Diagnostic endpoint** — полезно иметь endpoint отдающий текущее security state:

```java
@GetMapping("/debug/whoami")
public Map<String, Object> whoami(@AuthenticationPrincipal Jwt jwt) {
    return Map.of(
        "subject", jwt.getSubject(),
        "issuer", jwt.getIssuer().toString(),
        "audience", jwt.getAudience(),
        "expires", jwt.getExpiresAt(),
        "claims", jwt.getClaims(),
        "authorities", SecurityContextHolder.getContext().getAuthentication().getAuthorities()
    );
}
```

В prod закрыть за admin ролью или отключить.

## Migration с deprecated Keycloak adapter

Раньше стандарт был `keycloak-spring-boot-adapter`. С Keycloak 21+ и Spring Boot 3+ — **deprecated**. Правильный путь — Spring Security native OAuth support.

**Что менять**:

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

**Security config** — убрать extends `KeycloakWebSecurityConfigurerAdapter`, использовать современный `SecurityFilterChain` bean (пример выше).

**Extraction ролей** — раньше Keycloak adapter автоматически понимал `realm_access.roles`. Теперь надо custom `JwtAuthenticationConverter` (см. выше).

**Плюсы миграции**:
- Стандарт (OAuth spec, не vendor-specific).
- Активно поддерживается (Keycloak adapter — не развивается).
- Работает с любым OIDC-совместимым IDP (Auth0, Okta, Google — не только Keycloak).
- Меньше зависимостей.

## Заключение

**Три типа Spring OAuth приложений**: OAuth 2.0 Client (инициирует login, session-based, для server-side web-apps), Resource Server (принимает JWT, stateless, для API), комбо. Современные API — обычно Resource Server.

**Setup Resource Server** — один starter (`spring-boot-starter-oauth2-resource-server`) + одна строка `spring.security.oauth2.resourceserver.jwt.issuer-uri`. Spring автоматически: читает discovery, скачивает JWKS с кэшированием, настраивает `JwtDecoder` с validation подписи + iss + exp.

**Security Filter Chain** — цепочка фильтров: SecurityContextHolderFilter → CorsFilter → CsrfFilter → BearerTokenAuthenticationFilter (извлекает JWT, валидирует, создаёт Authentication) → ExceptionTranslationFilter → AuthorizationFilter.

**Что Spring НЕ проверяет по умолчанию**: `aud` (audience). Обязательно настроить в prod через custom `JwtValidator`.

**Custom JwtAuthenticationConverter** обязателен для Keycloak — стандартный не понимает `realm_access.roles` и `resource_access.<client>.roles`. Извлекать оба + scopes, mapping в GrantedAuthority с префиксом `ROLE_` (для hasRole) и `SCOPE_` (для hasAuthority).

**Method Security** через `@EnableMethodSecurity` + `@PreAuthorize`/`@PostAuthorize`/`@PreFilter`/`@PostFilter`. SpEL expressions с доступом к параметрам метода и Authentication. `@Secured` — legacy, использовать `@PreAuthorize`.

**Current user**: `@AuthenticationPrincipal Jwt jwt` — рекомендуется для типизированного доступа. Custom `AppUser` через custom `JwtAuthenticationConverter` — чище для business-специфичных полей.

**CORS для SPA** — настроить `CorsConfigurationSource`, включить в `HttpSecurity.cors()`. Preflight OPTIONS обычно permitAll.

**OAuth 2.0 Client** для server-side web app — starter `oauth2-client`, конфиг с client-id/secret/scope. Автоматический Authorization Code flow, session-based, `@AuthenticationPrincipal OAuth2User` в контроллерах.

**Logout**: front-channel через `OidcClientInitiatedLogoutSuccessHandler` — приложение redirect'ит на Keycloak logout, тот терминирует SSO session. Back-channel (Spring 6.2+) — Keycloak уведомляет приложения о logout напрямую через backchannel URL.

**Testing**: `@WithMockUser(roles=...)` для простых тестов, `jwt()` postprocessor для реалистичных JWT mock, Testcontainers Keycloak для integration tests.

**Диагностика в проде**:
- 401 → JWT валиден? iss совпадает? JWKS доступен?
- 403 → правильные authorities? Custom converter работает?
- Debug logging `org.springframework.security=DEBUG` показывает каждый шаг.
- Diagnostic endpoint `/debug/whoami` для быстрой проверки.

**Migration с Keycloak adapter**: убрать `keycloak-spring-boot-adapter`, добавить `spring-boot-starter-oauth2-resource-server`. Заменить конфиг на `issuer-uri`. Custom `JwtAuthenticationConverter` для ролей. Плюсы — стандарт, active support, работает с любым OIDC IDP.

OAuth/OIDC протокол — файл 117. JWT детально — 118. Keycloak как IDP — 119. Здесь была глубина по Spring Security + Keycloak integration через standards.
