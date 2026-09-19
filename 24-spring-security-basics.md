# 24. Spring Security основы

Как работает Spring Security. Фильтры, аутентификация, авторизация.

---

## 1. Что делает Spring Security

- **Authentication** — «кто ты?». Проверка credentials (пароль, токен, сертификат).
- **Authorization** — «что тебе можно?». Роли, permissions, policies.
- **Attack protection** — CSRF, XSS, session fixation, brute force.
- **Session management** — создание, timeout, invalidation.

Пример вопросов, на которые Security отвечает:
- Может ли пользователь `berik` вызвать `POST /api/fno/submit`?
- Токен валиден?
- Пароль правильный?
- Пользователь в роли `ADMIN`?

---

## 2. Основа: цепочка фильтров

**Spring Security = набор Servlet-фильтров**. Каждый входящий HTTP-запрос проходит через них до контроллера.

```
HTTP request
    │
    ▼
[FilterChainProxy]
    │
    ├─ SecurityContextPersistenceFilter  ← восстанавливает SecurityContext из session
    ├─ CsrfFilter                         ← проверка CSRF token
    ├─ LogoutFilter                       ← /logout
    ├─ UsernamePasswordAuthFilter         ← /login form
    ├─ BasicAuthenticationFilter          ← HTTP Basic
    ├─ BearerTokenAuthenticationFilter    ← Bearer / JWT (OAuth2)
    ├─ RequestCacheAwareFilter
    ├─ SecurityContextHolderAwareFilter
    ├─ AnonymousAuthenticationFilter      ← гость если нет других
    ├─ SessionManagementFilter
    ├─ ExceptionTranslationFilter         ← ловит AccessDenied, редиректит на login
    └─ FilterSecurityInterceptor / AuthorizationFilter   ← финальная проверка авторизации
    │
    ▼
DispatcherServlet
    │
    ▼
@RestController
```

Порядок фильтров важен. У каждого своя роль.

### 2.1 SecurityFilterChain

Основной bean-декларация цепочки (Spring Security 6):

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));

        return http.build();
    }
}
```

---

## 3. Ключевые концепты

### 3.1 SecurityContext + SecurityContextHolder

Thread-local хранилище текущего пользователя. Из любой точки кода:

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
```

### 3.2 Authentication

Объект с информацией об аутентифицированном пользователе:
- `getName()` — username / subject.
- `getPrincipal()` — сам объект пользователя (обычно `UserDetails` или JWT).
- `getCredentials()` — пароль (после аутентификации обычно null).
- `getAuthorities()` — коллекция ролей / permissions.
- `isAuthenticated()` — true / false.

### 3.3 GrantedAuthority

Интерфейс с одним методом `getAuthority()` — строка. Пример: `"ROLE_ADMIN"`, `"KNP_PERM_CREATE_NZ_N07"`.

По конвенции роли префиксятся `ROLE_`. Отсюда:
```java
.hasRole("ADMIN")        // проверяет ROLE_ADMIN
.hasAuthority("ROLE_ADMIN")   // прямая проверка
.hasAuthority("KNP_PERM_...")  // без префикса
```

### 3.4 UserDetails

Стандартный интерфейс пользователя:
```java
interface UserDetails {
    String getUsername();
    String getPassword();
    Collection<? extends GrantedAuthority> getAuthorities();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

### 3.5 UserDetailsService

Загружает UserDetails по username. Один метод:
```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

Своя реализация:
```java
@Service
class MyUserDetailsService implements UserDetailsService {
    @Autowired UserRepository repo;

    public UserDetails loadUserByUsername(String username) {
        User u = repo.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));
        return org.springframework.security.core.userdetails.User.builder()
            .username(u.getUsername())
            .password(u.getPasswordHash())
            .authorities(u.getRoles().stream().map(r -> "ROLE_" + r).toList().toArray(String[]::new))
            .build();
    }
}
```

Spring использует его в `DaoAuthenticationProvider` для проверки паролей.

### 3.6 PasswordEncoder

