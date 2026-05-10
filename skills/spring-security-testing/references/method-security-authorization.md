# Method Security & Authorization Testing (absorbed from unit-test-security-authorization)

---
name: unit-test-security-authorization
description: Provides patterns for unit testing Spring Security with `@PreAuthorize`, `@Secured`, `@RolesAllowed`. Validates role-based access control and authorization policies. Use when testing security configurations and access control logic.
allowed-tools: Read, Write, Bash, Glob, Grep
---

# Unit Testing Security and Authorization

## Overview

This skill provides patterns for unit testing Spring Security authorization logic using `@PreAuthorize`, `@Secured`, `@RolesAllowed`, and custom permission evaluators. It covers testing role-based access control (RBAC), expression-based authorization, custom permission evaluators, and verifying access denied scenarios without full Spring Security context.

## When to Use

Use this skill when:
- Testing `@PreAuthorize` and `@Secured` method-level security
- Testing role-based access control (RBAC)
- Testing custom permission evaluators
- Verifying access denied scenarios
- Testing authorization with authenticated principals
- Want fast authorization tests without full Spring Security context

## Instructions

Follow these steps to test Spring Security authorization:

### 1. Set Up Security Testing Dependencies
Add spring-security-test to your test dependencies:

```xml
<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-test</artifactId>
  <scope>test</scope>
</dependency>
```

### 2. Enable Method Security in Test Configuration
```java
@Configuration
@EnableMethodSecurity
class TestSecurityConfig { }
```

### 3. Test with `@WithMockUser`
```java
@Test
@WithMockUser(roles = "ADMIN")
void shouldAllowAdminAccess() {
  assertThatCode(() -> service.deleteUser(1L))
    .doesNotThrowAnyException();
}

@Test
@WithMockUser(roles = "USER")
void shouldDenyUserAccess() {
  assertThatThrownBy(() -> service.deleteUser(1L))
    .isInstanceOf(AccessDeniedException.class);
}
```

### 4. Test Custom Permission Evaluators
```java
@Test
void shouldGrantPermissionToOwner() {
  Authentication auth = new UsernamePasswordAuthenticationToken(
    "alice", null, List.of(new SimpleGrantedAuthority("ROLE_USER"))
  );
  Document doc = new Document(1L, "Test", new User("alice"));

  boolean result = evaluator.hasPermission(auth, doc, "WRITE");
  assertThat(result).isTrue();
}
```

### 5. Validate Security is Active
If tests pass unexpectedly, add this assertion to verify security is enforced:
```java
@Test
void shouldRejectUnauthorizedWhenSecurityEnabled() {
  assertThatThrownBy(() -> service.deleteUser(1L))
    .isInstanceOf(AccessDeniedException.class);
}
```

## Quick Reference

| Annotation | Description | Example |
|------------|-------------|---------|
| `@PreAuthorize` | Pre-invocation authorization | `@PreAuthorize("hasRole('ADMIN')")` |
| `@PostAuthorize` | Post-invocation authorization | `@PostAuthorize("returnObject.owner == authentication.name")` |
| `@Secured` | Simple role-based security | `@Secured("ROLE_ADMIN")` |
| `@RolesAllowed` | JSR-250 standard | `@RolesAllowed({"ADMIN", "MANAGER"})` |
| `@WithMockUser` | Test annotation | `@WithMockUser(roles = "ADMIN")` |

## Examples

### Basic `@PreAuthorize` Test

```java
@Service
public class UserService {
  @PreAuthorize("hasRole('ADMIN')")
  public void deleteUser(Long userId) {
    // delete logic
  }
}

// Test
@Test
@WithMockUser(roles = "ADMIN")
void shouldAllowAdminToDeleteUser() {
  assertThatCode(() -> service.deleteUser(1L))
    .doesNotThrowAnyException();
}

@Test
@WithMockUser(roles = "USER")
void shouldDenyUserFromDeletingUser() {
  assertThatThrownBy(() -> service.deleteUser(1L))
    .isInstanceOf(AccessDeniedException.class);
}
```

