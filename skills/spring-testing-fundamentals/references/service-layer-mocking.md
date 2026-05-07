# Service-Layer Mocking Patterns (absorbed from unit-test-service-layer)

---
name: unit-test-service-layer
description: Provides patterns for unit testing service layer with Mockito. Creates isolated tests that mock repository calls, verify method invocations, test exception scenarios, and stub external API responses. Use when testing service behaviors and business logic without database or external services.
allowed-tools: Read, Write, Bash, Glob, Grep
---

# Unit Testing Service Layer with Mockito

## Overview

Provides patterns for unit testing `@Service` classes using Mockito. Mocks repository calls, verifies method invocations, tests exception scenarios, and stubs external API responses. Enables fast, isolated tests without Spring container or database.

## When to Use

- Testing business logic in `@Service` classes
- Mocking repository and external client dependencies
- Verifying service interactions with mocked collaborators
- Testing error handling and edge cases in services
- Writing fast, isolated unit tests (no database, no API calls)

## Instructions

Follow this workflow to test service layer with Mockito, including validation checkpoints:

### 1. Setup Test Class

Use `@ExtendWith(MockitoExtension.class)` to enable Mockito annotations.

### 2. Declare Mocks with `@Mock` and `@InjectMocks`

Use `@Mock` for dependencies (repositories, clients) and `@InjectMocks` for the service under test.

### 3. Arrange-Act-Assert with Validation

**Arrange**: Create test data and configure mock return values using `when().thenReturn()`.

**Act**: Execute the service method being tested.

**Assert**:
- Verify returned values with AssertJ assertions
- Verify mock interactions with `verify()`
- **Validation checkpoint**: Run test and confirm green bar

### 4. Test Exception Scenarios

Configure mocks to throw exceptions with `when().thenThrow()`.

**Validation checkpoint**: Verify exception type and message

### 5. Verify Complete Coverage

- Run full test suite: `mvn test` or `gradle test`
- Check coverage report: `mvn test jacoco:report`
- **Validation checkpoint**: Confirm all service methods have corresponding tests

## Examples

### Basic Service Test Pattern

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

  @Mock
  private UserRepository userRepository;

  @InjectMocks
  private UserService userService;

  @Test
  void shouldReturnUserWhenFound() {
    // Arrange
    User expected = new User(1L, "Alice");
    when(userRepository.findById(1L)).thenReturn(Optional.of(expected));

    // Act
    User result = userService.getUser(1L);

    // Assert
    assertThat(result.getName()).isEqualTo("Alice");
    verify(userRepository).findById(1L);
  }

  @Test
  void shouldThrowWhenUserNotFound() {
    // Arrange
    when(userRepository.findById(999L)).thenReturn(Optional.empty());

    // Act & Assert
    assertThatThrownBy(() -> userService.getUser(999L))
      .isInstanceOf(UserNotFoundException.class);
  }
}
```

### Verify Method Invocations

```java
@Test
void shouldSendEmailOnUserCreation() {
  User newUser = new User(1L, "Alice", "alice@example.com");
  when(userRepository.save(any(User.class))).thenReturn(newUser);

  enrichmentService.registerNewUser("Alice", "alice@example.com");

  verify(userRepository).save(any(User.class));
  verify(emailService).sendWelcomeEmail("alice@example.com");
}
```

For additional patterns (multiple dependencies, argument captors, async services, InOrder verification), see `references/examples.md`.

## Best Practices

- **Use `@ExtendWith(MockitoExtension.class)`** for JUnit 5 integration
- **Mock only direct dependencies** of the service under test
- **Verify interactions** to ensure correct collaboration
- **Test one behavior per test method** - keep tests focused
- **Use descriptive variable names**: `expectedUser`, `actualUser`, `captor`
- **Create real instances** for value objects and DTOs (don't mock them)

## Constraints and Warnings

- Do not mock value objects or DTOs; create real instances with test data.
- Avoid mocking too many dependencies; consider refactoring if a service has too many collaborators.
- Tests must be independent; do not rely on execution order.
- Be cautious with `@Spy`; partial mocking is harder to understand and maintain.
- Do not test private methods directly; test them through public method behavior.
- Argument matchers (`any()`, `eq()`) cannot be mixed with actual values in the same stub.
- Avoid over-verifying; verify only interactions important to the test scenario.

## References

- [Mockito Documentation](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html)
- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [AssertJ Assertions](https://assertj.github.io/assertj-core-features-highlight.html)

---

## Source references (concatenated)

### examples.md

# Unit Testing Service Layer - Examples

Complete code examples for unit testing `@Service` classes with Mockito.

## Setup with Mockito and JUnit 5

### Maven
```xml
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-core</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-junit-jupiter</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.assertj</groupId>
  <artifactId>assertj-core</artifactId>
  <scope>test</scope>
