# API Documentation — Swagger / OpenAPI, Versioning Docs, Securing Endpoints

📁 Package: `com.catmanscode.spring.documentation`

---

## OpenAPI Specification (OAS)

OpenAPI 3.x is the industry-standard machine-readable API description format. Swagger is the original toolchain; "Swagger" and "OpenAPI" are often used interchangeably but OpenAPI is the specification, Swagger tools (Swagger UI, Swagger Editor) implement it.

The spec describes:
- Servers, base paths
- Endpoints (`paths`), HTTP methods, parameters
- Request/response schemas (`components/schemas`)
- Security schemes (`components/securitySchemes`)

---

## Springdoc OpenAPI (Spring Boot 3.x)

`springdoc-openapi` is the maintained integration library for Spring Boot 3.x (replaces `springfox` which is abandoned and incompatible with Spring 6).

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.x.x</version>
</dependency>
```

Default endpoints:
- `GET /v3/api-docs` — machine-readable JSON spec
- `GET /v3/api-docs.yaml` — YAML spec
- `GET /swagger-ui.html` → redirects to `GET /swagger-ui/index.html`

Springdoc auto-generates the spec by scanning `@RestController` classes, their `@RequestMapping` annotations, parameter types, and return types. No additional annotations are required for a basic spec.

---

## Enriching the Spec

### Global Metadata

```java
@Bean
public OpenAPI apiInfo() {
    return new OpenAPI()
        .info(new Info()
            .title("Spring All-In-One API")
            .version("1.0.0")
            .description("Interview prep API")
            .contact(new Contact().name("catMan").email("dev@catmanscode.com")))
        .addServersItem(new Server().url("https://api.example.com").description("Production"));
}
```

### Operation-Level Annotations

```java
@Operation(
    summary = "Get user by ID",
    description = "Returns a single user. Returns 404 if not found.",
    responses = {
        @ApiResponse(responseCode = "200", description = "User found",
            content = @Content(schema = @Schema(implementation = UserResponse.class))),
        @ApiResponse(responseCode = "404", description = "User not found",
            content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
    }
)
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) { ... }
```

### Schema Annotations

```java
@Schema(description = "User creation request")
public record CreateUserRequest(
    @Schema(description = "Full name", example = "Alice Smith") String name,
    @Schema(description = "Email address", example = "alice@example.com") String email
) {}
```

**Balance:** over-annotating creates maintenance burden. Let springdoc infer what it can from types and Bean Validation annotations (`@NotBlank`, `@Email` are reflected in the schema automatically).

---

## Versioning Documentation

### Per-Version GroupedOpenApi

```java
@Bean
public GroupedOpenApi v1Api() {
    return GroupedOpenApi.builder()
        .group("v1")
        .pathsToMatch("/api/v1/**")
        .build();
}

@Bean
public GroupedOpenApi v2Api() {
    return GroupedOpenApi.builder()
        .group("v2")
        .pathsToMatch("/api/v2/**")
        .build();
}
```

Each group gets its own spec at `/v3/api-docs/v1` and `/v3/api-docs/v2`, selectable in Swagger UI via the dropdown.

### Deprecating Endpoints

```java
@Operation(deprecated = true, summary = "Get user (deprecated, use v2)")
@GetMapping("/api/v1/users/{id}")
```

Mark deprecated endpoints in the spec and add `Deprecation` and `Sunset` HTTP response headers for API consumers:

```java
response.addHeader("Deprecation", "true");
response.addHeader("Sunset", "Sat, 01 Jan 2026 00:00:00 GMT");
```

---

## Securing Swagger Endpoints

### Hiding Swagger in Production

```yaml
springdoc:
  api-docs:
    enabled: false   # per profile
  swagger-ui:
    enabled: false
```

Use `@Profile("!prod")` on the `OpenAPI` bean or disable via `application-prod.yml`.

### Requiring Authentication for Swagger UI

Option 1 — Spring Security path restriction:
```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").hasRole("INTERNAL")
    .anyRequest().authenticated()
)
```

Option 2 — HTTP Basic over HTTPS for internal tooling access.

### Documenting Security Schemes in the Spec

```java
@Bean
public OpenAPI securedApi() {
    return new OpenAPI()
        .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
        .components(new Components()
            .addSecuritySchemes("bearerAuth",
                new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")));
}
```

This enables the "Authorize" button in Swagger UI — users enter their JWT and all subsequent requests include the `Authorization: Bearer <token>` header.

---

## Production Documentation Strategy

### API Client Generation

The OpenAPI spec is machine-readable — generate client SDKs and server stubs with `openapi-generator`:

```bash
openapi-generator-cli generate \
  -i http://localhost:8080/v3/api-docs \
  -g typescript-axios \
  -o ./generated/client
```

Integrate into the build pipeline: generate spec at build time, fail CI if the spec differs from the committed version (contract-first discipline).

### Contract-First vs Code-First

| | Code-First | Contract-First |
|--|-----------|----------------|
| Spec source | Derived from code annotations | Written first in YAML |
| Advantage | Single source of truth in code | API design reviewed before implementation |
| Tools | Springdoc | OpenAPI Generator (generates server stubs) |

For internal APIs: code-first with springdoc is pragmatic. For public or partner APIs: contract-first enforces intentional design.

### Changelog and Versioning Docs

Maintain a `CHANGELOG.md` alongside the spec. Use semantic versioning for the API (`MAJOR.MINOR.PATCH`):
- **MAJOR** — breaking changes (removed fields, changed types)
- **MINOR** — additive changes (new endpoints, new optional fields)
- **PATCH** — non-observable fixes (documentation corrections)

Tools like `openapi-diff` can automatically detect breaking changes between spec versions — integrate into CI to gate merges.

---

## Common Interview Pitfalls

- `springfox` is incompatible with Spring Boot 3.x / Spring 6 — always use `springdoc-openapi` in modern projects.
- Exposing `/v3/api-docs` publicly in production leaks your full API surface and internal schema structure to potential attackers — restrict or disable.
- Swagger UI sends real HTTP requests to your API — if pointed at production, test operations can mutate real data. Use server selectors to separate environments.
- `@Hidden` on a controller or method excludes it from the spec entirely — useful for internal endpoints that should not appear in documentation.
- Bean Validation annotations (`@NotNull`, `@Size`) are automatically reflected as `required` and constraints in the generated schema — do not duplicate them in `@Schema` to avoid drift.