Хеширование паролей:
```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

Никогда plain-text. Используй BCrypt / Argon2 / PBKDF2.

При регистрации:
```java
String hash = encoder.encode(rawPassword);
// сохранить hash в БД
```

При проверке:
```java
encoder.matches(rawPassword, storedHash);
```

### 3.7 AuthenticationManager / ProviderManager

**AuthenticationManager** — интерфейс, обычно реализация `ProviderManager` — держит список **AuthenticationProvider**.

Каждый провайдер знает как проверить свой тип аутентификации:
- `DaoAuthenticationProvider` — по UserDetailsService.
- `JwtAuthenticationProvider` — JWT.
- `OAuth2LoginAuthenticationProvider` — OAuth2.
- Custom.

При каждом `authenticate(...)` — идёт по списку, возвращает первый успешный.

---

## 4. Аутентификация по фильтрам

### 4.1 HTTP Basic

`Authorization: Basic <base64(user:pass)>`.

Фильтр `BasicAuthenticationFilter`:
1. Декодирует header.
2. Создаёт `UsernamePasswordAuthenticationToken`.
3. Отдаёт AuthenticationManager.
4. Success → кладёт в SecurityContext.

Настройка:
```java
http.httpBasic(Customizer.withDefaults());
```

### 4.2 Form login

Стандартная HTML-форма. POST /login с `username`, `password`.

```java
http.formLogin(form -> form
    .loginPage("/login")
    .defaultSuccessUrl("/")
    .failureUrl("/login?error"));
```

### 4.3 Bearer / JWT (OAuth2)

`Authorization: Bearer <jwt-token>`.

Фильтр `BearerTokenAuthenticationFilter`:
1. Извлекает токен.
2. Декодирует + верифицирует JWT (`JwtDecoder`, обычно через JWKS).
3. Создаёт `JwtAuthenticationToken`.

Подробно — в файле `27-spring-security-oauth2-keycloak.md`.

### 4.4 Cookies / Session

Классика для web: успешный login → session cookie. Каждый следующий запрос — cookie → session lookup.

Для микросервисов чаще stateless (JWT), см. §5.

---

## 5. Stateless vs stateful

### 5.1 Stateful (classic)

Session на сервере (в memory или Redis). Cookie с session-id. Плюсы:
- Легко invalidate (log out).
- Можно хранить много данных в session.

Минусы:
- Sticky sessions или общая session store (Redis).
- Не масштабируется на много инстансов из коробки.

### 5.2 Stateless (микросервисы)

Каждый запрос несёт токен (JWT). Сервер ничего не хранит. Проверяет подпись/expiration.

```java
http.sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

Плюсы:
- Масштабируется линейно.
- Не нужна session store.
- Можно легко проксировать через LB.

Минусы:
- Invalidate токена сложнее (нужен revoke-list / короткий TTL).
- Токен большой (передаётся с каждым запросом).

**В микросервисах — всегда stateless + JWT**.

---

## 6. CSRF

**CSRF (Cross-Site Request Forgery)** — атакующий сайт заставляет твой браузер сделать запрос на защищённый ресурс от твоего имени (cookies отправятся автоматически).

Защита: **CSRF-token** — секрет, генерируется сервером, кладётся в hidden-поле формы. Атакующий не знает.

```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()));
```

Отключить (для REST API с JWT, где нет cookies):
```java
http.csrf(csrf -> csrf.disable());
```

**Правило**:
- Session-based UI → CSRF ON.
- REST API с Bearer token → CSRF OFF (нет cookies).

---

## 7. CORS

**CORS (Cross-Origin Resource Sharing)** — механизм браузера разрешать/запрещать AJAX запросы на другой домен.

```java
@Bean
CorsConfigurationSource corsConfig() {
    CorsConfiguration cfg = new CorsConfiguration();
    cfg.setAllowedOrigins(List.of("https://knp.kgd.gov.kz"));
    cfg.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    cfg.setAllowedHeaders(List.of("*"));
    cfg.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource src = new UrlBasedCorsConfigurationSource();
    src.registerCorsConfiguration("/**", cfg);
    return src;
}

http.cors(Customizer.withDefaults());
```

---

## 8. Method Security

Тонкая авторизация на методах:

```java
@EnableMethodSecurity
class Config {}

@Service
class FnoService {

    @PreAuthorize("hasRole('ADMIN')")
    void deleteFno(Long id) { ... }

    @PreAuthorize("hasAuthority('KNP_PERM_CREATE_NZ_N07')")
    void createN07() { ... }

    @PreAuthorize("#userId == authentication.name")
    void updateOwnProfile(String userId) { ... }

    @PostAuthorize("returnObject.owner == authentication.name")
    Fno getFno(Long id) { ... }

    @Secured("ROLE_ADMIN")
    void something() { ... }
}
```

- `@PreAuthorize` — проверка до вызова. SpEL.
- `@PostAuthorize` — проверка после (по returnObject).
- `@Secured` — старый, только по authority list.
- `@RolesAllowed` — Jakarta security стандарт.

Реализуется через AOP-прокси. **Всё те же грабли self-invocation!**

---

## 9. Обработка ошибок

### 9.1 Не аутентифицирован → 401

Если нет валидной auth → `AuthenticationEntryPoint` решает что делать:
- Session UI — redirect на /login.
- API — 401 UNAUTHORIZED + JSON error.

```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint((req, resp, e) -> {
        resp.setStatus(401);
        resp.getWriter().write("{\"error\":\"unauthorized\"}");
    }));
```