### Expression-Based Security Test

```java
@PreAuthorize("#userId == authentication.principal.id")
public UserProfile getUserProfile(Long userId) {
  // get profile
}

// For custom principal properties, use @WithUserDetails with a custom UserDetailsService
@Test
@WithUserDetails("alice")
void shouldAllowUserToAccessOwnProfile() {
  assertThatCode(() -> service.getUserProfile(1L))
    .doesNotThrowAnyException();
}
```

> **Validation tip**: If a security test passes unexpectedly, verify that `@EnableMethodSecurity` is active on the test configuration — a missing annotation causes all `@PreAuthorize` checks to be bypassed silently.

See [references/basic-testing.md](references/basic-testing.md) for more basic patterns and [references/advanced-authorization.md](references/advanced-authorization.md) for complex expressions and custom evaluators.

## Best Practices

1. **Use `@WithMockUser`** for setting authenticated user context
2. **Test both allow and deny cases** for each security rule
3. **Test with different roles** to verify role-based decisions
4. **Test expression-based security** comprehensively
5. **Mock external dependencies** (permission evaluators, etc.)
6. **Test anonymous access separately** from authenticated access
7. **Use `@EnableGlobalMethodSecurity`** in configuration for method-level security

## Common Pitfalls

- Forgetting to enable method security in test configuration
- Not testing both allow and deny scenarios
- Testing framework code instead of authorization logic
- Not handling null authentication in tests
- Mixing authentication and authorization tests unnecessarily

## Constraints and Warnings

- **Method security requires proxy**: `@PreAuthorize` works via proxies; direct method calls bypass security
- **`@EnableGlobalMethodSecurity`**: Must be enabled for `@PreAuthorize`, `@Secured` to work
- **Role prefix**: Spring adds "ROLE_" prefix automatically; use `hasRole('ADMIN')` not `hasRole('ROLE_ADMIN')`
- **Authentication context**: Security context is thread-local; be careful with async tests
- **`@WithMockUser` limitations**: Creates a simple Authentication; complex auth scenarios need custom setup
- **SpEL expressions**: Complex SpEL in `@PreAuthorize` can be difficult to debug; test thoroughly
- **Performance impact**: Method security adds overhead; consider security at layer boundaries

## References

### Setup and Configuration
- **[references/setup.md](references/setup.md)** - Maven dependencies and security configuration

### Testing Patterns
- **[references/basic-testing.md](references/basic-testing.md)** - Basic patterns for `@PreAuthorize`, `@Secured`, MockMvc testing, and parameterized tests

### Advanced Topics
- **[references/advanced-authorization.md](references/advanced-authorization.md)** - Expression-based authorization, custom permission evaluators, SpEL expressions

### Complete Examples
- **[references/complete-examples.md](references/complete-examples.md)** - Before/after examples showing transition from manual to declarative security

---

## Source references (concatenated)

### advanced-authorization.md

# Advanced Authorization Testing

## Testing Expression-Based Authorization

### Complex Permission Expressions

```java
@Service
public class DocumentService {

  @PreAuthorize("hasRole('ADMIN') or authentication.principal.username == #owner")
  public Document getDocument(String owner, Long docId) {
    // get document
  }

  @PreAuthorize("hasPermission(#docId, 'Document', 'WRITE')")
  public void updateDocument(Long docId, String content) {
    // update logic
  }

  @PreAuthorize("#userId == authentication.principal.id")
  public UserProfile getUserProfile(Long userId) {
    // get profile
  }
}
```

### Tests

