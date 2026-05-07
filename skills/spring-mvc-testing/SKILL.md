---
name: spring-mvc-testing
description: "Boot 4+ replaced MockMvc assertions with MockMvcTester — incompatible API your training data gets wrong. Read before writing any @WebMvcTest, @RestControllerAdvice test, or @ExceptionHandler test. Triggers: @WebMvcTest, MockMvc, MockMvcTester, @RestController, jsonPath, @GetMapping, @PostMapping, @AutoConfigureMockMvc, MockMvcRequestBuilders, status().isOk(), @RestControllerAdvice, @ControllerAdvice, @ExceptionHandler, MethodArgumentNotValidException, ProblemDetail, setControllerAdvice, contentType(APPLICATION_JSON)."
version: 0.2.0
license: Apache-2.0
---

# Spring MVC Testing

**Signals**: `@WebMvcTest`, `MockMvc`, `jsonPath`, `@AutoConfigureMockMvc`, `@RestController`, `@RestControllerAdvice`, `@ExceptionHandler`, `MockMvcRequestBuilders`, `MockMvcResultMatchers`, `objectMapper`

## Tested With

- Spring Boot 3.2+ / Spring Boot 4.x
- Spring Framework 6.x / Spring Framework 7.x
- Spring Security 6.x / 7.x (auto-configured in `@WebMvcTest`)
- JUnit 5

## Do NOT Use This Skill When

- Testing authentication, authorization, CSRF, JWT, or OAuth2 specifically → use `spring-security-testing`
- Testing reactive endpoints (`WebFlux`, `WebClient`, SSE) → use `spring-webflux-testing`
- Testing JPA repositories or database persistence → use `spring-jpa-testing`
- Testing WebSocket or STOMP messaging → use `spring-websocket-testing`
- Writing pure unit tests without a Spring context → use `spring-testing-fundamentals`

## When to Read References

| Situation | Read |
|-----------|------|
| Setting up `@WebMvcTest`, MockMvc configuration, static imports | `references/mvc-rest-testing-patterns.md` |
| GET, POST, PUT, DELETE request patterns | `references/mvc-rest-testing-patterns.md` |
| `jsonPath` assertions, response headers, multipart uploads | `references/mvc-rest-testing-patterns.md` |
| Testing `@Valid` bean validation (overview) | `references/mvc-rest-testing-patterns.md` |
| Boot 4 `RestTestClient` migration | `references/mvc-rest-testing-patterns.md` |
| Anti-patterns and common mistakes | `references/mvc-rest-testing-patterns.md` |
| `@RestControllerAdvice` / `@ExceptionHandler` deep dive: `setControllerAdvice()`, field-level validation errors, handler dependency mocking, `ProblemDetail` assertions | `references/exception-handler-patterns.md` |
| Standalone vs `@WebMvcTest` controller test setup, status/header/jsonPath patterns (absorbed legacy `unit-test-controller-layer` content) | `references/controller-testing-patterns.md` |

## Testing Exception Handlers

Spring MVC tests reach `@RestControllerAdvice` / `@ExceptionHandler` only when the advice is on the test's component scan **or** registered explicitly in standalone setup. This section covers both flows; for full examples and edge cases see `references/exception-handler-patterns.md`.

### Standalone setup (advice + test controller)

```java
class GlobalExceptionHandlerTest {

    private MockMvc mockMvc;

    @BeforeEach
    void setUp() {
        mockMvc = MockMvcBuilders.standaloneSetup(new TestController())
            .setControllerAdvice(new GlobalExceptionHandler())
            .build();
    }

    @Test
    void notFoundExceptionMapsTo404WithProblemDetail() throws Exception {
        mockMvc.perform(get("/test/not-found"))
            .andExpect(status().isNotFound())
            .andExpect(content().contentType(MediaType.APPLICATION_PROBLEM_JSON))
            .andExpect(jsonPath("$.type").value("https://example.com/errors/not-found"))
            .andExpect(jsonPath("$.title").value("Resource Not Found"))
            .andExpect(jsonPath("$.status").value(404));
    }
}
```

### `@WebMvcTest` slice (advice auto-detected)

```java
@WebMvcTest(OrderController.class)
@Import(GlobalExceptionHandler.class)   // include the advice in the slice
class OrderControllerErrorTest {

    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;

    @Test
    void validationErrorReturnsFieldDetails() throws Exception {
        String invalidPayload = """
            {"email": "not-an-email", "amount": -1}
            """;

        mockMvc.perform(post("/orders").contentType(APPLICATION_JSON).content(invalidPayload))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors.email").exists())
            .andExpect(jsonPath("$.errors.amount").exists());
    }
}
```

### What to assert

- HTTP status from the handler's `@ResponseStatus` (or `ResponseEntity` body).
- Content type — for `ProblemDetail` (Boot 3+, idiomatic in Boot 4) it's `application/problem+json`.
- Body shape: `type`, `title`, `status`, `detail`, `instance` for RFC 7807; or your custom `errors` map for `MethodArgumentNotValidException`.
- For handler dependency mocking (e.g. an audit logger), use `@Mock` + `@InjectMocks` on the advice and pass it to `setControllerAdvice()`.

### Common pitfalls

- Forgetting to register the advice in standalone setup → handler never fires; test fails with default 500.
- Using `@MockBean` for the controller dependency in `@WebMvcTest` is mandatory — real beans won't be wired into the slice.
- Asserting on the raw exception message rather than the response body — error contracts must be tested at the response level.
- Mixing `@ControllerAdvice` (returns view name) with `@RestControllerAdvice` (returns body): use the latter for REST.
