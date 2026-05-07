# Controller Testing Patterns (absorbed from unit-test-controller-layer)

---
name: unit-test-controller-layer
description: Provides patterns for unit testing REST controllers using MockMvc and @WebMvcTest. Generates controller tests that validates request/response mapping, validation, exception handling, and HTTP status codes. Use when testing web layer endpoints in isolation for API endpoint testing, Spring MVC tests, mock HTTP requests, or controller layer unit tests.
allowed-tools: Read, Write, Bash, Glob, Grep
---

# Unit Testing REST Controllers with MockMvc

## Overview

Provides patterns for unit testing `@RestController` and `@Controller` classes using MockMvc. Covers request/response handling, HTTP status codes, request parameter binding, validation, content negotiation, response headers, and exception handling with mocked service dependencies.

## When to Use

Use for: controller tests, API endpoint testing, Spring MVC tests, mock HTTP requests, unit testing web layer endpoints, verifying REST controllers in isolation.

## Instructions

1. **Setup standalone MockMvc**: `MockMvcBuilders.standaloneSetup(controller)` for isolated testing
2. **Mock service dependencies**: Use `@Mock` for all services, `@InjectMocks` for the controller
3. **Test HTTP methods**: GET, POST, PUT, PATCH, DELETE with correct status codes
4. **Validate responses**: JsonPath assertions for JSON, content matchers for body
5. **Test validation**: Send invalid input, verify 400 status with error details
6. **Test errors**: Verify 404, 400, 401, 403, 500 for appropriate conditions
7. **Validate headers**: Both request (Authorization) and response headers
8. **Test content negotiation**: Different Accept and Content-Type headers

### Validation Workflow

```
Run test → If fails: add .andDo(print()) → Check actual vs expected → Fix assertion
```

## Examples

### Maven / Gradle Dependencies

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```

### Basic Pattern: GET Endpoint

```java
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@ExtendWith(MockitoExtension.class)
class UserControllerTest {

  @Mock
  private UserService userService;

  @InjectMocks
  private UserController userController;

  private MockMvc mockMvc;

  @BeforeEach
  void setUp() {
    mockMvc = MockMvcBuilders.standaloneSetup(userController).build();
  }

  @Test
  void shouldReturnAllUsers() throws Exception {
    List<UserDto> users = List.of(new UserDto(1L, "Alice"), new UserDto(2L, "Bob"));
    when(userService.getAllUsers()).thenReturn(users);

    mockMvc.perform(get("/api/users"))
      .andExpect(status().isOk())
      .andExpect(jsonPath("$[0].id").value(1))
      .andExpect(jsonPath("$[0].name").value("Alice"));

    verify(userService, times(1)).getAllUsers();
  }