```java
class ExpressionBasedSecurityTest {

  @Test
  @WithMockUser(username = "alice", roles = "ADMIN")
  void shouldAllowAdminToAccessAnyDocument() {
    DocumentService service = new DocumentService();

    assertThatCode(() -> service.getDocument("bob", 1L))
      .doesNotThrowAnyException();
  }

  @Test
  @WithMockUser(username = "alice")
  void shouldAllowOwnerToAccessOwnDocument() {
    DocumentService service = new DocumentService();

    assertThatCode(() -> service.getDocument("alice", 1L))
      .doesNotThrowAnyException();
  }

  @Test
  @WithMockUser(username = "alice")
  void shouldDenyUserAccessToOtherUserDocument() {
    DocumentService service = new DocumentService();

    assertThatThrownBy(() -> service.getDocument("bob", 1L))
      .isInstanceOf(AccessDeniedException.class);
  }

  @Test
  @WithMockUser(username = "alice", id = "1")
  void shouldAllowUserToAccessOwnProfile() {
    DocumentService service = new DocumentService();

    assertThatCode(() -> service.getUserProfile(1L))
      .doesNotThrowAnyException();
  }

  @Test
  @WithMockUser(username = "alice", id = "1")
  void shouldDenyUserAccessToOtherProfile() {
    DocumentService service = new DocumentService();

    assertThatThrownBy(() -> service.getUserProfile(999L))
      .isInstanceOf(AccessDeniedException.class);
  }
}
```

## Testing Custom Permission Evaluator

### Custom Permission Evaluator Implementation

```java
@Component
public class DocumentPermissionEvaluator implements PermissionEvaluator {

  private final DocumentRepository documentRepository;

  public DocumentPermissionEvaluator(DocumentRepository documentRepository) {
    this.documentRepository = documentRepository;
  }

  @Override
  public boolean hasPermission(Authentication authentication,
                               Object targetDomainObject,
                               Object permission) {
    if (authentication == null) return false;

    Document document = (Document) targetDomainObject;
    String userUsername = authentication.getName();

    return document.getOwner().getUsername().equals(userUsername) ||
           userHasRole(authentication, "ADMIN");
  }

  @Override
  public boolean hasPermission(Authentication authentication,
                               Serializable targetId,
                               String targetType,
                               Object permission) {
    if (authentication == null) return false;
    if (!"Document".equals(targetType)) return false;

    Document document = documentRepository.findById((Long) targetId).orElse(null);
    if (document == null) return false;

    return hasPermission(authentication, document, permission);
  }

  private boolean userHasRole(Authentication authentication, String role) {
    return authentication.getAuthorities().stream()
      .anyMatch(auth -> auth.getAuthority().equals("ROLE_" + role));
  }
}
```

### Unit Tests for Custom Evaluator

```java
class DocumentPermissionEvaluatorTest {

  private DocumentPermissionEvaluator evaluator;
  private DocumentRepository documentRepository;
  private Authentication adminAuth;
  private Authentication userAuth;
  private Document document;

  @BeforeEach
  void setUp() {
    documentRepository = mock(DocumentRepository.class);
    evaluator = new DocumentPermissionEvaluator(documentRepository);

    document = new Document(1L, "Test Doc", new User("alice"));

    adminAuth = new UsernamePasswordAuthenticationToken(
      "admin",
      null,
      List.of(new SimpleGrantedAuthority("ROLE_ADMIN"))
    );

    userAuth = new UsernamePasswordAuthenticationToken(
      "alice",
      null,
      List.of(new SimpleGrantedAuthority("ROLE_USER"))
    );
  }

  @Test
  void shouldGrantPermissionToDocumentOwner() {
    boolean hasPermission = evaluator.hasPermission(userAuth, document, "WRITE");

    assertThat(hasPermission).isTrue();
  }

  @Test
  void shouldDenyPermissionToNonOwner() {
    Authentication otherUserAuth = new UsernamePasswordAuthenticationToken(
      "bob",
      null,
      List.of(new SimpleGrantedAuthority("ROLE_USER"))
    );

    boolean hasPermission = evaluator.hasPermission(otherUserAuth, document, "WRITE");

    assertThat(hasPermission).isFalse();
  }

  @Test
  void shouldGrantPermissionToAdmin() {
    boolean hasPermission = evaluator.hasPermission(adminAuth, document, "WRITE");

    assertThat(hasPermission).isTrue();
  }

  @Test
  void shouldDenyNullAuthentication() {
    boolean hasPermission = evaluator.hasPermission(null, document, "WRITE");

    assertThat(hasPermission).isFalse();
  }

  @Test
  void shouldHandleDocumentNotFound() {
    when(documentRepository.findById(1L)).thenReturn(Optional.empty());

    boolean hasPermission = evaluator.hasPermission(adminAuth, 1L, "Document", "WRITE");

    assertThat(hasPermission).isFalse();
  }
}
```

