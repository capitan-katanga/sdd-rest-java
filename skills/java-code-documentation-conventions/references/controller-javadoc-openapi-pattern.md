# Controller + Service Documentation Patterns

Two complete examples showing the layer split on Spring Boot 4 / Java 25 / Jakarta EE 11:

- `@RestController` classes carry **only OpenAPI annotations**. Intent for the HTTP layer lives in `@Operation(description = "…")` as **Markdown** (OpenAPI 3 CommonMark).
- `@Service`, `@Repository` (with `@Query`/`@Modifying`/`@Lock`), and custom exceptions carry **JEP 467 `///` Markdown Javadoc**.

No separate API interface. No plain `/** … */` Javadoc. No `therapi-runtime-javadoc`.

## Example 1 — `@RestController` with OpenAPI only

```java
package com.example.books.web;

import com.example.books.application.BookService;
import com.example.books.web.dto.BookResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/books")
@Tag(
    name = "Books",
    description = """
        Book catalog read operations.

        This controller owns HTTP-facing concerns only — request binding, status codes,
        payload shaping. Domain rules and persistence belong to `BookService`.
        Authentication is enforced by the security filter chain.
        """
)
public class BookController {

    private final BookService bookService;

    public BookController(BookService bookService) {
        this.bookService = bookService;
    }

    @Operation(
        summary = "Get a book by ID",
        description = """
            Retrieves a single book by its numeric identifier.

            **Idempotent**: repeated calls with the same `id` return the same payload
            until the underlying record changes.

            **Transaction boundary**: read-only — runs inside the
            `@Transactional(readOnly = true)` boundary opened by
            `BookService.findById`.

            **Side effects**: none — no events are published and no external calls are made.

            See the underlying service method `BookService.findById(Long)` for the full
            semantic contract.
            """
    )
    @ApiResponse(
        responseCode = "200",
        description = "Book found",
        content = @Content(schema = @Schema(implementation = BookResponse.class))
    )
    @ApiResponse(
        responseCode = "404",
        description = "No book exists with the given id (maps to `BookNotFoundException` thrown by the service)"
    )
    @GetMapping("/{id}")
    public BookResponse getBook(
        @Parameter(description = "Book identifier", example = "42", required = true)
        @PathVariable Long id
    ) {
        return bookService.findById(id);
    }
}
```

Reading this file, a future agent can extract:

- **Bounded responsibility** of the controller — captured in `@Tag.description` Markdown.
- **Idempotency, transaction boundary, side effects** — captured in `@Operation.description` Markdown.
- **Exception contract** — `@ApiResponse(responseCode = "404")` cross-references `BookNotFoundException` thrown by the service.
- **Pointer to the semantic contract** — `BookService.findById(Long)` is named so the agent knows where to look for the full Javadoc.

Note: no `/** */` Javadoc and no `///` Javadoc appear on the controller class. The class is OpenAPI-only.

## Example 2 — `@Service` interface with JEP 467 `///` Markdown Javadoc

```java
package com.example.books.application;

import com.example.books.application.exception.BookNotFoundException;
import com.example.books.web.dto.BookResponse;
import com.example.books.web.dto.CreateBookRequest;
import org.springframework.context.ApplicationEventPublisher;

/// Application service for the book lifecycle.
///
/// Owns creation, lookup, and deletion of books. Does **not** own:
///
/// - loan rules — see {@link LoanService}
/// - permissions — handled at the controller and security filter layer
///
/// Implementations may publish events via {@link ApplicationEventPublisher}; consult
/// the per-method Javadoc for which events fire.
///
/// @since 1.0
public interface BookService {

    /// Retrieves a book by its identifier.
    ///
    /// **Read-only**: runs inside a `@Transactional(readOnly = true)` boundary on
    /// the implementation. **Idempotent** — repeated calls with the same `id`
    /// return the same payload until the underlying record changes.
    ///
    /// **Side effects**: none.
    ///
    /// @param id the unique book identifier; must be positive
    /// @return the book payload; never `null` (a missing book throws instead)
    /// @throws BookNotFoundException if no book matches the given id
    ///         (maps to HTTP 404 on the controller via the global exception handler)
    /// @since 1.0
    BookResponse findById(Long id);

    /// Creates a new book.
    ///
    /// **Transaction boundary**: opens a new transaction
    /// (`@Transactional(propagation = REQUIRED)` on the implementation).
    ///
    /// **Side effect**: publishes a `BookCreatedEvent` after commit.
    ///
    /// **Not idempotent** — each call inserts a new row even with identical input.
    /// Callers requiring idempotency must supply an external de-duplication key.
    ///
    /// @param request the validated creation payload; must not be `null`
    /// @return the persisted book payload, including the assigned identifier
    /// @since 1.0
    BookResponse create(CreateBookRequest request);
}
```

Reading this file, a future agent can extract:

- **Bounded responsibility**: lifecycle ops only; loans elsewhere; permissions elsewhere.
- **Transaction boundaries**: explicit per method (read-only vs `REQUIRED`).
- **Idempotency**: stated per method; `create` explicitly flagged non-idempotent with the workaround pattern.
- **Side effects**: `BookCreatedEvent` published after commit on `create`; explicitly nothing on `findById`.
- **Nullability**: `findById` never returns `null`; `create` always returns the persisted shape.
- **Exception → HTTP status mapping**: `BookNotFoundException` → 404.

## Example 3 — Custom exception with `///` Markdown Javadoc

```java
package com.example.books.application.exception;

/// Thrown when a book lookup fails because no book matches the provided identifier.
///
/// Thrown by `BookService.findById(Long)` and any other service method that resolves
/// a book by id. The global `@RestControllerAdvice` maps this exception to **HTTP 404**.
///
/// @since 1.0
public class BookNotFoundException extends RuntimeException {

    /// Builds a `BookNotFoundException` for a specific identifier.
    ///
    /// @param id the identifier that was looked up and did not resolve
    public BookNotFoundException(Long id) {
        super("Book not found: id=" + id);
    }
}
```

## What these patterns are not

- **Not** a Javadoc-anywhere policy. Controllers have **zero** Javadoc.
- **Not** a separate API interface attached to a `@RestController`. The controller is annotated directly.
- **Not** plain `/** … */` Javadoc. All Javadoc uses JEP 467 `///` Markdown form.
- **Not** `therapi-runtime-javadoc` — SpringDoc reads OpenAPI annotations, not runtime Javadoc.
- **Not** a comment on every getter, setter, or trivially-named method. Document intent on public API; let well-named code speak for the rest.