</dependency>
```

### Gradle
```kotlin
dependencies {
  testImplementation("org.junit.jupiter:junit-jupiter")
  testImplementation("org.mockito:mockito-core")
  testImplementation("org.mockito:mockito-junit-jupiter")
  testImplementation("org.assertj:assertj-core")
}
```

## Basic Pattern: Service with Mocked Dependencies

### Single Dependency

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

  @Mock
  private UserRepository userRepository;

  @InjectMocks
  private UserService userService;

  @Test
  void shouldReturnAllUsers() {
    // Arrange
    List<User> expectedUsers = List.of(
      new User(1L, "Alice"),
      new User(2L, "Bob")
    );
    when(userRepository.findAll()).thenReturn(expectedUsers);

    // Act
    List<User> result = userService.getAllUsers();

    // Assert
    assertThat(result).hasSize(2);
    assertThat(result).containsExactly(
      new User(1L, "Alice"),
      new User(2L, "Bob")
    );
    verify(userRepository, times(1)).findAll();
  }
}
```

### Multiple Dependencies

```java
@ExtendWith(MockitoExtension.class)
class UserEnrichmentServiceTest {

  @Mock
  private UserRepository userRepository;

  @Mock
  private EmailService emailService;

  @Mock
  private AnalyticsClient analyticsClient;

  @InjectMocks
  private UserEnrichmentService enrichmentService;

  @Test
  void shouldCreateUserAndSendWelcomeEmail() {
    User newUser = new User(1L, "Alice", "alice@example.com");
    when(userRepository.save(any(User.class))).thenReturn(newUser);
    doNothing().when(emailService).sendWelcomeEmail(newUser.getEmail());

    User result = enrichmentService.registerNewUser("Alice", "alice@example.com");

    assertThat(result.getId()).isEqualTo(1L);
    assertThat(result.getName()).isEqualTo("Alice");

    verify(userRepository).save(any(User.class));
    verify(emailService).sendWelcomeEmail("alice@example.com");
    verify(analyticsClient, never()).trackUserRegistration(any());
  }
}
```

## Testing Exception Handling

### Service Throws Expected Exception

```java
@Test
void shouldThrowExceptionWhenUserNotFound() {
  when(userRepository.findById(999L))
    .thenThrow(new UserNotFoundException("User not found"));

  assertThatThrownBy(() -> userService.getUserDetails(999L))
    .isInstanceOf(UserNotFoundException.class)
    .hasMessageContaining("User not found");

  verify(userRepository).findById(999L);
}

@Test
void shouldRethrowRepositoryException() {
  when(userRepository.findAll())
    .thenThrow(new DataAccessException("Database connection failed"));

  assertThatThrownBy(() -> userService.getAllUsers())
    .isInstanceOf(DataAccessException.class)
    .hasMessageContaining("Database connection failed");
}
```

## Testing Complex Workflows

### Multiple Service Method Calls

```java
@Test
void shouldTransferMoneyBetweenAccounts() {
  Account fromAccount = new Account(1L, 1000.0);
  Account toAccount = new Account(2L, 500.0);

  when(accountRepository.findById(1L)).thenReturn(Optional.of(fromAccount));
  when(accountRepository.findById(2L)).thenReturn(Optional.of(toAccount));
  when(accountRepository.save(any(Account.class)))
    .thenAnswer(invocation -> invocation.getArgument(0));

  moneyTransferService.transfer(1L, 2L, 200.0);

  // Verify both accounts were updated
  verify(accountRepository, times(2)).save(any(Account.class));
  assertThat(fromAccount.getBalance()).isEqualTo(800.0);
  assertThat(toAccount.getBalance()).isEqualTo(700.0);
}
```

## Argument Capturing and Verification

### Capture Arguments Passed to Mock

