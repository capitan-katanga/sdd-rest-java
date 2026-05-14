---
name: java-code-documentation-conventions
description: Philosophy, style guide, and authoring patterns for in-code documentation on Spring Boot 4.x / Java 25 / Jakarta EE 11 public APIs. A class either carries OpenAPI annotations OR Javadoc — never both. @RestController classes use ONLY OpenAPI annotations (@Tag, @Operation, @ApiResponse, @Parameter, @Schema), with intent in @Operation(description) as Markdown. @Service, @Repository (with @Query/@Modifying/@Lock), and custom exceptions use JEP 467 /// Markdown Javadoc. Owns full authoring guidance: controller and DTO patterns, attribute-by-attribute annotation reference, error response authoring (status codes, constraint violations, custom exception intent). Use when documenting public Java APIs, writing or reviewing OpenAPI annotations or Javadoc, choosing between OpenAPI vs Javadoc on a class, looking up an annotation attribute, or preparing code so future agents can extract intent (preconditions, idempotency, exception contracts, transaction boundaries, side effects).
version: 0.3.0
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Java Code Documentation Conventions

## Overview

This skill is the *philosophy and style guide* for documenting Java public APIs so that both human readers and future Claude agents can extract intent when modifying code.

The single load-bearing rule: **a class either carries OpenAPI annotations OR Javadoc, never both on the same class.** Controllers get OpenAPI; everything else public gets JEP 467 Markdown Javadoc.

This skill does **not** cover SpringDoc dependency setup, Swagger UI configuration, or annotation mechanics — those live in `spring-boot-openapi-documentation`.

## When to Use

- Writing or reviewing OpenAPI annotations on `@RestController` classes
- Writing patterns for `@Tag`, `@Operation`, `@ApiResponse`, `@Parameter`, and `@Schema` annotations on controllers and DTO classes (see [openapi-authoring-patterns.md](references/openapi-authoring-patterns.md))
- Looking up the attributes of an OpenAPI annotation (see [openapi-annotation-reference.md](references/openapi-annotation-reference.md))
- Documenting error response codes, constraint violations, or custom exception intent (see [error-response-authoring.md](references/error-response-authoring.md))
- Writing or reviewing JEP 467 `///` Markdown Javadoc on `@Service`, `@Repository`, or custom exceptions
- Deciding which Javadoc tags (`@param`, `@return`, `@throws`, `@since`, `@see`) belong on a given public method
- Capturing intent (preconditions, idempotency, transaction boundary, side effects, exception contracts, nullability) where future agents will find it
- Documenting `@Repository` methods that carry `@Query`, `@Modifying`, or `@Lock`
- Documenting custom exception types so callers know when they fire and which HTTP status they map to

## Do NOT Use This Skill When

- Setting up springdoc-openapi dependency, Swagger UI config, security scheme **wiring**, or pagination **plumbing** → use `spring-boot-openapi-documentation` (mechanics only)
- Designing endpoints, DTOs, error envelopes, status codes, HATEOAS, or pagination contracts → use `spring-boot-rest-api-standards`
- Implementing `@Query` JPA mechanics (JPQL/native syntax, projections, locking) → use `spring-data-jpa`
- Documenting private, package-private, or test-internal code — this skill targets public API surface only
- Writing external documentation artifacts (ADRs, C4 diagrams, deployment runbooks, multi-audience manuals)

## The Layer Split (load-bearing)

| Layer | Documentation mechanism | Carries |
|---|---|---|
| `@RestController` classes | **OpenAPI annotations only** | HTTP contract (status codes, schemas, parameter binding, security) + intent for the HTTP layer in `@Operation(description = "…")` as Markdown |
| `@Service` (interface or class) | **JEP 467 `///` Markdown Javadoc** | Semantic intent: preconditions, postconditions, transaction boundary, idempotency, events published, external calls, nullability |
| `@Repository` methods with `@Query`/`@Modifying`/`@Lock` | **JEP 467 `///` Markdown Javadoc** on those methods | Business meaning of the query, lock semantics, persistence-context effects |
| Custom exceptions | **JEP 467 `///` Markdown Javadoc** | When the exception fires, which layer throws it, which HTTP status the global handler maps it to |

**Hard rules:**

- **Never** put any Javadoc (neither `/** */` nor `///`) on a `@RestController` class. Not at class level, not at method level.
- **Never** put OpenAPI annotations on a non-controller class.
- **Annotate `@RestController` directly** — never split into a separate `XxxApi` interface that the controller implements.
- **Never** introduce the `therapi-runtime-javadoc` bridge dependency.

## What Lives Where for Each Target

### `@RestController` — OpenAPI annotations only