## Common SpEL Expressions

### Authentication-Based Expressions

```java
// Check if user is authenticated
@PreAuthorize("isAuthenticated()")

// Check if user is anonymous
@PreAuthorize("isAnonymous()")

// Check if user is fully authenticated (not remember-me)
@PreAuthorize("isFullyAuthenticated()")

// Check if user has remember-me authentication
@PreAuthorize("hasPermission(#userId, 'read')")
```

### Role-Based Expressions

```java
// Has specific role (ROLE_ prefix added automatically)
@PreAuthorize("hasRole('ADMIN')")

// Has any of the specified roles
@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER', 'SUPERVISOR')")

// Does not have role
@PreAuthorize("!hasRole('GUEST')")
```

### Principal-Based Expressions

```java
// Access principal username
@PreAuthorize("authentication.principal.username == #username")

// Access principal properties
@PreAuthorize("authentication.principal.accountNonLocked")

// Check principal ID
@PreAuthorize("authentication.principal.id == #userId")
```

### Permission-Based Expressions

```java
// Check custom permission
@PreAuthorize("hasPermission(#objectId, 'READ')")

// Check permission with type
@PreAuthorize("hasPermission(#objectId, 'Document', 'WRITE')")

// Multiple permission checks
@PreAuthorize("hasPermission(#docId, 'READ') and hasPermission(#docId, 'WRITE')")
```

### Complex Expressions

```java
// OR condition
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")

// AND condition
@PreAuthorize("hasRole('ADMIN') and hasPermission(#docId, 'WRITE')")

// NOT condition
@PreAuthorize("hasRole('ADMIN') and !isBanned(#username)")

// Complex expression with parentheses
@PreAuthorize("(hasRole('ADMIN') or #isOwner) and !isLocked(#userId)")
```

## Testing `@PostAuthorize`

```java
@Service
public class MessageService {

  @PostAuthorize("returnObject.owner == authentication.principal.username")
  public Message getMessage(Long messageId) {
    // fetch and return message
  }
}

@Test
@WithMockUser(username = "alice")
void shouldAllowAccessToOwnMessage() {
  MessageService service = new MessageService();

  Message message = service.getMessage(1L);
  assertThat(message.getOwner()).isEqualTo("alice");
}

@Test
@WithMockUser(username = "alice")
void shouldDenyAccessToOtherMessage() {
  MessageService service = new MessageService();

  assertThatThrownBy(() -> service.getMessage(2L))
    .isInstanceOf(AccessDeniedException.class);
}
```

## Testing `@PostFilter` and `@PreFilter`

```java
@Service
public class DataService {

  @PreFilter("hasPermission(filterObject, 'READ')")
  public void processData(List<Data> items) {
    // items filtered before method execution
  }

  @PostFilter("hasPermission(filterObject, 'READ')")
  public List<Data> getAllData() {
    // results filtered after method execution
    return repository.findAll();
  }
}

@Test
@WithMockUser(roles = "ADMIN")
void shouldFilterDataBasedOnPermissions() {
  List<Data> input = List.of(data1, data2, data3);

  service.processData(input);

  // Verify only permitted items were processed
  verify(repository).saveAll(List.of(data1, data3));
}
```

