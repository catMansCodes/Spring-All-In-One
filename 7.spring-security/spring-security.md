# Spring Security — Filter Chain, JWT, OAuth2, CSRF, CORS, Method Security

📁 Package: `com.catmanscode.spring.security`

---

## Authentication vs Authorization

**Authentication** — who are you? Establishes identity. Produces a `Authentication` object stored in `SecurityContextHolder`.

**Authorization** — what can you do? Checks privileges of the established identity against the requested resource or operation.

Spring Security processes both through the **Security Filter Chain** before the request reaches any Spring MVC component.

---

## Security Filter Chain

`SecurityFilterChain` is a chain of `javax.servlet.Filter` implementations registered with the servlet container. Spring Security 6 abandons the `WebSecurityConfigurerAdapter` approach — configure via a `@Bean SecurityFilterChain`:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
        .build();
}
```

**Key filters in default order:**
1. `SecurityContextPersistenceFilter` — loads/saves `SecurityContext` (session or stateless)
2. `UsernamePasswordAuthenticationFilter` — handles form login
3. `BearerTokenAuthenticationFilter` — handles JWT/OAuth2 bearer tokens
4. `ExceptionTranslationFilter` — converts `AuthenticationException` → 401, `AccessDeniedException` → 403
5. `FilterSecurityInterceptor` — performs authorization decisions

Multiple `SecurityFilterChain` beans can coexist — matched by `requestMatcher()`. The first matching chain handles the request.

---

## Authentication Flow

```
Request
  → AuthenticationFilter
  → AuthenticationManager (ProviderManager)
  → AuthenticationProvider (e.g., DaoAuthenticationProvider)
      → UserDetailsService#loadUserByUsername
      → PasswordEncoder#matches
  → Authentication (populated with UserDetails + authorities)
  → SecurityContextHolder
```

`ProviderManager` delegates to a list of `AuthenticationProvider` beans — iterate until one succeeds or all fail. A parent `ProviderManager` is consulted if none match (supports hierarchical auth configuration).

---

## JWT-Based Authentication

JWT is not a Spring Security concept — it is a token format. Spring Security must be wired to extract and validate JWTs from requests.

**Custom filter approach:**

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        String token = extractBearerToken(request);
        if (token != null && jwtService.isValid(token)) {
            UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(
                    jwtService.extractUser(token), null,
                    jwtService.extractAuthorities(token));
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        filterChain.doFilter(request, response);
    }
}
```

**Spring Security OAuth2 Resource Server** (preferred for JWT):
```java
http.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
```
Configure `spring.security.oauth2.resourceserver.jwt.issuer-uri` — Spring fetches the JWKS URI from the authorization server's discovery endpoint and validates signatures automatically. No custom filter needed.

**JWT validation checklist:** signature, `exp` (expiry), `iss` (issuer), `aud` (audience), `nbf` (not before).

---

## OAuth2

### Roles
- **Resource Owner** — user
- **Client** — your application
- **Authorization Server** — issues tokens (Keycloak, Auth0, Okta, Spring Authorization Server)
- **Resource Server** — your API

### Flows

| Flow | Use when |
|------|---------|
| Authorization Code + PKCE | Browser-based apps (SPA, traditional web) |
| Client Credentials | Machine-to-machine (no user involved) |
| Device Code | Limited-input devices (TV, CLI) |
| ~~Implicit~~ | Deprecated — use Authorization Code + PKCE |
| ~~Resource Owner Password~~ | Deprecated — never recommended |

**Spring Boot as OAuth2 Client** (`spring-boot-starter-oauth2-client`): handles Authorization Code flow, token storage, refresh. `@RegisteredOAuth2AuthorizedClient` injects tokens into controller methods.

**Spring Boot as Resource Server** (`spring-boot-starter-oauth2-resource-server`): validates JWTs or opaque tokens via introspection endpoint.

---

## CSRF

**Cross-Site Request Forgery:** a malicious site tricks the browser into sending a state-changing request to your API, using the victim's cookies.

Spring Security's `CsrfFilter` generates a token per session and validates it on mutating requests (`POST`, `PUT`, `DELETE`, `PATCH`).

**When to disable CSRF:**
- Stateless API using JWT (no session cookies) — CSRF attacks require cookies, so it is irrelevant
- Server-to-server calls with `Authorization` header only

**When to keep CSRF enabled:**
- Traditional form-based web apps with session cookies
- Any app where the browser attaches credentials automatically (cookies, Basic Auth)

For SPAs with JWT in `Authorization` header: disable CSRF (`csrf.disable()`). For SPAs with `HttpOnly` session cookies: keep CSRF enabled; sync token via a `GET /csrf` endpoint or double-submit cookie pattern.

---

## CORS

CORS is a browser security policy enforced by the browser, not your server. Your server only needs to send the right headers; the actual enforcement is on the client.

Spring Security's `CorsFilter` must run before `CsrfFilter`. Configure via `CorsConfigurationSource`:

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization","Content-Type"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

**Do not use `allowedOrigins("*")` with `allowCredentials(true)`** — browsers reject this combination (CORS spec violation).

---

## Method-Level Security

Enable with `@EnableMethodSecurity` (replaces deprecated `@EnableGlobalMethodSecurity`):

```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public UserResponse getUser(@PathVariable Long userId) { ... }

@PostAuthorize("returnObject.owner == authentication.name")
public Document findDocument(Long id) { ... }

@PreFilter / @PostFilter  // filter collections
```

**`@Secured`** — simpler but does not support SpEL. **`@RolesAllowed`** — JSR-250 standard, same limitation.

Method security works via AOP — subject to all AOP limitations (self-invocation bypass, `final` method restriction). Annotate the service interface or implementation, not both.

---

## Common Interview Pitfalls

- `SecurityContextHolder` is `ThreadLocal`-based by default. In async contexts (`@Async`, `CompletableFuture`), the security context is lost. Fix: `SecurityContextHolder.setStrategyName(MODE_INHERITABLETHREADLOCAL)` or propagate manually via `DelegatingSecurityContextExecutor`.
- Role names in `hasRole("ADMIN")` automatically prepend `ROLE_` — stored authority must be `ROLE_ADMIN`. Use `hasAuthority("ROLE_ADMIN")` to be explicit and avoid confusion.
- `permitAll()` still executes the filter chain — a malformed JWT will still trigger `AuthenticationException`. Use `shouldNotFilter()` override on the JWT filter for public paths.
- `@PreAuthorize` on `@Bean` methods inside `@Configuration` classes is not intercepted — Spring AOP does not proxy `@Configuration` subclass methods for security.
- Password storage: always use `BCryptPasswordEncoder` (or `Argon2`, `SCrypt`) — never MD5, SHA-1, or plain text. `DelegatedPasswordEncoder` supports migrating hashes across schemes.
