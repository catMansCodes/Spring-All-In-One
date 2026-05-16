# Spring Boot — Auto-Configuration, Starters, Profiles, Embedded Servers

📁 Package: `com.catmanscode.spring.boot`

---

## Auto-Configuration Internals

Spring Boot's auto-configuration is driven by `@EnableAutoConfiguration` (pulled in by `@SpringBootApplication`). At startup, `AutoConfigurationImportSelector` reads all `AutoConfiguration.imports` files (Spring Boot 3.x; previously `spring.factories`) on the classpath and loads candidate configuration classes.

Each candidate is guarded by `@Conditional*` annotations evaluated in order:

```
@ConditionalOnClass        — class must be on classpath
@ConditionalOnMissingBean  — no user-defined bean of that type
@ConditionalOnProperty     — property present / matches value
@ConditionalOnWebApplication — running in a servlet or reactive context
```

**Key insight:** auto-configuration is not magic — it is `@Configuration` + `@Conditional`. The moment you define your own bean, `@ConditionalOnMissingBean` causes Boot to back off. This is the override mechanism.

To inspect what was applied and why: `--debug` flag or `ConditionEvaluationReport`. `spring-boot-actuator` exposes `/actuator/conditions`.

---

## Starters

A starter is a curated POM that pulls in a coherent set of dependencies. It contains no code; the auto-configuration lives in the library itself (typically `*-autoconfigure` artifact).

| Starter | Brings in |
|---------|-----------|
| `spring-boot-starter-web` | Spring MVC, Tomcat, Jackson |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA, HikariCP |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, Spring Test |

Custom starter structure:
```
my-starter/
├── my-starter-autoconfigure/   ← @Configuration classes + AutoConfiguration.imports
└── my-starter/                 ← POM-only, depends on autoconfigure
```

---

## `application.yml` vs `application.properties`

Both are equivalent in capability; YAML is preferred for hierarchical config. Spring Boot 3.x loads in this order (higher = higher priority):

1. Command-line arguments (`--server.port=9090`)
2. `SPRING_APPLICATION_JSON` env var
3. OS environment variables
4. `application-{profile}.yml` inside jar
5. `application.yml` inside jar
6. `@PropertySource` annotations (not processed early — cannot set `logging.*`)

**Type-safe configuration:** `@ConfigurationProperties(prefix = "app")` binds a whole namespace to a POJO. Pair with `@Validated` for startup-time constraint checking. Prefer this over scattered `@Value` injections.

---

## Profiles

Profiles activate conditional configuration:

```yaml
# application-prod.yml — active when spring.profiles.active=prod
spring:
  datasource:
    url: jdbc:postgresql://prod-host/db
```

Activation priority: `spring.profiles.active` property > `SPRING_PROFILES_ACTIVE` env var > `SpringApplication.setAdditionalProfiles()`.

`@Profile("!prod")` — negate a profile. `spring.profiles.include` — always-on supplementary profiles. In Spring Boot 2.4+, profile groups (`spring.profiles.group.prod=prod-db,prod-monitoring`) replace cascading profile files.

---

## Embedded Servers

Spring Boot embeds Tomcat by default. The `TomcatServletWebServerFactory` (or Jetty/Undertow equivalents) is auto-configured and registered as a bean, making it overridable.

Tomcat thread model (blocking): one thread per request from a pool (`server.tomcat.threads.max`, default 200). Under high concurrency this becomes the bottleneck — switch to WebFlux + Netty for truly reactive workloads.

Undertow is the recommended alternative for traditional MVC: lower memory footprint, XNIO non-blocking I/O for connectors while keeping servlet semantics.

Switching:
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <exclusions>
    <exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
    </exclusion>
  </exclusions>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

---

## `SpringApplication` Bootstrap Sequence

1. Detect application type (SERVLET / REACTIVE / NONE)
2. Load `ApplicationContextInitializer` and `ApplicationListener` from `spring.factories`
3. Determine main application class via stack walk
4. `run()` — publish `ApplicationStartingEvent`
5. Prepare environment (`ConfigurableEnvironment`), publish `ApplicationEnvironmentPreparedEvent`
6. Create and refresh `ApplicationContext`
7. Publish `ApplicationStartedEvent` → `ApplicationReadyEvent`

`CommandLineRunner` / `ApplicationRunner` beans execute between `Started` and `Ready`. Failure before `Ready` fires `ApplicationFailedEvent`.

---

## Common Interview Pitfalls

- `@SpringBootApplication` is a composed annotation: `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. Component scan roots at the annotated class's package — misplacing it causes beans to be missed.
- `spring.main.allow-bean-definition-overriding=true` is a smell; it masks duplicate bean definitions that should be resolved explicitly.
- Actuator `/actuator/env` can leak secrets in production — always configure `management.endpoints.web.exposure.include` explicitly.
- Fat jar (`spring-boot-maven-plugin`) uses a nested classloader; libraries are loaded from `BOOT-INF/lib/` JARs, not the flat classpath. This can cause `ClassCastException` when a library loaded by the parent classloader tries to cast to a class loaded by the nested one.