### basic-testing.md

# Basic Testing Patterns

## Testing `@PreAuthorize` with Role-Based Access Control

### Service with Security Annotations

```java
@Service
public class UserService {

  @PreAuthorize("hasRole('ADMIN')")
  public void deleteUser(Long userId) {
    // delete logic
  }

  @PreAuthorize("hasRole('USER')")
  public User getCurrentUser() {
    // get user logic
  }

  @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
  public List<User> listAllUsers() {
    // list logic
  }
}
```

### Unit Tests

```java
import org.junit.jupiter.api.Test;
import org.springframework.security.test.context.support.WithMockUser;
import static org.assertj.core.api.Assertions.*;

class UserServiceSecurityTest {

  @Test
  @WithMockUser(roles = "ADMIN")
  void shouldAllowAdminToDeleteUser() {
    UserService service = new UserService();

    assertThatCode(() -> service.deleteUser(1L))
      .doesNotThrowAnyException();
  }

  @Test
  @WithMockUser(roles = "USER")
  void shouldDenyUserFromDeletingUser() {
    UserService service = new UserService();

    assertThatThrownBy(() -> service.deleteUser(1L))
      .isInstanceOf(AccessDeniedException.class);
  }

  @Test
  @WithMockUser(roles = "ADMIN")
  void shouldAllowAdminAndManagerToListUsers() {
    UserService service = new UserService();

    assertThatCode(() -> service.listAllUsers())
      .doesNotThrowAnyException();
  }

  @Test
  void shouldDenyAnonymousUserAccess() {
    UserService service = new UserService();

    assertThatThrownBy(() -> service.deleteUser(1L))
      .isInstanceOf(AccessDeniedException.class);
  }
}
```

## Testing `@Secured` Annotation

### Legacy Security Configuration

```java
@Service
public class OrderService {

  @Secured("ROLE_ADMIN")
  public Order approveOrder(Long orderId) {
    // approval logic
  }

  @Secured({"ROLE_ADMIN", "ROLE_MANAGER"})
  public List<Order> getOrders() {
    // get orders
  }
}
```

### Tests

```java
class OrderSecurityTest {

  @Test
  @WithMockUser(roles = "ADMIN")
  void shouldAllowAdminToApproveOrder() {
    OrderService service = new OrderService();

    assertThatCode(() -> service.approveOrder(1L))
      .doesNotThrowAnyException();
  }

  @Test
  @WithMockUser(roles = "USER")
  void shouldDenyUserFromApprovingOrder() {
    OrderService service = new OrderService();

    assertThatThrownBy(() -> service.approveOrder(1L))
      .isInstanceOf(AccessDeniedException.class);
  }
}
```

## Testing Controller Security with MockMvc

### Secure REST Endpoints

```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {

  @GetMapping("/users")
  @PreAuthorize("hasRole('ADMIN')")
  public List<UserDto> listAllUsers() {
    // logic
  }

  @DeleteMapping("/users/{id}")
  @PreAuthorize("hasRole('ADMIN')")
  public void deleteUser(@PathVariable Long id) {
    // delete logic
  }
}
```

### Tests with MockMvc

```java
import org.springframework.security.test.context.support.WithMockUser;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

class AdminControllerSecurityTest {

  private MockMvc mockMvc;

  @BeforeEach
  void setUp() {
    mockMvc = MockMvcBuilders
      .standaloneSetup(new AdminController())
      .apply(springSecurity())
      .build();
  }

  @Test
  @WithMockUser(roles = "ADMIN")
  void shouldAllowAdminToListUsers() throws Exception {
    mockMvc.perform(get("/api/admin/users"))
      .andExpect(status().isOk());
  }

  @Test
  @WithMockUser(roles = "USER")
  void shouldDenyUserFromListingUsers() throws Exception {
    mockMvc.perform(get("/api/admin/users"))
      .andExpect(status().isForbidden());
  }

  @Test
  void shouldDenyAnonymousAccessToAdminEndpoint() throws Exception {
    mockMvc.perform(get("/api/admin/users"))
      .andExpect(status().isUnauthorized());
  }

  @Test
  @WithMockUser(roles = "ADMIN")
  void shouldAllowAdminToDeleteUser() throws Exception {
    mockMvc.perform(delete("/api/admin/users/1"))
      .andExpect(status().isOk());
  }
}
```

