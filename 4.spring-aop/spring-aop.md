# Spring AOP — Aspects, Proxies, Pointcuts, `@Transactional`

📁 Package: `com.catmanscode.spring.aop`

---

## Core Concepts

| Term | Definition |
|------|-----------|
| **Aspect** | Module encapsulating a cross-cutting concern (`@Aspect` class) |
| **Advice** | Action taken at a join point (`@Before`, `@After`, `@Around`, `@AfterReturning`, `@AfterThrowing`) |
| **Join Point** | Point in execution where advice can be applied; in Spring AOP, always a method execution |
| **Pointcut** | Predicate selecting join points; expressed in AspectJ pointcut expression language |
| **Weaving** | Applying aspects to target objects; Spring does this at **runtime** via proxies |
| **Target** | The object being advised; Spring wraps it with a proxy |

---

## Proxy Mechanism

Spring AOP uses **proxy-based** weaving, not bytecode weaving. Two proxy strategies:

### JDK Dynamic Proxy
- Requires the target to implement at least one interface
- The proxy implements the same interface(s)
- `java.lang.reflect.Proxy` creates the proxy at runtime
- **Limitation:** can only intercept calls made through the interface reference

### CGLIB Proxy
- Subclasses the target class at runtime (using CGLIB byte-buddy in Spring 6)
- No interface required
- **Limitation:** cannot proxy `final` classes or `final` methods (subclass override is impossible)
- Spring Boot defaults to CGLIB for `@Configuration` and prefers it for `@Transactional` beans since Spring 5.2

Forcing CGLIB: `@EnableAspectJAutoProxy(proxyTargetClass = true)` or `spring.aop.proxy-target-class=true` (Boot default).

---

## Self-Invocation Problem

The most critical AOP gotcha: when a bean calls its own method, the call bypasses the proxy.

```java
@Service
public class OrderService {
    public void placeOrder() {
        this.processPayment();  // bypasses proxy — @Transactional on processPayment is ignored
    }

    @Transactional
    public void processPayment() { ... }
}
```

**Fixes:**
1. Restructure to inject the service into itself (`@Autowired OrderService self`) — ugly but works
2. Use `AopContext.currentProxy()` (requires `exposeProxy = true`)
3. Extract `processPayment` into a separate bean

---

## Advice Types and Execution Order

```
@Around (before proceed())
  @Before
    target method
  @AfterReturning / @AfterThrowing
@After (always runs, like finally)
@Around (after proceed())
```

`@Around` is the most powerful — it can suppress exceptions, modify return values, and short-circuit execution entirely. `ProceedingJoinPoint#proceed(args)` allows modifying arguments before delegation.

When multiple aspects apply to the same join point, `@Order` controls precedence: lower value = outer advice (wraps first, exits last). Unordered aspects have undefined relative ordering.

---

## Pointcut Expressions

```java
// method in a package
execution(* com.catmanscode.spring.service.*.*(..))

// method with specific annotation
@annotation(org.springframework.transaction.annotation.Transactional)

// method on a type with an annotation
@within(org.springframework.stereotype.Service)

// argument type
args(java.lang.String, ..)

// combining
execution(* com.example..*(..)) && @annotation(Logged)
```

**Performance:** `execution` expressions are matched per method invocation. Cache the pointcut in a `@Pointcut` method to reuse it across advice and avoid string parsing overhead.

---

## `@Transactional` Deep Dive

`@Transactional` is implemented as an AOP advice (`TransactionInterceptor`). The proxy intercepts the call, opens a transaction, delegates to the target, and commits or rolls back.

### Propagation

| Propagation | Behavior |
|-------------|---------|
| `REQUIRED` (default) | Joins existing tx; creates new if none |
| `REQUIRES_NEW` | Always creates a new tx; suspends outer |
| `NESTED` | Savepoint within outer tx; rolls back to savepoint on exception |
| `SUPPORTS` | Joins if exists; runs non-transactionally if not |
| `NOT_SUPPORTED` | Suspends active tx; runs non-transactionally |
| `NEVER` | Throws if a tx is active |
| `MANDATORY` | Throws if no active tx |

### Isolation Levels

| Level | Prevents |
|-------|---------|
| `READ_UNCOMMITTED` | Nothing |
| `READ_COMMITTED` | Dirty reads |
| `REPEATABLE_READ` | Dirty + non-repeatable reads |
| `SERIALIZABLE` | All anomalies (dirty, non-repeatable, phantom) |

### Rollback Rules

Default: rolls back on `RuntimeException` and `Error`; commits on checked exceptions.  
Override: `rollbackFor = Exception.class` or `noRollbackFor = OptimisticLockException.class`.

### Common Pitfalls

- Self-invocation (covered above)
- `@Transactional` on `private` or `final` methods — proxy cannot intercept, annotation silently ignored
- `@Transactional(readOnly = true)` sets a Hibernate flush mode of `NEVER` — mutations are not flushed, but reads get a performance hint (connection marked read-only, Hibernate skips dirty checking)
- `REQUIRES_NEW` inside `REQUIRED` — the inner tx commits or rolls back independently; the outer tx does not see inner changes until the outer commits

---

## Cross-Cutting Use Cases

**Logging:** `@Around` on `@annotation(Loggable)` — capture method name, args, return value, execution time.

**Security:** Spring Security is itself an AOP-based system; `@PreAuthorize` uses `MethodSecurityInterceptor` backed by `@EnableMethodSecurity`.

**Performance monitoring:** `@Around` with `StopWatch` or Micrometer `Timer`.

**Retry:** Spring Retry's `@Retryable` uses AOP; the aspect catches the specified exception, waits (with backoff), and re-invokes `proceed()`.

---

## Common Interview Pitfalls

- `@EnableAspectJAutoProxy` is auto-applied by Spring Boot — explicitly adding it is harmless but redundant.
- `@Aspect` classes must themselves be Spring beans (`@Component`) to be detected.
- AspectJ (load-time or compile-time weaving) can advise any method including `private`, constructors, and field access — Spring AOP cannot.
- The proxy wraps a *single* target instance; `prototype`-scoped advised beans create a new proxy per injection, which is expensive.
