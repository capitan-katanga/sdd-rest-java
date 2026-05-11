---
paths:
  - "**/*.java"
---
# Rule: Java Project Structure

## Context
Enforce a consistent, domain-driven project structure for Java applications, particularly Spring Boot projects, to maintain separation of concerns and scalability.

## Guidelines

### Package Organization
Organize packages by domain module, then by layer within each module:

```
com.company.project/
├── config/                  # Application-level configuration
│   ├── SecurityConfig.java
│   └── WebConfig.java
├── common/                  # Shared utilities and base classes
│   ├── exception/
│   │   ├── BusinessException.java
│   │   └── GlobalExceptionHandler.java
│   └── util/
├── order/                   # Domain module: Order
│   ├── web/                 # Controllers / REST endpoints
│   │   ├── OrderController.java
│   │   └── dto/
│   │       ├── OrderRequest.java
│   │       └── OrderResponse.java
│   ├── service/             # Business logic
│   │   ├── OrderService.java
│   │   └── impl/
│   │       └── OrderServiceImpl.java
│   ├── repository/          # Data access
│   │   └── OrderRepository.java
│   └── domain/              # Entity / domain model
│       ├── Order.java
│       └── OrderStatus.java
└── user/                    # Domain module: User
    ├── web/
    ├── service/
    ├── repository/
    └── domain/
```

### Layer Responsibilities (MVC + Service + Repository)
This layout is a **layered MVC** structure: `web/` is the **C**ontroller layer, `domain/` carries the **M**odel (DTOs are the response representation), and the **V**iew is JSON (no server-side rendering). `service/` is the business-logic layer between web and persistence; `repository/` is the data-access boundary.

- **web/**: REST controllers (`@RestController`, `@RequestMapping`), request/response DTOs, input validation. **Thin** — no business logic.
- **service/**: Business logic, transaction management, orchestration. Speaks to repositories and other services; never speaks back to controllers.
- **repository/**: Data access interfaces (Spring Data JPA repositories). Returns entities; never returns DTOs.
- **domain/**: JPA entities, value objects, enums. No Spring dependencies.
- **config/**: Spring `@Configuration` classes.
- **exception/**: Custom exceptions and global `@RestControllerAdvice` handlers.

### Dependency direction
Dependencies flow one way: **web → service → repository → domain**. Never the reverse.
- A repository must not import a controller or a DTO.
- A service must not import a controller (use `web/dto/` records as data, not as a coupling channel).
- The `domain/` package depends on nothing inside the module.

Violations of this direction are the most common architectural smell in Spring Boot codebases — they make tests harder, transactions leakier, and refactors riskier.

### File Organization Rules
- One public class per file
- DTOs belong in `web/dto/` or a shared `dto/` package, never in `domain/`
- Use Records for DTOs (Java 17+)
- Keep controllers thin — delegate logic to services
- Repository interfaces extend `JpaRepository` or `CrudRepository`
- Configuration classes should be focused and single-purpose

### Test Structure
Mirror the source package structure under `src/test/java/`:
```
src/test/java/com/company/project/order/
├── web/
│   └── OrderControllerTest.java
├── service/
│   └── OrderServiceTest.java
└── repository/
    └── OrderRepositoryTest.java
```

## Examples

### ✅ Good
```java
// src/main/java/com/acme/order/web/dto/OrderResponse.java
package com.acme.order.web.dto;

public record OrderResponse(Long id, String status, BigDecimal total) {}
```

### ❌ Bad
```java
// Placing DTOs in domain package
package com.acme.order.domain;

public class OrderDTO { ... }  // Wrong package and not a Record
```