## Testing Multiple Roles with Parameterized Tests

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

class RoleBasedAccessTest {

  private AdminService service;

  @BeforeEach
  void setUp() {
    service = new AdminService();
  }

  @ParameterizedTest
  @ValueSource(strings = {"ADMIN", "SUPER_ADMIN", "SYSTEM"})
  @WithMockUser(roles = "ADMIN")
  void shouldAllowPrivilegedRolesToDeleteUser(String role) {
    assertThatCode(() -> service.deleteUser(1L))
      .doesNotThrowAnyException();
  }

  @ParameterizedTest
  @ValueSource(strings = {"USER", "GUEST", "READONLY"})
  void shouldDenyUnprivilegedRolesToDeleteUser(String role) {
    assertThatThrownBy(() -> service.deleteUser(1L))
      .isInstanceOf(AccessDeniedException.class);
  }
}
```

## `@WithMockUser` Options

```java
// Basic usage
@WithMockUser
void testWithDefaultUser() { }

// With custom username
@WithMockUser(username = "alice")
void testWithCustomUsername() { }

// With roles
@WithMockUser(roles = "ADMIN")
void testWithRole() { }

// With multiple roles
@WithMockUser(roles = {"ADMIN", "USER"})
void testWithMultipleRoles() { }

// With authorities
@WithMockUser(authorities = "READ_PERMISSION")
void testWithAuthority() { }

// Complete configuration
@WithMockUser(
  username = "admin",
  password = "secret",
  roles = {"ADMIN", "USER"},
  authorities = {"READ", "WRITE"}
)
void testWithFullConfiguration() { }
```

### complete-examples.md

# Complete Examples - Before and After

## Example 1: Adding Security Tests

### Input: Service Without Security Testing

```java
@Service
public class AdminService {
    public void deleteUser(Long userId) {
        // Delete logic without security check
        repository.deleteById(userId);
    }
}
```

### Output: Service With Security Test Coverage

```java
@Service
public class AdminService {
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long userId) {
        // Delete logic with security check
        repository.deleteById(userId);
    }
}

// Test
@Test
@WithMockUser(roles = "ADMIN")
void shouldAllowAdminToDeleteUser() {
    assertThatCode(() -> adminService.deleteUser(1L))
        .doesNotThrowAnyException();
}

@Test
@WithMockUser(roles = "USER")
void shouldDenyUserFromDeletingUser() {
    assertThatThrownBy(() -> adminService.deleteUser(1L))
        .isInstanceOf(AccessDeniedException.class);
}
```

## Example 2: Replacing Manual Security Checks

### Input: Manual Security Check (Anti-Pattern)

```java
@Service
public class AdminService {
    private final UserRepository userRepository;

    public void deleteUser(Long userId, User currentUser) {
        // Manual security check in business logic
        if (currentUser.hasRole("ADMIN")) {
            repository.deleteById(userId);
        } else {
            throw new AccessDeniedException("Not authorized");
        }
    }
}
```

### Output: Declarative Security with Testing

```java
@Service
public class AdminService {
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long userId) {
        // Business logic only, security is declarative
        repository.deleteById(userId);
    }
}

// Test verifies security enforcement
@Test
@WithMockUser(roles = "ADMIN")
void shouldExecuteDelete() {
    service.deleteUser(1L);
    verify(repository).deleteById(1L);
}