```java
import org.mockito.ArgumentCaptor;

@Test
void shouldCaptureUserDataWhenSaving() {
  ArgumentCaptor<User> userCaptor = ArgumentCaptor.forClass(User.class);
  when(userRepository.save(any(User.class)))
    .thenAnswer(invocation -> invocation.getArgument(0));

  userService.createUser("Alice", "alice@example.com");

  verify(userRepository).save(userCaptor.capture());
  User capturedUser = userCaptor.getValue();

  assertThat(capturedUser.getName()).isEqualTo("Alice");
  assertThat(capturedUser.getEmail()).isEqualTo("alice@example.com");
}

@Test
void shouldCaptureMultipleArgumentsAcrossMultipleCalls() {
  ArgumentCaptor<User> userCaptor = ArgumentCaptor.forClass(User.class);

  userService.createUser("Alice", "alice@example.com");
  userService.createUser("Bob", "bob@example.com");

  verify(userRepository, times(2)).save(userCaptor.capture());

  List<User> capturedUsers = userCaptor.getAllValues();
  assertThat(capturedUsers).hasSize(2);
  assertThat(capturedUsers.get(0).getName()).isEqualTo("Alice");
  assertThat(capturedUsers.get(1).getName()).isEqualTo("Bob");
}
```

## Verification Patterns

### Verify Call Order and Frequency

```java
import org.mockito.InOrder;

@Test
void shouldCallMethodsInCorrectOrder() {
  InOrder inOrder = inOrder(userRepository, emailService);

  userService.registerNewUser("Alice", "alice@example.com");

  inOrder.verify(userRepository).save(any(User.class));
  inOrder.verify(emailService).sendWelcomeEmail(any());
}

@Test
void shouldCallMethodExactlyOnce() {
  userService.getUserDetails(1L);

  verify(userRepository, times(1)).findById(1L);
  verify(userRepository, never()).findAll();
}
```

## Testing Async/Reactive Services

### Service with CompletableFuture

```java
@Test
void shouldReturnCompletableFutureWhenFetchingAsyncData() {
  List<User> users = List.of(new User(1L, "Alice"));
  when(userRepository.findAllAsync())
    .thenReturn(CompletableFuture.completedFuture(users));

  CompletableFuture<List<User>> result = userService.getAllUsersAsync();

  assertThat(result).isCompletedWithValue(users);
}
```

## Before/After Examples

### Input: Service Without Test Coverage

```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public User getUser(Long id) {
        return userRepository.findById(id).orElse(null);
    }
}
```

### Output: Service With Complete Test Coverage

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldReturnUserWhenFound() {
        User expectedUser = new User(1L, "Alice");
        when(userRepository.findById(1L)).thenReturn(Optional.of(expectedUser));

        User result = userService.getUser(1L);

        assertThat(result).isEqualTo(expectedUser);
        verify(userRepository).findById(1L);
    }

    @Test
    void shouldThrowExceptionWhenNotFound() {
        when(userRepository.findById(999L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> userService.getUser(999L))
            .isInstanceOf(UserNotFoundException.class);
    }
}
```

### Input: Manual Mock Creation (Anti-Pattern)

```java
UserService service = new UserService(new FakeUserRepository());
```

### Output: Mockito-Based Test

```java
@Mock
private UserRepository userRepository;

@InjectMocks
private UserService userService;

@Test
void test() {
    when(userRepository.findById(1L)).thenReturn(Optional.of(user));
    // Test logic
}
```

## Common Patterns

**Partial Mock with Spy**:
```java
@Spy
@InjectMocks
private UserService userService; // Real instance, but can stub some methods

@Test
void shouldUseRealMethodButMockDependency() {
  when(userRepository.findById(any())).thenReturn(Optional.of(new User()));
  // Calls real userService methods but userRepository is mocked
}
```

**Constructor Injection for Testing**:
```java
// In your service (production code)
public class UserService {
  private final UserRepository userRepository;

  public UserService(UserRepository userRepository) {
    this.repository = userRepository;
  }
}

// In your test - can inject mocks directly
@Test
void test() {
  UserRepository mockRepo = mock(UserRepository.class);
  UserService service = new UserService(mockRepo);
}
```

## Troubleshooting

**UnfinishedStubbingException**: Ensure all `when()` calls are completed with `thenReturn()`, `thenThrow()`, or `thenAnswer()`.

**UnnecessaryStubbingException**: Remove unused stub definitions. Use `@ExtendWith(MockitoExtension.class)` with `MockitoExtension.LENIENT` if you intentionally have unused stubs.

**NullPointerException in test**: Verify `@InjectMocks` correctly injects all mocked dependencies into the service constructor.