  @Test
  void shouldReturn404WhenUserNotFound() throws Exception {
    when(userService.getUserById(999L))
      .thenThrow(new UserNotFoundException("User not found"));

    mockMvc.perform(get("/api/users/999"))
      .andExpect(status().isNotFound());

    verify(userService).getUserById(999L);
  }
}
```

### POST: Create Resource

```java
@Test
void shouldCreateUserAndReturn201() throws Exception {
  UserDto createdUser = new UserDto(1L, "Alice", "alice@example.com");
  when(userService.createUser(any())).thenReturn(createdUser);

  mockMvc.perform(post("/api/users")
      .contentType("application/json")
      .content("{\"name\":\"Alice\",\"email\":\"alice@example.com\"}"))
    .andExpect(status().isCreated())
    .andExpect(jsonPath("$.id").value(1))
    .andExpect(jsonPath("$.name").value("Alice"));

  verify(userService).createUser(any(UserCreateRequest.class));
}
```

### PUT: Update Resource

```java
@Test
void shouldUpdateUserAndReturn200() throws Exception {
  UserDto updatedUser = new UserDto(1L, "Updated");
  when(userService.updateUser(eq(1L), any())).thenReturn(updatedUser);

  mockMvc.perform(put("/api/users/1")
      .contentType("application/json")
      .content("{\"name\":\"Updated\"}"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.name").value("Updated"));

  verify(userService).updateUser(eq(1L), any());
}
```

### DELETE: Remove Resource

```java
@Test
void shouldDeleteUserAndReturn204() throws Exception {
  doNothing().when(userService).deleteUser(1L);

  mockMvc.perform(delete("/api/users/1"))
    .andExpect(status().isNoContent());

  verify(userService).deleteUser(1L);
}
```

### Query Parameters

```java
@Test
void shouldFilterUsersByName() throws Exception {
  when(userService.searchUsers("Alice")).thenReturn(List.of(new UserDto(1L, "Alice")));

  mockMvc.perform(get("/api/users/search").param("name", "Alice"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$[0].name").value("Alice"));

  verify(userService).searchUsers("Alice");
}
```

### Path Variables

```java
@Test
void shouldGetUserByIdFromPath() throws Exception {
  when(userService.getUserById(123L)).thenReturn(new UserDto(123L, "Alice"));

  mockMvc.perform(get("/api/users/{id}", 123L))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.id").value(123));
}
```

### Validation Errors (400)

```java
@Test
void shouldReturn400WhenRequestBodyInvalid() throws Exception {
  mockMvc.perform(post("/api/users")
      .contentType("application/json")
      .content("{\"name\":\"\"}"))
    .andExpect(status().isBadRequest())
    .andExpect(jsonPath("$.errors").isArray());
}
```

### Response Headers

```java
@Test
void shouldReturnCustomHeaders() throws Exception {
  when(userService.getAllUsers()).thenReturn(List.of());

  mockMvc.perform(get("/api/users"))
    .andExpect(status().isOk())
    .andExpect(header().exists("X-Total-Count"))
    .andExpect(header().string("X-Total-Count", "0"));
}
```

### Authorization Header

```java
@Test
void shouldRequireAuthorizationHeader() throws Exception {
  mockMvc.perform(get("/api/users"))
    .andExpect(status().isUnauthorized());

  mockMvc.perform(get("/api/users").header("Authorization", "Bearer token"))
    .andExpect(status().isOk());
}
```

### Content Negotiation

```java
@Test
void shouldReturnJsonWhenAcceptHeaderIsJson() throws Exception {
  when(userService.getUserById(1L)).thenReturn(new UserDto(1L, "Alice"));

  mockMvc.perform(get("/api/users/1").accept("application/json"))
    .andExpect(status().isOk())
    .andExpect(content().contentType("application/json"));
}
```

## Best Practices

- Use `standaloneSetup()` for isolated controller testing
- Mock service layer — controllers handle HTTP, services handle business logic
- Verify mock interactions: `verify(service).method(args)`
- Test happy path AND error scenarios (404, 400, 500)
- Use `jsonPath()` for fluent JSON assertions
- One focused assertion per test method

## Constraints and Warnings

- Controller tests verify HTTP handling only — not full request flow
- `standaloneSetup()` may not support `@Validated` without full context
- JsonPath requires valid JSON in response body
- `@PreAuthorize`/`@Secured` need additional setup — consider separate security tests
- File uploads require `MockMultipartFile`

## References

- [Spring MockMvc Documentation](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/servlet/MockMvc.html)
- [Spring Testing Best Practices](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)

---

## Source references (concatenated)

### advanced.md

# Advanced Controller Testing Patterns

## Multiple Status Code Scenarios

```java
@Test
void shouldReturnDifferentStatusCodesForDifferentScenarios() throws Exception {
  // Successful response
  when(userService.getUserById(1L)).thenReturn(new UserDto(1L, "Alice"));
  mockMvc.perform(get("/api/users/1"))
    .andExpect(status().isOk());

  // Not found
  when(userService.getUserById(999L))
    .thenThrow(new UserNotFoundException("Not found"));
  mockMvc.perform(get("/api/users/999"))
    .andExpect(status().isNotFound());

  // Unauthorized
  mockMvc.perform(get("/api/admin/users"))
    .andExpect(status().isUnauthorized());
}
```

## Security Testing

### Testing Role-Based Access

```java
@Test
void shouldReturn403WhenUserLacksRole() throws Exception {
  mockMvc.perform(delete("/api/admin/users/1"))
    .andExpect(status().isForbidden());
}
```

### Testing Authentication

```java
@Test
void shouldReturn401WhenNoTokenProvided() throws Exception {
  mockMvc.perform(get("/api/protected"))
    .andExpect(status().isUnauthorized());
}

@Test
void shouldReturn200WhenValidTokenProvided() throws Exception {
  when(authService.validateToken("valid-token")).thenReturn(true);
  when(userService.getCurrentUser()).thenReturn(new UserDto(1L, "Alice"));

  mockMvc.perform(get("/api/protected")
      .header("Authorization", "Bearer valid-token"))
    .andExpect(status().isOk());
}
```

## File Upload Testing

### MockMultipartFile

```java
@Test
void shouldUploadFileSuccessfully() throws Exception {
  byte[] fileContent = "test file content".getBytes();
  MockMultipartFile file = new MockMultipartFile(
    "file", "test.txt", "text/plain", fileContent);

  when(fileService.store(any())).thenReturn("stored-file-id");

  mockMvc.perform(multipart("/api/files/upload").file(file))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.fileId").value("stored-file-id"));

  verify(fileService).store(any(MultipartFile.class));
}
```

## Pagination

### Testing Paginated Responses

```java
@Test
void shouldReturnPaginatedUsers() throws Exception {
  Page<UserDto> page = new PageImpl<>(
    List.of(new UserDto(1L, "Alice"), new UserDto(2L, "Bob")),
    PageRequest.of(0, 10), 2);

  when(userService.getUsers(any(Pageable.class))).thenReturn(page);

  mockMvc.perform(get("/api/users")
      .param("page", "0")
      .param("size", "10"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.content").isArray())
    .andExpect(jsonPath("$.content.length()").value(2))
    .andExpect(jsonPath("$.totalElements").value(2))
    .andExpect(jsonPath("$.totalPages").value(1));
}
```

## Exception Handling

### Custom Exception Handlers

```java
@Test
void shouldHandleValidationException() throws Exception {
  when(userService.createUser(any()))
    .thenThrow(new MethodArgumentNotValidException(
      null, new BeanPropertyBindingResult(null, "user")));

  mockMvc.perform(post("/api/users")
      .contentType("application/json")
      .content("{\"invalid\":\"data\"}"))
    .andExpect(status().isBadRequest())
    .andExpect(jsonPath("$.errors").isArray());
}
```

## Testing Async Endpoints

```java
@Test
void shouldHandleAsyncResponses() throws Exception {
  CompletableFuture<UserDto> futureUser = CompletableFuture.completedFuture(
    new UserDto(1L, "Alice"));
  when(userService.getUserAsync(1L)).thenReturn(futureUser);

  MvcResult result = mockMvc.perform(get("/api/users/async/1"))
    .andExpect(status().isOk())
    .andReturn();

  // For actual async testing, use withAsyncDispatch()
  String content = result.getResponse().getContentAsString();
  assertThat(content).contains("Alice");
}
```

### common-pitfalls.md

# Common Pitfalls in Controller Testing

## Testing Business Logic in Controller

**Don't**: Test business logic in controller tests

**Do**: Keep controller tests focused on HTTP handling only

```java
// BAD - Testing business logic
@Test
void shouldCalculateTotalCorrectly() throws Exception {
  // This should be in service tests
  mockMvc.perform(get("/api/orders/1/total"))
    .andExpect(jsonPath("$.total").value(150.00));
}

// GOOD - Testing HTTP response
@Test
void shouldReturnOrderTotalAsJson() throws Exception {
  when(orderService.getOrderTotal(1L)).thenReturn(150.00);

  mockMvc.perform(get("/api/orders/1/total"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.total").exists());

  verify(orderService).getOrderTotal(1L);
}
```

## Not Mocking Service Layer

**Don't**: Leave service layer unmocked

**Do**: Always mock service dependencies

```java
// BAD - Unmocked service
@InjectMocks
private UserController userController;  // userService is null!

// GOOD - Mocked service
@Mock
private UserService userService;

@InjectMocks
private UserController userController;
```

## Testing Framework Behavior

**Don't**: Test Spring MVC framework internals

**Do**: Test your controller code only

```java
// BAD - Testing Spring behavior
mockMvc.perform(get("/api/users"))
  .andExpect(handler().methodName("getUsers"))  // Don't test this
  .andExpect(handler().controllerClass(UserController.class));

// GOOD - Testing your code
mockMvc.perform(get("/api/users"))
  .andExpect(status().isOk())
  .andExpect(jsonPath("$").isArray());
```

## Hardcoding URLs

**Don't**: Use string URLs throughout tests

**Do**: Use controller's `@RequestMapping` values or constants

```java
// BAD
mockMvc.perform(get("/users/1")).andExpect(status().isOk());
mockMvc.perform(post("/users")).andExpect(status().isCreated());

// GOOD - Extract to constants
private static final String BASE_URL = "/api/users";

@Test
void shouldGetUser() throws Exception {
  mockMvc.perform(get(BASE_URL + "/1")).andExpect(status().isOk());
}
```

## Not Verifying Mock Interactions

**Don't**: Forget to verify service was called

**Do**: Always verify mock interactions

```java
// BAD - No verification
@Test
void shouldReturnUser() throws Exception {
  when(userService.getUserById(1L)).thenReturn(new UserDto(1L, "Alice"));
  mockMvc.perform(get("/api/users/1"))
    .andExpect(status().isOk());
  // Service might not have been called at all!
}

// GOOD - With verification
@Test
void shouldReturnUser() throws Exception {
  when(userService.getUserById(1L)).thenReturn(new UserDto(1L, "Alice"));
  mockMvc.perform(get("/api/users/1"))
    .andExpect(status().isOk());

  verify(userService).getUserById(1L);  // Verify it was called
}
```

## Missing Error Path Tests

**Don't**: Only test happy paths

**Do**: Test error scenarios too

```java
// BAD - Only happy path
@Test
void shouldReturnUser() throws Exception {
  when(userService.getUserById(1L)).thenReturn(new UserDto(1L, "Alice"));
  mockMvc.perform(get("/api/users/1"))
    .andExpect(status().isOk());
}

// GOOD - Happy path AND error path
@Test
void shouldReturnUser() throws Exception {
  when(userService.getUserById(1L)).thenReturn(new UserDto(1L, "Alice"));
  mockMvc.perform(get("/api/users/1"))
    .andExpect(status().isOk());
}

@Test
void shouldReturn404WhenUserNotFound() throws Exception {
  when(userService.getUserById(999L))
    .thenThrow(new UserNotFoundException("Not found"));
  mockMvc.perform(get("/api/users/999"))
    .andExpect(status().isNotFound());
}
```

## Incorrect JSON Comparison

**Don't**: Use exact string comparison for JSON

**Do**: Use JsonPath for flexible assertions

```java
// BAD - Brittle
.andExpect(content().string(containsString("{\"id\":1,\"name\":\"Alice\"}")));

// GOOD - Flexible
.andExpect(jsonPath("$.id").value(1))
.andExpect(jsonPath("$.name").value("Alice"));
```

## Forgotten Setup Method

**Don't**: Initialize MockMvc in each test

**Do**: Use `@ BeforeEach` for shared setup

```java
// BAD - Repetitive
@Test
void shouldGetUser() throws Exception {
  MockMvc mockMvc = MockMvcBuilders.standaloneSetup(userController).build();
  ...
}

// GOOD - Shared setup
private MockMvc mockMvc;

@BeforeEach
void setUp() {
  mockMvc = MockMvcBuilders.standaloneSetup(userController).build();
}
```

### troubleshooting.md

# Troubleshooting Controller Tests

## Common Issues and Solutions

### Content Type Mismatch

**Symptom**: `415 Unsupported Media Type` error

**Solution**: Ensure `contentType()` matches controller's `@PostMapping(consumes=...)`

```java
// Wrong
mockMvc.perform(post("/api/users").content("{}"));

// Correct
mockMvc.perform(post("/api/users")
    .contentType("application/json")
    .content("{}"));
```

### JsonPath Not Matching

**Symptom**: JsonPath assertions fail without clear reason

**Solution**: Use `.andDo(print())` to see actual response

```java
mockMvc.perform(get("/api/users/1"))
  .andDo(print())  // Add this to debug
  .andExpect(jsonPath("$.name").value("Alice"));
```

### Status Code Assertions Fail

**Symptom**: Expected 200 but got 404/500

**Solution**:
1. Check controller `@RequestMapping` paths match test URLs
2. Verify error handling returns expected status
3. Ensure service mocks are set up before request

```java
// Verify controller path
@RequestMapping("/api/users")  // Matches "/api/users" not "/users"

// Verify mock setup order
when(userService.getUserById(1L)).thenReturn(user);  // Before perform
mockMvc.perform(get("/api/users/1"))...
```

### Mock Not Being Called

**Symptom**: `verify()` fails with "never invoked"

**Solution**:
1. Check mock setup happens before `perform()`
2. Verify mock parameters match actual call parameters
3. Use `any()` for flexible matching

```java
// Use any() for flexible matching
when(userService.createUser(any())).thenReturn(user);
verify(userService).createUser(any());  // Not exact argument
```

### Response Body Empty

**Symptom**: JsonPath finds nothing in response

**Solution**:
1. Verify controller returns response body
2. Check `@ResponseBody` or `@RestController` annotation
3. Ensure object serialization works

```java
// Controller should return the object
@GetMapping("/{id}")
public UserDto getUser(@PathVariable Long id) {
  return userService.getUserById(id);  // Return, not void
}
```

### ObjectMapper Issues

**Symptom**: JSON serialization errors or null values

**Solution**: Configure ObjectMapper in standalone setup

```java
Jackson2ObjectMapperBuilder mapperBuilder = Jackson2ObjectMapperBuilder.json();
mockMvc = MockMvcBuilders.standaloneSetup(controller)
  .setMessageConverters(new MappingJackson2HttpMessageConverter(mapperBuilder.build()))
  .build();
```

### Validation Not Working

**Symptom**: Validation constraints ignored, 500 instead of 400

**Solution**: Add validator to standalone setup

```java
@BeforeEach
void setUp() {
  MockitoAnnotations.openMocks(this);
  mockMvc = MockMvcBuilders.standaloneSetup(userController)
    .setValidator(validator)  // Add local validator
    .build();
}
```

### NullPointerException in Controller

**Symptom**: Tests pass locally but fail in CI

**Solution**: Ensure all controller dependencies are mocked

```java
@Mock
private UserService userService;

@Mock
private ValidationService validationService;  // Don't forget this one

@InjectMocks
private UserController userController;
```