@Test
@WithMockUser(roles = "USER")
void shouldNotExecuteDeleteDueToSecurity() {
    assertThatThrownBy(() -> service.deleteUser(1L))
        .isInstanceOf(AccessDeniedException.class);

    verify(repository, never()).deleteById(anyLong());
}
```

## Example 3: Controller Security Testing

### Input: Insecure Controller

```java
@RestController
@RequestMapping("/api")
public class UserController {

    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(service.findById(id));
    }

    @DeleteMapping("/users/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        service.deleteUser(id);
        return ResponseEntity.ok().build();
    }
}
```

### Output: Secure Controller with Tests

```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {

    @GetMapping("/users/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(service.findById(id));
    }

    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        service.deleteUser(id);
        return ResponseEntity.ok().build();
    }
}

// Tests
@SpringBootTest
@AutoConfigureMockMvc
class AdminControllerSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldAllowAdminToGetUser() throws Exception {
        mockMvc.perform(get("/api/admin/users/1"))
            .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "USER")
    void shouldDenyUserFromGettingUser() throws Exception {
        mockMvc.perform(get("/api/admin/users/1"))
            .andExpect(status().isForbidden());
    }

    @Test
    void shouldDenyAnonymousAccess() throws Exception {
        mockMvc.perform(get("/api/admin/users/1"))
            .andExpect(status().isUnauthorized());
    }
}
```

## Example 4: Custom Permission Evaluator

### Input: Inline Permission Check

```java
@Service
public class DocumentService {
    private final DocumentRepository repository;

    public Document getDocument(Long docId, User currentUser) {
        Document doc = repository.findById(docId)
            .orElseThrow(() -> new NotFoundException());

        // Inline permission check
        if (!doc.getOwner().equals(currentUser.getUsername()) &&
            !currentUser.hasRole("ADMIN")) {
            throw new AccessDeniedException("Access denied");
        }

        return doc;
    }
}
```

### Output: Declarative Security with Custom Evaluator

```java
@Service
public class DocumentService {

    @PreAuthorize("hasPermission(#docId, 'Document', 'READ')")
    public Document getDocument(Long docId) {
        return repository.findById(docId)
            .orElseThrow(() -> new NotFoundException());
    }
}

// Custom Permission Evaluator
@Component
public class DocumentPermissionEvaluator implements PermissionEvaluator {

    @Override
    public boolean hasPermission(Authentication authentication,
                               Serializable targetId,
                               String targetType,
                               Object permission) {
        // Permission logic extracted and reusable
        Document doc = repository.findById(targetId).orElse(null);
        if (doc == null) return false;

        return doc.getOwner().equals(authentication.getName()) ||
               hasRole(authentication, "ADMIN");
    }
}

// Test
@SpringBootTest
class DocumentServiceSecurityTest {

    @Autowired
    private DocumentService service;

    @Test
    @WithMockUser(username = "alice")
    void shouldAllowOwnerToReadDocument() {
        Document doc = service.getDocument(1L);
        assertThat(doc.getOwner()).isEqualTo("alice");
    }

    @Test
    @WithMockUser(username = "alice")
    void shouldDenyNonOwnerFromReadingDocument() {
        assertThatThrownBy(() -> service.getDocument(2L))
            .isInstanceOf(AccessDeniedException.class);
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldAllowAdminToReadAnyDocument() {
        Document doc = service.getDocument(2L);
        assertThat(doc).isNotNull();
    }
}
```

## Example 5: Complex Expression-Based Security

### Input: Multiple Manual Checks

```java
@Service
public class ProfileService {

    public UserProfile updateProfile(Long userId, ProfileUpdate update, User currentUser) {
        // Multiple manual checks
        if (!currentUser.getId().equals(userId) &&
            !currentUser.hasRole("ADMIN") &&
            !currentUser.hasRole("MODERATOR")) {
            throw new AccessDeniedException("Access denied");
        }

        if (update.isPublic() && !currentUser.isVerified()) {
            throw new AccessDeniedException("Verified users only");
        }

        return repository.update(userId, update);
    }
}
```

### Output: Declarative Expression-Based Security

```java
@Service
public class ProfileService {