### 9.2 Аутентифицирован, но нет прав → 403

`AccessDeniedHandler`:
```java
http.exceptionHandling(ex -> ex
    .accessDeniedHandler((req, resp, e) -> {
        resp.setStatus(403);
        resp.getWriter().write("{\"error\":\"forbidden\"}");
    }));
```

---

## 10. Attack protection

Spring Security защищает от:

- **CSRF** — token (§6).
- **Session fixation** — новая session после login.
- **Clickjacking** — `X-Frame-Options: DENY` header.
- **HSTS** — `Strict-Transport-Security` для HTTPS.
- **XSS** — не панацея (нужен escape в шаблонах); но добавляет `X-Content-Type-Options: nosniff`.
- **CSP** — Content Security Policy.

```java
http.headers(h -> h
    .frameOptions(f -> f.deny())
    .httpStrictTransportSecurity(hsts -> hsts.maxAgeInSeconds(31536000)));
```

---

## 11. Пример полной конфигурации (базовая)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filter(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**", "/actuator/info").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED)))
            .headers(h -> h.frameOptions(f -> f.deny()));

        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 12. Тестирование

```java
@Test
@WithMockUser(username = "berik", roles = {"USER"})
void authorizedRequest() {
    // тест как пользователь berik с ROLE_USER
}

@Test
@WithMockUser(authorities = {"KNP_PERM_CREATE_NZ_N07"})
void withPermission() { ... }
```

Или через `SecurityContextHolder.setContext(...)` вручную.

MockMvc:
```java
mockMvc.perform(get("/api/admin").with(user("admin").roles("ADMIN")));
```

---

## 13. Реальные грабли

### 13.1 Забыл disable CSRF в REST API

Symptom: `POST /api/fno` → 403 Forbidden. В логах `Invalid CSRF token`.

Fix: `http.csrf(csrf -> csrf.disable())` для чистого REST API.

### 13.2 Порядок filterChain'ов

Если несколько `SecurityFilterChain` — важен порядок (`@Order`). Первый матчащий выигрывает.

### 13.3 Self-invocation в @PreAuthorize

Метод с @PreAuthorize внутри того же класса → вызов через `this` → прокси не работает → проверка не срабатывает.

### 13.4 Забыл `ROLE_` префикс

`.hasRole("ADMIN")` проверяет `ROLE_ADMIN`. `.hasAuthority("ADMIN")` — просто `ADMIN`.

### 13.5 SecurityContext в другом потоке

`SecurityContextHolder` thread-local. При `@Async` или virtual threads по умолчанию не пробрасывается.

```java
@Bean
DelegatingSecurityContextAsyncTaskExecutor asyncExecutor() {
    return new DelegatingSecurityContextAsyncTaskExecutor(new ThreadPoolTaskExecutor());
}
```

Или `SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL)` (осторожно с pool'ами).

---

## 14. Собесные вопросы

1. **Как работает Spring Security?** — Цепочка фильтров перед DispatcherServlet.
2. **Что такое SecurityContext?** — Thread-local с текущим Authentication.
3. **Разница Authentication и Authorization?** — Аутентификация — кто; авторизация — что можно.
4. **Что такое UserDetails / UserDetailsService?** — Абстракция пользователя + загрузка по username.
5. **PasswordEncoder — зачем?** — Хеширование паролей; BCrypt / Argon2 / PBKDF2.
6. **Разница hasRole vs hasAuthority?** — hasRole автопрефиксует ROLE_.
7. **Что такое CSRF, зачем защищаться?** — Атака cross-site, где атакующий заставляет твой браузер сделать запрос; token защищает.
8. **Когда отключать CSRF?** — Stateless REST API с Bearer token (нет cookies).
9. **Что такое CORS?** — Механизм браузера ограничить cross-origin AJAX.
10. **`@PreAuthorize` vs `@Secured`?** — PreAuthorize с SpEL, гибче; Secured старый, только authority.
11. **Stateless vs stateful auth?** — Stateless (JWT) для микросервисов; stateful (session) для UI.
12. **Что такое AuthenticationManager?** — Оркестратор AuthenticationProvider'ов.

---

## Итог

- Spring Security = цепочка **Servlet-фильтров**.
- **SecurityContext** — thread-local Authentication.
- **UserDetailsService + PasswordEncoder** — базовая аутентификация.
- **Method Security** (`@PreAuthorize`) — тонкая авторизация.
- **CSRF ON** для UI, **OFF** для REST + JWT.
- **Stateless** для микросервисов.
- **CORS** для cross-origin.
- **Attack protection**: заголовки безопасности, HSTS, session-fixation.

Следующий — `25-oauth2-oidc-theory.md`.
