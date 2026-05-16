# Spring Core — IoC, DI, Bean Lifecycle, ApplicationContext

📁 Package: `com.catmanscode.spring.core`

---

## Inversion of Control (IoC)

IoC inverts the responsibility of object creation and dependency wiring from application code to the container. The container reads configuration metadata (annotations, XML, or Java `@Configuration`) and manages object instantiation, wiring, and destruction.

The `BeanFactory` is the root contract; `ApplicationContext` extends it with event publication, i18n, resource loading, and AOP integration. In production code you always use `ApplicationContext` — `BeanFactory` exists as the bare-minimum SPI.

---

## Dependency Injection Styles

| Style | How it works | Trade-offs |
|-------|-------------|-----------|
| Constructor | Dependencies passed via constructor | Immutable fields, easy to test, fails fast if missing |
| Setter | Dependencies injected via setters after construction | Allows optional dependencies; mutable state |
| Field (`@Autowired`) | Injected directly into field via reflection | Hides dependencies; breaks without container; avoid in production |

**Prefer constructor injection** — it makes the dependency graph explicit and enables `final` fields.

### `@Qualifier` vs `@Primary`

- `@Primary` — marks one bean as the default when multiple candidates exist; resolved at registration time.
- `@Qualifier("beanName")` — injected at the call site; overrides `@Primary`; required when two beans of the same type must coexist.

---

## Bean Scopes

| Scope | Lifecycle | Typical use |
|-------|-----------|-------------|
| `singleton` | One instance per `ApplicationContext` | Stateless services, repositories |
| `prototype` | New instance every `getBean()` call | Stateful objects, commands |
| `request` | One per HTTP request | Web-tier DTOs, request state |
| `session` | One per HTTP session | Shopping carts, user preferences |
| `application` | One per `ServletContext` | Shared app-wide state |

**Singleton injecting prototype — the injection point problem.** A singleton bean receives the prototype at construction; subsequent calls get the same stale instance. Fix: inject `ObjectFactory<T>` or use `@Lookup`-annotated method injection so the container intercepts each call.

---

## Bean Lifecycle

```
Instantiation
  → Populate properties (DI)
  → BeanNameAware / BeanFactoryAware / ApplicationContextAware
  → BeanPostProcessor#postProcessBeforeInitialization   ← AOP proxies are created here
  → @PostConstruct / InitializingBean#afterPropertiesSet / init-method
  → BeanPostProcessor#postProcessAfterInitialization
  → Bean ready for use
  → @PreDestroy / DisposableBean#destroy / destroy-method   (singleton only)
```

`BeanPostProcessor` runs for *every* bean — placing expensive logic there impacts startup time. `BeanFactoryPostProcessor` runs *before* instantiation and is used to modify bean definitions (e.g., `PropertySourcesPlaceholderConfigurer`).

---

## ApplicationContext Hierarchy

Child contexts inherit parent beans but not vice versa. Classic Spring MVC setup splits contexts: root context (services, repos) loaded by `ContextLoaderListener`; servlet context (controllers, view resolvers) loaded by `DispatcherServlet`. With Spring Boot this separation is collapsed into a single context.

**Refresh cycle** — `AbstractApplicationContext#refresh()` is the 12-step bootstrap sequence (prepareRefresh → obtainBeanFactory → invokeBeanFactoryPostProcessors → registerBeanPostProcessors → finishBeanFactoryInitialization → finishRefresh). Understanding it is critical for diagnosing startup failures.

---

## Key Annotations

| Annotation | Purpose |
|-----------|---------|
| `@Component` | Generic stereotype; triggers classpath scanning |
| `@Service` | Semantic alias for `@Component` in the service layer |
| `@Repository` | `@Component` + enables PersistenceExceptionTranslationPostProcessor |
| `@Configuration` | Declares a bean factory; methods annotated `@Bean` produce managed beans |
| `@Lazy` | Defers initialization to first use; breaks eager validation at startup |
| `@DependsOn` | Forces explicit ordering when natural DI order is insufficient |
| `@Conditional` | Registers beans only when a condition evaluates true (foundation of Boot auto-config) |

---

## Circular Dependencies

Spring resolves circular dependencies for **singleton + setter/field injection** by eagerly exposing a partially constructed proxy via the third-level cache (`singletonFactories`). **Constructor injection circular dependencies always fail** — by design, since they indicate a design problem. Fix by refactoring to break the cycle or using `@Lazy` on one constructor parameter.

---

## Common Interview Pitfalls

- `@Configuration` classes are CGLIB-subclassed so `@Bean` method calls are intercepted to return the cached singleton. Plain `@Component` classes are NOT subclassed — calling a `@Bean` method on them creates a new instance each time.
- `ApplicationContext` itself is a bean and can be `@Autowired`, but doing so is usually a smell; prefer injecting specific dependencies.
- `@PostConstruct` runs after DI but before the bean is placed in the context — `ApplicationContext#getBean()` inside it can trigger partial initialization.