    @PreAuthorize("#userId == authentication.principal.id or " +
                  "hasAnyRole('ADMIN', 'MODERATOR')")
    public UserProfile updateProfile(Long userId, ProfileUpdate update) {
        return repository.update(userId, update);
    }

    @PreAuthorize("isVerified() and hasRole('USER')")
    public void makePublic(Long userId) {
        repository.setPublic(userId, true);
    }
}

// Test
@SpringBootTest
class ProfileServiceSecurityTest {

    @Autowired
    private ProfileService service;

    @Test
    @WithMockUser(username = "alice", id = "1")
    void shouldAllowUserToUpdateOwnProfile() {
        ProfileUpdate update = new ProfileUpdate("Alice Updated");
        assertThatCode(() -> service.updateProfile(1L, update))
            .doesNotThrowAnyException();
    }

    @Test
    @WithMockUser(username = "alice", id = "1")
    void shouldDenyUserFromUpdatingOtherProfile() {
        ProfileUpdate update = new ProfileUpdate("Hacked");
        assertThatThrownBy(() -> service.updateProfile(2L, update))
            .isInstanceOf(AccessDeniedException.class);
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldAllowAdminToUpdateAnyProfile() {
        ProfileUpdate update = new ProfileUpdate("Admin Update");
        assertThatCode(() -> service.updateProfile(2L, update))
            .doesNotThrowAnyException();
    }

    @Test
    @WithMockUser(roles = "USER")
    void shouldDenyUnverifiedUserFromMakingProfilePublic() {
        assertThatThrownBy(() -> service.makePublic(1L))
            .isInstanceOf(AccessDeniedException.class);
    }
}
```

## Key Takeaways

1. **Move from imperative to declarative security** - Use annotations instead of manual checks
2. **Separate security logic from business logic** - Keep code focused on business requirements
3. **Test both positive and negative cases** - Verify both access granted and denied scenarios
4. **Use appropriate test annotations** - `@WithMockUser` for most cases, custom setup for complex scenarios
5. **Test with MockMvc for controllers** - Verifies security filters are properly configured
6. **Create reusable permission evaluators** - Extract complex permission logic into dedicated components

### setup.md

# Setup - Security Testing Dependencies

## Maven Configuration

Add the following dependencies to your `pom.xml`:

```xml
<dependencies>
  <!-- Spring Security -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>

  <!-- Spring Boot Test (includes JUnit 5) -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>

  <!-- Spring Security Test -->
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

## Enable Method Security

Add the `@EnableGlobalMethodSecurity` annotation to your configuration:

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class SecurityConfig {

    // Other security configuration
}
```

**Configuration Options:**
- `prePostEnabled = true` — Enables `@PreAuthorize` and `@PostAuthorize` annotations
- `securedEnabled = true` — Enables `@Secured` annotation
- `jsr250Enabled = true` — Enables `@RolesAllowed` annotation (JSR-250)

## Test Dependencies Included

When you add `spring-boot-starter-test`, you automatically get:
- **JUnit 5** (Jupiter) - Testing framework
- **AssertJ** - Fluent assertion library
- **Mockito** - Mocking framework
- **Hamcrest** - Matcher objects
- **JsonPath** - JSON path expressions
- **JsonAssert** - JSON assertions

## Verify Setup

Create a simple test to verify everything is configured:

```java
import org.junit.jupiter.api.Test;
import org.springframework.security.test.context.support.WithMockUser;
import static org.assertj.core.api.Assertions.*;

class SecuritySetupTest {

    @Test
    @WithMockUser
    void shouldLoadSpringSecurityContext() {
        // If this test runs, Spring Security Test is properly configured
        assertThat(true).isTrue();
    }
}
```

