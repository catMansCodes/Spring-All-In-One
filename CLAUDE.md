# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is an **interview preparation study repository** for Spring Framework v6+, targeting mid-to-senior/architect-level engineers (3–12+ years experience). It is not a deployable application — it is a structured learning resource organized into 9 modules.

**No build tooling or source code exists yet.** The repository currently contains only documentation. When implementation begins, it will use Maven or Gradle with Spring Boot 3.x (Java 17+) as the base.

## Module Map

Modules are visited in this sequence and map to Java packages under `com.catmanscode.spring.*`:

| Order | Module dir | Package suffix | Core topics |
|-------|-----------|----------------|-------------|
| 1 | `spring-core/` | `.core` | IoC, DI, Bean Lifecycle, ApplicationContext |
| 2 | `spring-boot/` | `.boot` | Auto-Configuration, Starters, Profiles, Embedded Servers |
| 3 | `spring-mvc/` | `.mvc` | DispatcherServlet, Controllers, Validation, ExceptionHandling |
| 4 | `spring-aop/` | `.aop` | Aspects, Proxies (JDK vs CGLIB), `@Transactional` internals |
| 5 | `spring-rest/` | `.rest` | REST design, DTOs, versioning, pagination, error handling |
| 6 | `spring-jpa/` | `.jpa` | Entity lifecycle, Fetch strategies, N+1, JPQL, caching |
| 7 | `spring-security/` | `.security` | Filter chain, JWT, OAuth2, CSRF, CORS, method security |
| 8 | `spring-testing/` | `.testing` | JUnit, Mockito, `@SpringBootTest`, Testcontainers, `@WebMvcTest` |
| 9 | `spring-documentation/` | `.documentation` | Swagger/OpenAPI, versioning docs, securing endpoints |

## Module Layout Convention

Every module follows this structure exactly:

```
module-name/
├── README.md           # Theory + Concepts (the primary reference)
├── examples/           # Runnable code examples
├── interview/          # Q&A organized Basic → Intermediate → Advanced
├── tricky-scenarios/   # Edge cases, gotchas, and real-world debugging
└── diagrams/           # Architecture & flow diagrams (draw.io / PlantUML)
```

## Content Authoring Conventions

- **Audience:** assume deep Java/Spring production experience; skip introductory definitions, focus on internals, trade-offs, and "why it works this way."
- **`interview/` files:** structure Q&A from Basic → Advanced within each file; each answer should explain *why*, not just *what*.
- **`tricky-scenarios/` files:** each scenario should describe the symptom, root cause, and the correct fix/workaround — format as a mini case study.
- **`examples/` code:** annotate only non-obvious behaviour (e.g., proxy self-invocation bypass, lazy-init edge cases); avoid obvious comments.
- **Diagrams:** prefer PlantUML (`.puml`) for sequence/flow and draw.io (`.drawio`) for architecture overviews.

## Build Commands (once implemented)

```bash
# Maven
./mvnw clean install                              # Build all modules
./mvnw test                                       # Run all tests
./mvnw test -Dtest=ClassName                      # Run a single test class
./mvnw spring-boot:run                            # Run the application

# Gradle
./gradlew build                                   # Build all modules
./gradlew test                                    # Run all tests
./gradlew test --tests "fully.qualified.ClassName" # Run a single test
./gradlew bootRun                                 # Run the application
```
