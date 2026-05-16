# Spring Boot Testing — JUnit 5, Mockito, `@SpringBootTest`, Testcontainers, `@WebMvcTest`

📁 Package: `com.catmanscode.spring.testing`

---

## Testing Pyramid

```
         /\
        /E2E\         — Few; slow; fragile; catch integration gaps
       /------\
      /Integration\   — Moderate; Spring context; real DB (Testcontainers)
     /------------\
    /  Unit Tests  \  — Many; fast; no Spring context; pure logic
   /-----------------\
```

Spring's test support spans all three layers. The goal is to minimize context-loading tests (expensive) and maximize plain unit tests.

---

## JUnit 5 Fundamentals

| Annotation | Purpose |
|-----------|---------|
| `@Test` | Marks a test method |
| `@BeforeEach` / `@AfterEach` | Setup/teardown per test |
| `@BeforeAll` / `@AfterAll` | Static setup/teardown per class |
| `@ParameterizedTest` + `@MethodSource` | Data-driven tests |
| `@Nested` | Groups related tests in inner classes |
| `@ExtendWith` | Registers JUnit 5 extensions (e.g., `MockitoExtension`) |
| `@DisplayName` | Human-readable test name |

`@TestInstance(Lifecycle.PER_CLASS)` — shared instance across test methods; allows non-static `@BeforeAll`. Use when state setup is expensive and tests are independent via careful cleanup.

---

## Mockito

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    OrderRepository orderRepository;

    @InjectMocks
    OrderService orderService;

    @Test
    void shouldPlaceOrder() {
        given(orderRepository.save(any())).willReturn(savedOrder);
        OrderResponse result = orderService.place(createRequest);
        then(orderRepository).should().save(argThat(o -> o.getStatus() == PENDING));
        assertThat(result.id()).isNotNull();
    }
}
```

**`@Mock` vs `@Spy`:** `@Mock` replaces all methods with no-ops returning defaults; `@Spy` wraps a real instance and intercepts only stubbed calls. Use `@Spy` when you want to test partial behavior of a real object.

**`ArgumentCaptor`:** captures the actual argument passed to a mock for detailed assertions — useful when `argThat` lambdas become too complex.

**`doReturn` vs `when/thenReturn`:** `when(spy.method())` actually calls the real method during stubbing — use `doReturn(value).when(spy).method()` on spies to avoid side effects during setup.

**Strict stubs (`@ExtendWith(MockitoExtension.class)`):** fails tests with unused stubs — keeps tests clean and reveals dead test setup.

---

## `@SpringBootTest`

Loads the full `ApplicationContext`. Expensive — reuse across test classes by keeping the same context configuration.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserApiIntegrationTest {

    @Autowired
    TestRestTemplate restTemplate;

    @Test
    void shouldReturnUser() {
        ResponseEntity<UserResponse> response =
            restTemplate.getForEntity("/api/users/1", UserResponse.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

| `webEnvironment` | Behavior |
|-----------------|---------|
| `MOCK` (default) | Mock servlet environment; use `MockMvc` |
| `RANDOM_PORT` | Real server on random port; use `TestRestTemplate`/`WebTestClient` |
| `DEFINED_PORT` | Real server on `server.port` |
| `NONE` | No web server; tests service layer only |

**Context caching:** Spring caches `ApplicationContext` across test classes with identical configuration. Adding `@MockBean` creates a new context — group tests with the same `@MockBean` set to preserve caching.

---

## `@WebMvcTest`

Loads only the MVC layer (controllers, `@ControllerAdvice`, filters, `WebMvcConfigurer`). Does NOT load services or repositories — use `@MockBean` for those.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockBean
    UserService userService;

    @Test
    void shouldReturn404WhenUserNotFound() throws Exception {
        given(userService.findById(99L)).willThrow(new ResourceNotFoundException("User not found"));

        mockMvc.perform(get("/api/users/99"))
               .andExpect(status().isNotFound())
               .andExpect(jsonPath("$.detail").value("User not found"));
    }
}
```

`MockMvc` performs requests without starting a real server — fast, isolated. Use `.andDo(print())` to debug request/response during development.

**`@WebMvcTest` vs `@SpringBootTest(MOCK)`:** `@WebMvcTest` is narrower (faster context, only MVC beans); `@SpringBootTest(MOCK)` loads everything. Prefer `@WebMvcTest` for controller-layer tests.

---

## `@DataJpaTest`

Loads JPA infrastructure only (entity manager, repositories, embedded H2 by default). Each test runs in a transaction that is rolled back.

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    UserRepository userRepository;

    @Test
    void shouldFindByEmail() {
        userRepository.save(new User("Alice", "alice@example.com"));
        Optional<User> found = userRepository.findByEmail("alice@example.com");
        assertThat(found).isPresent();
    }
}
```

**Problem:** H2 may not support database-specific SQL (PostgreSQL array types, `ON CONFLICT`, CTEs). Use `@AutoConfigureTestDatabase(replace = NONE)` with Testcontainers instead.

---

## Testcontainers

Spins up real Docker containers for integration tests — eliminates H2/in-memory database discrepancies.

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

**`@ServiceConnection` (Spring Boot 3.1+):** eliminates `@DynamicPropertySource` boilerplate — Boot auto-wires the container properties:

```java
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
```

**Reuse across test classes:** `static` containers + `withReuse(true)` — container stays running between test class instantiations if Testcontainers reuse is enabled in `~/.testcontainers.properties`.

---

## `@MockBean` vs `@SpyBean`

| | Description |
|--|--|
| `@MockBean` | Adds a Mockito mock to the Spring context; replaces existing bean of that type |
| `@SpyBean` | Wraps the real Spring bean with a Mockito spy; partial stubbing while keeping real behavior |

`@SpyBean` is useful when you want to verify interactions with a real bean without fully mocking it — e.g., checking a real `EmailService` was called without actually sending emails.

---

## Common Interview Pitfalls

- `@MockBean` causes Spring to create a new `ApplicationContext` for each unique set of mocked beans. Too many `@MockBean` annotations fragment the context cache and slow the test suite. Consolidate into a shared base test class.
- `@Transactional` on integration tests rolls back after each test. If the code under test spawns a new thread (async methods), that thread runs in a different transaction and its changes persist — can cause test interference.
- `MockMvc` performs the full filter chain by default. If security is not the focus of a controller test, add `@WithMockUser` or configure `http.authorizeRequests().anyRequest().permitAll()` in the test security config.
- `@SpringBootTest` does not start a real server by default (`MOCK` environment). Calling `localhost:8080` in such a test will fail silently — use `RANDOM_PORT` with `@LocalServerPort` for actual HTTP calls.
- `Testcontainers` requires Docker to be running. In CI environments, use a Docker-in-Docker (DinD) setup or Testcontainers Cloud to avoid flaky failures from Docker unavailability.