- `@Tag` at the class level — bounded responsibility of this controller, base path / domain it owns.
- `@Operation(summary = "...", description = "...")` on each method. The `description` field is **Markdown** (OpenAPI 3 / CommonMark) — use it to encode the intent that would otherwise live in Javadoc:
  - preconditions ("Caller must hold a valid bearer token")
  - idempotency ("Idempotent — repeated calls with the same `{id}` return the same payload")
  - side effects on the service layer ("Publishes `BookCreatedEvent` after commit on success")
  - transaction boundary ("Read-only; runs in a single read-only transaction opened by `BookService`")
  - links to the corresponding service method via Markdown
- `@ApiResponse(responseCode = "...", description = "...", content = ...)` — one per documented status code. Every error response that the global exception handler can produce should appear here.
- `@Parameter` on every path/query parameter that needs an example or description beyond what `@Schema` provides.
- `@Schema` on DTO records (lives in DTO files, not the controller).

For full pattern examples (controller, DTOs, end-to-end), see [openapi-authoring-patterns.md](references/openapi-authoring-patterns.md). For an attribute-by-attribute reference of each annotation, see [openapi-annotation-reference.md](references/openapi-annotation-reference.md).

### `@Service` — JEP 467 `///` Markdown Javadoc

- **Class-level**: the single responsibility this service owns and what it explicitly does *not* own (delegates elsewhere).
- **Public methods**: contract — preconditions, postconditions, transaction boundary (propagation, read-only), idempotency, events published, external calls made, nullability of return. Tags: `@param`, `@return`, `@throws`, `@since`, `@see`.

### `@Repository` methods with `@Query`/`@Modifying`/`@Lock` — JEP 467 `///` Markdown Javadoc

- Business-level description of what the query does (not what the JPQL says — the code shows that).
- Why a custom query is used instead of derived method names.
- Lock semantics (`@Lock`), whether `@Modifying(clearAutomatically = true)` is in play.
- Plain derived-method-name repository methods need no Javadoc — the method name documents intent.

### Custom exceptions — JEP 467 `///` Markdown Javadoc

- **Class-level**: when this exception fires, which layer throws it, which HTTP status the global handler maps it to.
- **Constructors**: meaning of each parameter and what the formatted message conveys.

For a full example (exception class with `///` Javadoc + matching `@ApiResponse` on the controller), see [error-response-authoring.md](references/error-response-authoring.md).

## Javadoc Style (JEP 467 Markdown)

See [javadoc-tag-reference.md](references/javadoc-tag-reference.md) for the full table.

Core rules:

- Use JEP 467 `///` Markdown Javadoc syntax. Each line starts with `///`. The body is CommonMark.
- **Forbidden**: plain `/** ... */` Javadoc on the documented surface; `therapi-runtime-javadoc` dependency.
- First line is the summary, ends with a period, ≤ 120 chars. Blank `///` line, then description block.
- Use Markdown for emphasis (`**bold**`, `*italic*`), lists, fenced code blocks, and `[link text](URL)` when useful.
- Tags (`@param`, `@return`, `@throws`, `@since`, `@see`) are still standard Javadoc tags inside the `///` block.

## Intent Signals Future Agents Need to Extract

When documenting a public method, make these signals discoverable to a future agent reading the file:

- **Preconditions**: what must be true before calling (auth state, prior calls, entity existence)
- **Exception contract**: which exception fires when, and (for controllers) which `@ApiResponse` status code it maps to
- **Idempotency**: state explicitly whether a second identical call has the same effect
- **Transaction boundary**: when does a `@Transactional` boundary start, what propagation, is the method read-only
- **Observable side effects**: events published, external HTTP calls made, files written, caches evicted
- **Nullability of return**: when `Optional<T>`, document the empty case; when nullable `T`, document why and when

If a signal is non-obvious from the method body, it belongs in the documentation — `@Operation.description` Markdown for controllers, `///` Markdown Javadoc for the rest.

## Alignment Check

For each controller endpoint, walk the underlying service method's `@throws` tags and confirm every exception either:

