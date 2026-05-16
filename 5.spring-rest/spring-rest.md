# Spring REST — API Design, DTOs, Versioning, Pagination, Error Handling

📁 Package: `com.catmanscode.spring.rest`

---

## REST Principles

**Statelessness** — each request carries all state needed to fulfill it. Session state lives client-side (JWT, cookies). Server-side sessions violate REST and break horizontal scaling.

**Idempotency** — repeated identical requests produce the same server state:
- `GET`, `HEAD`, `OPTIONS`, `DELETE` — idempotent
- `PUT` — idempotent (replace the resource entirely)
- `POST` — not idempotent (creates new resource each time)
- `PATCH` — not idempotent by default (relative updates); can be made idempotent with conditional requests

**HATEOAS** — responses include links to related actions. Rarely implemented fully in practice; Spring HATEOAS provides `EntityModel`, `CollectionModel`, `RepresentationModelAssembler` if needed.

**Richardson Maturity Model:** Level 0 (HTTP tunnel) → Level 1 (Resources) → Level 2 (HTTP verbs + status codes) → Level 3 (HATEOAS). Most production APIs target Level 2.

---

## HTTP Status Codes

| Code | Meaning | When to use |
|------|---------|-------------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST; include `Location` header |
| 204 | No Content | Successful DELETE, PATCH with no response body |
| 400 | Bad Request | Validation failure, malformed JSON |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | Optimistic lock, duplicate resource |
| 422 | Unprocessable Entity | Semantically invalid (business rule violation) |
| 500 | Internal Server Error | Unhandled exception |

---

## DTO Patterns

Separate DTOs for request and response. Never expose JPA entities directly — entity changes should not break API contracts.

```java
// Request: only what the client provides
record CreateUserRequest(
    @NotBlank String name,
    @Email String email
) {}

// Response: what the client receives (may include computed fields, links)
record UserResponse(Long id, String name, String email, Instant createdAt) {}
```

**MapStruct** for DTO mapping: generates compile-time code, zero reflection overhead. Lombok `@Builder` on DTOs for readability.

**Avoid bidirectional coupling:** request DTOs must not reference response DTOs or domain objects. If a use case needs different shapes for the same entity, create distinct DTOs per use case.

---

## API Versioning Strategies

| Strategy | Mechanism | Trade-offs |
|----------|-----------|-----------|
| URI path | `/api/v1/users` | Most visible; easy to route; pollutes URLs |
| Query param | `/api/users?version=1` | Easy to implement; not RESTful (same resource, different representation) |
| Header | `Accept-Version: v1` | Clean URLs; harder to test in browsers |
| Content negotiation | `Accept: application/vnd.app.v1+json` | True REST; complex to implement and document |

**Recommended:** URI versioning for public APIs (visibility, cacheability, simplicity). Header versioning for internal APIs where URL cleanliness matters.

**Deprecation strategy:** Keep old version running, add `Deprecation` and `Sunset` response headers, document removal timeline.

---

## Pagination and Filtering

### Spring Data Pageable

```java
@GetMapping("/users")
public Page<UserResponse> list(
        @PageableDefault(size = 20, sort = "createdAt", direction = DESC) Pageable pageable) {
    return userRepository.findAll(pageable).map(mapper::toResponse);
}
```

Response includes `totalElements`, `totalPages`, `number`, `size` — clients can build navigation from these. For large datasets, `Slice` avoids the expensive `COUNT(*)` query; use it when you only need "has next page."

**Cursor-based pagination** (Keyset pagination): use a unique, indexed column as the cursor (`WHERE created_at < :cursor`). Stable under inserts/deletes; required for real-time feeds. Spring Data 3.x `ScrollPosition`/`Window` API supports this natively.

### Filtering

`Specification<T>` (JPA Criteria API) for dynamic filtering:

```java
repository.findAll(
    Specification.where(nameContains(filter.name()))
                 .and(statusEquals(filter.status())),
    pageable
);
```

**QueryDSL** alternative — type-safe predicate building; less boilerplate than raw Criteria API.

Avoid unbounded queries — always enforce a maximum page size to prevent memory exhaustion.

---

## Standardized Error Handling

### RFC 9457 — Problem Details for HTTP APIs

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Field 'email' must be a valid email address",
  "instance": "/api/v1/users",
  "traceId": "abc-123"
}
```

Spring 6 / Boot 3 natively supports Problem Details via `ProblemDetail` and `ErrorResponse`. Enable with `spring.mvc.problemdetails.enabled=true`.

**Always include `traceId`** — correlates with distributed tracing (Sleuth/Micrometer Tracing). Structured logging keyed on trace ID enables log aggregation.

---

## `RestTemplate` vs `WebClient` vs `RestClient`

| Client | Model | Use when |
|--------|-------|----------|
| `RestTemplate` | Blocking, synchronous | Legacy; still supported but in maintenance mode |
| `WebClient` | Non-blocking, reactive | Reactive stacks (WebFlux); composing async calls |
| `RestClient` | Blocking, fluent (Spring 6.1+) | Modern replacement for `RestTemplate` |

`RestClient` example:
```java
UserResponse user = restClient.get()
    .uri("/users/{id}", id)
    .retrieve()
    .body(UserResponse.class);
```

---

## Common Interview Pitfalls

- Returning `200 OK` for a `POST` that creates a resource is incorrect — use `201 Created` with a `Location` header pointing to the new resource URL.
- `DELETE /resources/{id}` should return `204 No Content`, not `200 OK` with a body.
- Exposing stack traces in error responses is a security risk — map all unhandled exceptions to generic 500 responses in production.
- `Page<Entity>` returned directly from a controller leaks the JPA entity model to the API layer and triggers lazy-load issues during serialization — always map to `Page<DTO>`.
- PATCH semantics: use `Map<String, Object>` or JSON Merge Patch (`application/merge-patch+json`) rather than receiving a full DTO where null means "set to null" vs "leave unchanged."
