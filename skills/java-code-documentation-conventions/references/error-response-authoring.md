# Error Response Authoring

How to document HTTP error responses on `@RestController` methods, write constraint violation schemas, and capture intent on custom exception classes via JEP 467 `///` Markdown Javadoc.

This reference is about **authoring** — what to write and where. For the wiring side (the `@RestControllerAdvice` handlers, RFC 7807 schema beans, exception-to-status mapping), see `spring-boot-openapi-documentation/references/exception-handling.md`.

### Alignment with the layer-split rule

For each controller endpoint, the set of documented `@ApiResponse(responseCode = "...")` annotations must match the union of:

1. The `@throws` declared on the underlying `@Service` method's JEP 467 `///` Javadoc, **and**
2. Whatever the global `@RestControllerAdvice` handler maps those exceptions to.

Drift between service `@throws` and controller `@ApiResponse` is a documentation bug.

## Document HTTP status codes on each operation

### API response with all error codes enumerated

```java
@Operation(
    summary = "Get book by ID",
    responses = {
        @ApiResponse(
            responseCode = "200",
            description = "Book found",
            content = @Content(schema = @Schema(implementation = Book.class))
        ),
        @ApiResponse(
            responseCode = "404",
            description = "Book not found",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        ),
        @ApiResponse(
            responseCode = "401",
            description = "Unauthorized",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        ),
        @ApiResponse(
            responseCode = "500",
            description = "Internal server error",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        )
    }
)
@GetMapping("/{id}")
public Book getBook(@PathVariable Long id) {
    return repository.findById(id).orElseThrow(() -> new BookNotFoundException(id));
}
```

### Custom error response examples

```java
@Operation(
    summary = "Create book",
    responses = {
        @ApiResponse(
            responseCode = "201",
            description = "Book created"
        ),
        @ApiResponse(
            responseCode = "400",
            description = "Validation failed",
            content = @Content(
                schema = @Schema(implementation = ErrorResponse.class),
                examples = @ExampleObject(
                    value = """
                    {
                        "code": "VALIDATION_ERROR",
                        "message": "Validation failed",
                        "details": [
                            {"field": "title", "message": "Title is required"},
                            {"field": "isbn", "message": "Invalid ISBN format"}
                        ],
                        "timestamp": "2024-01-15T10:30:00Z",
                        "path": "/api/books"
                    }
                    """
                )
            )
        )
    }
)
@PostMapping
public Book createBook(@Valid @RequestBody Book book) {
    return repository.save(book);
}
```

## Document constraint violations

When the global handler maps Bean Validation failures to a structured violations payload, the violation record itself is a DTO — apply `@Schema` to it, no Javadoc:

```java
@Schema(description = "Constraint violation detail")
public record ConstraintViolation(
    @Schema(description = "Invalid field", example = "title")
    String field,

    @Schema(description = "Constraint that failed", example = "NotBlank")
    String constraint,

    @Schema(description = "Error message", example = "must not be blank")
    String message,

    @Schema(description = "Invalid value", example = "null")
    String rejectedValue
) {}

@Schema(description = "Validation error response")
public record ValidationErrorResponse(
    @Schema(description = "Error code", example = "VALIDATION_FAILED")
    String code,

    @Schema(description = "Validation errors")
    List<ConstraintViolation> violations,

    @Schema(description = "Total number of violations", example = "2")
    int violationCount,

    @Schema(description = "Timestamp", example = "2024-01-15T10:30:00Z")
    LocalDateTime timestamp
) {}
```

## Custom exception intent — JEP 467 `///` Markdown Javadoc

Custom exception classes carry their intent (when fired, which layer throws them, which HTTP status the global handler maps to) in JEP 467 `///` Javadoc — **not** in any OpenAPI annotation. The exception class is not part of the controller layer.

```java
/// Thrown when an operation cannot proceed because a book has fewer copies in stock
/// than the caller requested.
///
/// **Layer**: thrown by `BookService.reserve(...)` during stock checks.
///
/// **HTTP mapping**: the global `@RestControllerAdvice` maps this exception to
/// `409 Conflict` with an `ErrorResponse` body carrying the
/// `INSUFFICIENT_STOCK` code and the available/requested counts.
///
/// **Idempotency**: the throw is deterministic for a given stock state — repeated
/// reservation attempts with the same arguments produce the same exception until
/// stock changes.
///
/// @since 1.0
public class InsufficientStockException extends RuntimeException {

    private final Long bookId;
    private final int requested;
    private final int available;

    /// Builds the exception with the book and stock counts to be included in the
    /// formatted message and the `ErrorResponse` body.
    ///
    /// @param bookId the identifier of the book that triggered the shortage
    /// @param requested the number of copies the caller asked to reserve; must be > 0
    /// @param available the number of copies actually in stock at the time of the check; >= 0
    public InsufficientStockException(Long bookId, int requested, int available) {
        super(String.format("Insufficient stock for book %d: requested=%d, available=%d",
            bookId, requested, available));
        this.bookId = bookId;
        this.requested = requested;
        this.available = available;
    }

    // Getters...
}
```

On the corresponding `@RestController` method, the `409 Conflict` response should appear in `@ApiResponse`, with the description tying back to the same exception name so a future reader can trace it:

```java
@ApiResponse(
    responseCode = "409",
    description = "Insufficient stock — see InsufficientStockException",
    content = @Content(schema = @Schema(implementation = ErrorResponse.class))
)
```

## Best practices for error response authoring

1. **Enumerate every status the handler can produce**: every `@ExceptionHandler` in the `@RestControllerAdvice` should have a matching `@ApiResponse` on at least one controller method.
2. **Use one consistent error envelope**: pick `ErrorResponse` or RFC 7807 `ProblemDetail` and apply the same `@Schema(implementation = ...)` across all 4xx/5xx responses.
3. **Provide actionable messages**: error descriptions and examples should let the API client understand and fix the problem.
4. **Don't expose sensitive data**: sanitize exception messages and example payloads — no stack traces, credentials, or internal identifiers.
5. **Use appropriate HTTP status codes**: 4xx for caller faults, 5xx for server faults, 409 for state conflicts, 422 for semantic validation failures.
6. **Include a request ID for traceability** when the envelope supports it:

```java
@Schema(description = "Error response with request tracking")
public record ErrorResponse(
    String code,
    String message,
    Object details,
    LocalDateTime timestamp,
    String path,
    @Schema(description = "Request ID for support", example = "abc-123-xyz")
    String requestId
) {}
```

7. **Keep service `@throws` and controller `@ApiResponse` aligned**: if you add a new exception path on the service, walk the corresponding controller method and add the matching `@ApiResponse`.