1. Has a matching `@ApiResponse(responseCode = "...")` on the controller method, or
2. Is intentionally mapped to 500 / a generic fallback by the global exception handler (note this in the controller's `@Operation.description`).

Drift between service `@throws` and controller `@ApiResponse` is a documentation bug.

## Examples

### Minimal `@RestController` — OpenAPI only, intent in `@Operation.description` Markdown

```java
@RestController
@RequestMapping("/api/books")
@Tag(name = "Books", description = "Book catalog operations")
public class BookController {

    @Operation(
        summary = "Get book by ID",
        description = """
            Retrieves a single book by its numeric identifier.

            **Idempotent**: repeated calls with the same `id` return the same payload
            until the underlying record changes. Read-only — runs inside the
            read-only transaction opened by `BookService.findById`.

            **No side effects**: no events are published and no external calls are made.
            """
    )
    @ApiResponse(responseCode = "200", description = "Book found",
        content = @Content(schema = @Schema(implementation = BookResponse.class)))
    @ApiResponse(responseCode = "404", description = "No book exists with the given id")
    @GetMapping("/{id}")
    public BookResponse getBook(
        @Parameter(description = "Book identifier", example = "42") @PathVariable Long id) {
        return bookService.findById(id);
    }
}
```

### Minimal `@Service` — JEP 467 `///` Markdown Javadoc

```java
/// Application service for the book lifecycle.
///
/// Owns creation, lookup, and deletion of books. Does **not** own loan rules —
/// see {@link LoanService} — and does not enforce permissions, which are handled
/// at the controller and security layer.
///
/// @since 1.0
public interface BookService {

    /// Retrieves a book by its identifier.
    ///
    /// Read-only. Runs inside a `@Transactional(readOnly = true)` boundary
    /// on the implementation. **Idempotent**.
    ///
    /// @param id the unique book identifier; must be positive
    /// @return the book payload; never `null`
    /// @throws BookNotFoundException if no book matches the given id
    /// @since 1.0
    BookResponse findById(Long id);
}
```

Full versions live in [controller-javadoc-openapi-pattern.md](references/controller-javadoc-openapi-pattern.md).

## References

- **[javadoc-tag-reference.md](references/javadoc-tag-reference.md)** — Tag-by-tag table for JEP 467 `///` Markdown Javadoc: when each tag is required, what it must contain, fragment examples.
- **[controller-javadoc-openapi-pattern.md](references/controller-javadoc-openapi-pattern.md)** — Minimal layer-split example: a Spring Boot 4 `@RestController` with OpenAPI annotations only (intent in `@Operation.description` Markdown), and a `@Service` interface with JEP 467 `///` Markdown Javadoc covering transaction boundary, idempotency, and exception contract.
- **[openapi-authoring-patterns.md](references/openapi-authoring-patterns.md)** — Detailed annotation patterns: controller patterns (basic, request bodies, multiple response types, parameters, headers), DTO `@Schema` patterns (validation, nested, enums, hidden, read-only, arrays, polymorphic, required/optional), and a complete end-to-end controller + entity example.
- **[openapi-annotation-reference.md](references/openapi-annotation-reference.md)** — Attribute-by-attribute reference for `@Tag`, `@Operation`, `@ApiResponse`, `@Parameter`, `@RequestBody` (OpenAPI), `@Schema`, `@SecurityRequirement`, `@Hidden`, `@ParameterObject`, `@ArraySchema`, `@Content`, `@ExampleObject`, plus auto-documented Bean Validation annotations.
- **[error-response-authoring.md](references/error-response-authoring.md)** — Documenting HTTP status codes via `@ApiResponse`, custom error envelopes, constraint violations, and JEP 467 `///` Markdown Javadoc on custom exception classes (when fired, layer, HTTP status mapping).

## Constraints and Warnings

- **Never** put any Javadoc on a `@RestController` class. Intent for the HTTP layer goes in `@Operation(description = "...")` as Markdown.
- **Never** put OpenAPI annotations on a non-controller class.
- **Never** split `@RestController` into a separate API interface + implementation.
- **Never** use plain `/** ... */` Javadoc on documented public API — use JEP 467 `///` Markdown form.
- **Never** introduce `therapi-runtime-javadoc` as a dependency.
- Do not narrate what the code does (well-named identifiers do that). Document *intent*: why, when, what callers must guarantee, what the method guarantees back.
- Keep service-level `@throws` aligned with controller-level `@ApiResponse` status codes. Drift between them is a documentation bug.
- Do not document private or package-private members through this skill — its scope is the public API surface.

## Related Skills

- `spring-boot-openapi-documentation` — SpringDoc **wiring/mechanics only**: dependency setup, `application.yml`, Swagger UI configuration, `SecurityScheme` bean configuration, pagination plumbing, build integration, API groups, troubleshooting, `@RestControllerAdvice` handler wiring. The complement to this skill — that one wires the spec generator, this one writes the annotations.
- `spring-boot-rest-api-standards` — REST API design (endpoints, DTOs, error envelopes, status codes). Decides the contract; this skill documents it.
- `spring-data-jpa` — `@Query`, `@Modifying`, locking, projections mechanics. This skill documents intent; that skill implements it.

## External Resources

- [JEP 467: Markdown Documentation Comments](https://openjdk.org/jeps/467)
- [SpringDoc Official Documentation](https://springdoc.org/)
- [OpenAPI 3.0 Specification](https://swagger.io/specification/)
- [CommonMark Specification](https://commonmark.org/)
