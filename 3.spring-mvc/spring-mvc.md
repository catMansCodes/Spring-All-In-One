# Spring MVC — DispatcherServlet, Controllers, Validation, Exception Handling

📁 Package: `com.catmanscode.spring.mvc`

---

## DispatcherServlet Request Flow

`DispatcherServlet` is a front controller — all HTTP traffic enters through it. The request lifecycle:

```
HTTP Request
  → DispatcherServlet
  → HandlerMapping      — resolves handler (controller method) + interceptors
  → HandlerAdapter      — bridges DispatcherServlet to the handler (e.g., RequestMappingHandlerAdapter)
  → HandlerInterceptor#preHandle
  → Controller method execution
  → HandlerInterceptor#postHandle
  → ViewResolver (for MVC) / MessageConverter (for REST)
  → HandlerInterceptor#afterCompletion
  → HTTP Response
```

`HandlerExceptionResolver` intercepts at any point after handler resolution — this is how `@ExceptionHandler` and `@ControllerAdvice` integrate.

`RequestMappingHandlerAdapter` invokes `@RequestMapping` methods via `HandlerMethodArgumentResolver` (reads request data into parameters) and `HandlerMethodReturnValueHandler` (serializes return values).

---

## Controllers

`@Controller` — returns logical view names resolved by `ViewResolver`.  
`@RestController` — composed of `@Controller` + `@ResponseBody`; every method writes directly to the response body via `HttpMessageConverter`.

**Content negotiation:** Spring selects the `HttpMessageConverter` based on `Accept` header and `produces` attribute. `MappingJackson2HttpMessageConverter` handles JSON; `StringHttpMessageConverter` handles plain text. Order in the converter list matters — first match wins.

---

## Request Mapping

```java
@GetMapping("/users/{id}")          // path variable
@PostMapping(value = "/users", consumes = MediaType.APPLICATION_JSON_VALUE)
@PutMapping("/users/{id}")
@DeleteMapping("/users/{id}")
@PatchMapping("/users/{id}")
```

`@PathVariable` — bound from URI template.  
`@RequestParam` — query string or form field; `required = false` for optional.  
`@RequestBody` — deserializes body via `HttpMessageConverter`; only one per method.  
`@RequestHeader` / `@CookieValue` — access individual headers and cookies.

**Method parameter resolution order matters.** If two resolvers could handle the same parameter type, the first registered wins. Custom `HandlerMethodArgumentResolver` beans are added before built-ins.

---

## Validation

`@Valid` / `@Validated` on a `@RequestBody` or `@ModelAttribute` triggers Bean Validation (Hibernate Validator). `BindingResult` must immediately follow the validated parameter to capture errors without throwing:

```java
public ResponseEntity<?> create(@Valid @RequestBody UserRequest req, BindingResult result) {
    if (result.hasErrors()) { ... }
}
```

Without `BindingResult`, Spring throws `MethodArgumentNotValidException` (400) which `@ExceptionHandler` can catch globally.

`@Validated` (Spring) supports **group validation** — different constraint subsets per operation (Create vs Update). `@Valid` (JSR-380) does not support groups.

Constraint cascade: `@Valid` on a nested object field triggers validation of that object's constraints. `@Validated` on a service class enables **method-level validation** via AOP proxy.

---

## Exception Handling

### `@ExceptionHandler` + `@ControllerAdvice`

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) { ... }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) { ... }
}
```

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`.  
`@ControllerAdvice` can be scoped: `basePackages`, `assignableTypes`, `annotations`.

Exception resolution order:
1. `@ExceptionHandler` on the controller class itself
2. `@ExceptionHandler` in `@ControllerAdvice` classes (ordered by `@Order`)
3. `ResponseEntityExceptionHandler` (Spring's built-in handlers for its own exceptions)

### `ResponseEntityExceptionHandler`

Extend this to get default handling for Spring MVC exceptions (`MethodArgumentNotValidException`, `HttpMessageNotReadableException`, etc.) and override individual methods to customize the response shape.

---

## View Resolution

For traditional MVC (Thymeleaf, FreeMarker):

```
InternalResourceViewResolver   → resolves to JSP/HTML files
ThymeleafViewResolver          → resolves to Thymeleaf templates
ContentNegotiatingViewResolver → delegates to other resolvers based on Accept header
```

`ModelAndView` or just a `String` return value goes through view resolution.  
`Model` / `ModelMap` parameters are populated and passed to the view template.

---

## HandlerInterceptor vs Servlet Filter

| | `HandlerInterceptor` | `Filter` |
|--|--|--|
| Scope | Spring MVC only | All requests (including static resources) |
| Access to handler | Yes — knows which controller method | No |
| Registered via | `WebMvcConfigurer#addInterceptors` | `FilterRegistrationBean` or `@WebFilter` |
| Runs after DispatcherServlet | Yes | No — wraps DispatcherServlet |

Use `Filter` for cross-cutting HTTP concerns (auth token extraction, MDC, CORS). Use `HandlerInterceptor` for MVC-aware logic (access logging, tenant resolution based on controller metadata).

---

## Common Interview Pitfalls

- `@RequestBody` reads the `InputStream` once. Reading it in a `Filter` before the controller requires wrapping the request in `ContentCachingRequestWrapper`.
- `@ResponseStatus` on an `@ExceptionHandler` method is ignored when the method returns `ResponseEntity` — `ResponseEntity` status takes precedence.
- `@ControllerAdvice` only intercepts exceptions thrown within the `DispatcherServlet` — exceptions in `Filter` chain are not caught by it.
- Async request processing (`DeferredResult`, `Callable`) releases the servlet thread; `HandlerInterceptor#postHandle` and `afterCompletion` run on the response thread, not the original request thread.
