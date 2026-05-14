# Exception Handler Wiring

This file documents `@RestControllerAdvice` handler wiring for SpringDoc-aware Spring Boot applications. Handlers hide themselves from the generated OpenAPI document via `@Operation(hidden = true)` so the response codes they emit stay declared on the controller methods, not on the handlers.

> For authoring the matching `@ApiResponse` annotations on controllers, the `@Schema` records (`ErrorResponse`, constraint violation envelopes, custom `ProblemDetail`) and JEP 467 `///` Javadoc on custom exception classes, see `java-code-documentation-conventions/references/error-response-authoring.md`.

## Global Exception Handler

### Comprehensive handler with custom error envelope

```java
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import io.swagger.v3.oas.annotations.Operation;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BookNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    @Operation(hidden = true)
    public ErrorResponse handleBookNotFound(BookNotFoundException ex) {
        return new ErrorResponse("BOOK_NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @Operation(hidden = true)
    public ErrorResponse handleValidation(ValidationException ex) {
        return new ErrorResponse("VALIDATION_ERROR", ex.getMessage());
    }

    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    @Operation(hidden = true)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex) {
        return new ErrorResponse("ACCESS_DENIED", "Insufficient permissions");
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @Operation(hidden = true)
    public ErrorResponse handleGeneric(Exception ex) {
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
}
```

## RFC 7807 Problem Details Handler

Spring 6+ ships `org.springframework.http.ProblemDetail` as the standard RFC 7807 envelope. A handler can return one directly without a custom record:

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import java.net.URI;

@RestControllerAdvice
public class ProblemDetailExceptionHandler {

    @ExceptionHandler(BookNotFoundException.class)
    @Operation(hidden = true)
    public ProblemDetail handleNotFound(BookNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setType(URI.create("https://example.com/probs/book-not-found"));
        pd.setTitle("Book Not Found");
        pd.setDetail(ex.getMessage());
        pd.setInstance(URI.create("/api/books/" + ex.getId()));
        return pd;
    }
}
```

## Business Exception Handler

```java
@ExceptionHandler(InsufficientStockException.class)
@ResponseStatus(HttpStatus.CONFLICT)
@Operation(hidden = true)
public ErrorResponse handleInsufficientStock(InsufficientStockException ex) {
    return new ErrorResponse(
        "INSUFFICIENT_STOCK",
        String.format("Only %d copies available, %d requested", ex.getAvailable(), ex.getRequested()),
        Map.of(
            "bookId", ex.getBookId(),
            "requested", ex.getRequested(),
            "available", ex.getAvailable()
        )
    );
}
```

## Wiring reminders

- Annotate every handler method with `@Operation(hidden = true)` so SpringDoc does not surface the handler in the generated spec.
- Pair `@ExceptionHandler` with `@ResponseStatus` so the HTTP status is set even when the handler returns a plain object (no `ResponseEntity`).
- The exception types the handler catches must match the `@throws` declared on the underlying `@Service` method's JEP 467 `///` Javadoc — drift produces missing status codes in the rendered Swagger UI.
- For status codes to render on each controller method, declare the corresponding `@ApiResponse` annotations on the controller — the handler alone does not feed them into the operation node.
