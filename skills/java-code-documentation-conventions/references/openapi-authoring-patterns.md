# OpenAPI Annotation Authoring Patterns

Detailed patterns for writing OpenAPI annotations on `@RestController` classes and DTOs.

This reference assumes the layer-split rule established by the parent skill: `@RestController` classes carry **only** OpenAPI annotations (never Javadoc). For the minimal layer-split example side-by-side with a `@Service` using JEP 467 `///` Javadoc, see [controller-javadoc-openapi-pattern.md](controller-javadoc-openapi-pattern.md). For the attribute-level reference of every annotation, see [openapi-annotation-reference.md](openapi-annotation-reference.md).

## Contents

1. [Controller annotation patterns](#controller-annotation-patterns)
2. [DTO `@Schema` patterns](#dto-schema-patterns)
3. [Complete end-to-end example](#complete-end-to-end-example)

---

## Controller annotation patterns

### Basic controller documentation

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/books")
@Tag(name = "Book", description = "Book management APIs")
public class BookController {

    @Operation(
        summary = "Retrieve a book by ID",
        description = "Get a Book object by specifying its ID. The response includes id, title, author and description."
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "200",
            description = "Successfully retrieved book",
            content = @Content(schema = @Schema(implementation = Book.class))
        ),
        @ApiResponse(
            responseCode = "404",
            description = "Book not found"
        )
    })
    @GetMapping("/{id}")
    public Book findById(
        @Parameter(description = "ID of book to retrieve", required = true)
        @PathVariable Long id
    ) {
        return repository.findById(id)
            .orElseThrow(() -> new BookNotFoundException());
    }
}
```

### Document request bodies

```java
import io.swagger.v3.oas.annotations.parameters.RequestBody;
import io.swagger.v3.oas.annotations.media.ExampleObject;

@Operation(summary = "Create a new book")
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public Book createBook(
    @RequestBody(
        description = "Book to create",
        required = true,
        content = @Content(
            schema = @Schema(implementation = Book.class),
            examples = @ExampleObject(
                value = """
                {
                    "title": "Clean Code",
                    "author": "Robert C. Martin",
                    "isbn": "978-0132350884",
                    "description": "A handbook of agile software craftsmanship"
                }
                """
            )
        )
    )
    Book book
) {
    return repository.save(book);
}
```

### Multiple response types

```java
@Operation(summary = "Search books")
@ApiResponses(value = {
    @ApiResponse(
        responseCode = "200",
        description = "Search completed",
        content = @Content(
            array = @ArraySchema(schema = @Schema(implementation = Book.class))
        )
    ),
    @ApiResponse(
        responseCode = "400",
        description = "Invalid search parameters",
        content = @Content(schema = @Schema(implementation = ErrorResponse.class))
    )
})
@GetMapping("/search")
public List<Book> searchBooks(
    @Parameter(description = "Search query")
    @RequestParam String query
) {
    return service.search(query);
}
```

### Document query parameters

```java
@GetMapping("/filtered")
public List<Book> filterBooks(
    @Parameter(description = "Title filter", example = "Clean Code")
    @RequestParam(required = false) String title,

    @Parameter(description = "Author filter", example = "Robert C. Martin")
    @RequestParam(required = false) String author,

    @Parameter(description = "Minimum publication year", example = "2000")
    @RequestParam(required = false) Integer fromYear
) {
    return service.filter(title, author, fromYear);
}
```

### Document path parameters with mixed binding

```java
@Operation(summary = "Get book attributes")
@GetMapping("/{id}/attributes/{attributeType}")
public BookAttribute getAttribute(
    @Parameter(description = "Book ID", required = true)
    @PathVariable Long id,

    @Parameter(description = "Attribute type (metadata, reviews, ratings)", required = true)
    @PathVariable String attributeType
) {
    return service.getAttribute(id, attributeType);
}
```

### Document request headers

```java
@Operation(summary = "Get authenticated user profile")
@GetMapping("/profile")
public UserProfile getProfile(
    @Parameter(description = "Authorization token", required = true, example = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...")
    @RequestHeader("Authorization") String authorization
) {
    return service.getProfile(authorization);
}
```

---

## DTO `@Schema` patterns

### Entity with validation

```java
import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.*;

@Entity
@Schema(description = "Book entity representing a published book")
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Schema(description = "Unique identifier", example = "1", accessMode = Schema.AccessMode.READ_ONLY)
    private Long id;

    @NotBlank(message = "Title is required")
    @Size(min = 1, max = 200)
    @Schema(description = "Book title", example = "Clean Code", required = true, maxLength = 200)
    private String title;

    @NotBlank(message = "Author is required")
    @Schema(description = "Book author", example = "Robert C. Martin", required = true)
    private String author;

    @Pattern(regexp = "^(?:ISBN(?:-1[03])?:? )?(?=[0-9X]{10}$|(?=(?:[0-9]+[- ]){3})[- 0-9X]{13}$|97[89][0-9]{10}$|(?=(?:[0-9]+[- ]){4})[- 0-9]{17}$)(?:97[89][- ]?)?[0-9]{1,5}[- ]?[0-9]+[- ]?[0-9]+[- ]?[0-9X]$")
    @Schema(description = "ISBN number", example = "978-0132350884")
    private String isbn;

    @Min(value = 0, message = "Price must be positive")
    @Schema(description = "Book price in USD", example = "29.99", minimum = "0")
    private BigDecimal price;

    @Past(message = "Publication date must be in the past")
    @Schema(description = "Publication date", example = "2008-08-01")
    private LocalDate publicationDate;

    @Email(message = "Publisher email must be valid")
    @Schema(description = "Publisher contact email", example = "contact@publisher.com")
    private String publisherEmail;

    // Constructors, getters, setters...
}
```

### Nested objects

```java
@Schema(description = "Book with publisher details")
public class BookDetail {

    @Schema(description = "Book information")
    private Book book;

    @Schema(description = "Publisher information")
    private Publisher publisher;

    @Schema(description = "Publication details")
    private PublicationInfo publicationInfo;
}

@Schema(description = "Publisher entity")
public class Publisher {
    @Schema(example = "Prentice Hall")
    private String name;

    @Schema(example = "contact@pearson.com")
    private String email;
}
```

### Enum documentation

```java
public enum BookStatus {
    @Schema(description = "Book is available for purchase")
    AVAILABLE,

    @Schema(description = "Book is out of stock")
    OUT_OF_STOCK,

    @Schema(description = "Book is discontinued")
    DISCONTINUED
}

@Schema(description = "Book entity")
public class Book {
    @Schema(description = "Current book status", example = "AVAILABLE")
    private BookStatus status;
}
```

### Hidden and write-only fields

```java
@Schema(hidden = true)
private String internalField;

@JsonIgnore
@Schema(accessMode = Schema.AccessMode.READ_ONLY)
private LocalDateTime createdAt;

@Schema(description = "Password hash (write-only)", accessMode = Schema.AccessMode.WRITE_ONLY)
private String password;
```

### Read-only timestamps

```java
@Schema(description = "Creation timestamp", accessMode = Schema.AccessMode.READ_ONLY, example = "2024-01-15T10:30:00Z")
private LocalDateTime createdAt;

@Schema(description = "Last update timestamp", accessMode = Schema.AccessMode.READ_ONLY, example = "2024-01-15T10:30:00Z")
private LocalDateTime updatedAt;
```

### Arrays and collections

```java
@Schema(description = "List of book tags")
private List<String> tags;

@Schema(description = "Map of book metadata")
private Map<String, String> metadata;

@Schema(description = "Set of book categories")
private Set<Category> categories;
```

### Polymorphic types

```java
@Schema(description = "Payment method (one of: creditCard, paypal, bankTransfer)")
@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,
    include = JsonTypeInfo.As.PROPERTY,
    property = "type"
)
@JsonSubTypes({
    @JsonSubTypes.Type(value = CreditCardPayment.class, name = "creditCard"),
    @JsonSubTypes.Type(value = PayPalPayment.class, name = "paypal"),
    @JsonSubTypes.Type(value = BankTransferPayment.class, name = "bankTransfer")
})
public abstract class PaymentMethod {
    @Schema(example = "100.00")
    protected BigDecimal amount;
}
```

### Required vs optional fields

```java
@Schema(description = "User profile")
public class UserProfile {

    @NotNull
    @Schema(description = "User first name", example = "John", required = true)
    private String firstName;

    @NotNull
    @Schema(description = "User last name", example = "Doe", required = true)
    private String lastName;

    @Schema(description = "User middle name (optional)", example = "William")
    private String middleName;

    @Schema(description = "User nickname (optional)", example = "Johnny")
    private String nickname;
}
```

---

## Complete end-to-end example

### Full-featured book controller

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.parameters.RequestBody;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springdoc.core.annotations.ParameterObject;
import org.springframework.web.bind.annotation.*;
import jakarta.validation.Valid;

@RestController
@RequestMapping("/api/books")
@Tag(name = "Book", description = "Book management APIs")
@SecurityRequirement(name = "bearer-jwt")
public class BookController {

    private final BookService bookService;

    public BookController(BookService bookService) {
        this.bookService = bookService;
    }

    @Operation(summary = "Get all books")
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "200",
            description = "Found all books",
            content = @Content(
                mediaType = "application/json",
                array = @ArraySchema(schema = @Schema(implementation = Book.class))
            )
        )
    })
    @GetMapping
    public List<Book> getAllBooks() {
        return bookService.getAllBooks();
    }

    @Operation(summary = "Get paginated books")
    @GetMapping("/paginated")
    public Page<Book> getBooksPaginated(@ParameterObject Pageable pageable) {
        return bookService.getBooksPaginated(pageable);
    }

    @Operation(summary = "Get book by ID")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Book found"),
        @ApiResponse(responseCode = "404", description = "Book not found")
    })
    @GetMapping("/{id}")
    public Book getBookById(
        @Parameter(description = "Book ID", required = true, example = "1")
        @PathVariable Long id
    ) {
        return bookService.getBookById(id);
    }

    @Operation(summary = "Create new book")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "201", description = "Book created successfully"),
        @ApiResponse(responseCode = "400", description = "Invalid input")
    })
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Book createBook(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            description = "Book to create",
            required = true,
            content = @Content(
                schema = @Schema(implementation = Book.class),
                examples = @io.swagger.v3.oas.annotations.media.ExampleObject(
                    value = """
                        {
                            "title": "Clean Code",
                            "author": "Robert C. Martin",
                            "isbn": "978-0132350884",
                            "price": 29.99,
                            "publicationDate": "2008-08-01"
                        }
                        """
                )
            )
        )
        @Valid @RequestBody Book book
    ) {
        return bookService.createBook(book);
    }

    @Operation(summary = "Update book")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Book updated"),
        @ApiResponse(responseCode = "404", description = "Book not found"),
        @ApiResponse(responseCode = "400", description = "Invalid input")
    })
    @PutMapping("/{id}")
    public Book updateBook(
        @Parameter(description = "Book ID", required = true)
        @PathVariable Long id,
        @Valid @RequestBody Book book
    ) {
        return bookService.updateBook(id, book);
    }

    @Operation(summary = "Delete book")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "204", description = "Book deleted"),
        @ApiResponse(responseCode = "404", description = "Book not found")
    })
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteBook(@PathVariable Long id) {
        bookService.deleteBook(id);
    }

    @Operation(summary = "Search books by title")
    @GetMapping("/search")
    public Page<Book> searchBooks(
        @Parameter(description = "Search query", example = "Clean")
        @RequestParam String query,
        @ParameterObject Pageable pageable
    ) {
        return bookService.searchBooks(query, pageable);
    }
}
```

### Complete book entity

```java
import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.*;
import java.math.BigDecimal;
import java.time.LocalDate;

@Entity
@Table(name = "books")
@Schema(description = "Book entity representing a published book")
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Schema(description = "Unique identifier", example = "1", accessMode = Schema.AccessMode.READ_ONLY)
    private Long id;

    @NotBlank(message = "Title is required")
    @Size(min = 1, max = 200)
    @Schema(description = "Book title", example = "Clean Code", required = true, maxLength = 200)
    private String title;

    @NotBlank(message = "Author is required")
    @Schema(description = "Book author", example = "Robert C. Martin", required = true)
    private String author;

    @Pattern(regexp = "^(?:ISBN(?:-1[03])?:? )?(?=[0-9X]{10}$|(?=(?:[0-9]+[- ]){3})[- 0-9X]{13}$|97[89][0-9]{10}$|(?=(?:[0-9]+[- ]){4})[- 0-9]{17}$)(?:97[89][- ]?)?[0-9]{1,5}[- ]?[0-9]+[- ]?[0-9]+[- ]?[0-9X]$")
    @Schema(description = "ISBN number", example = "978-0132350884")
    private String isbn;

    @Min(value = 0, message = "Price must be positive")
    @Schema(description = "Book price in USD", example = "29.99", minimum = "0")
    private BigDecimal price;

    @Past(message = "Publication date must be in the past")
    @Schema(description = "Publication date", example = "2008-08-01")
    private LocalDate publicationDate;

    @Schema(description = "Book description", example = "A handbook of agile software craftsmanship")
    private String description;

    @Email(message = "Publisher email must be valid")
    @Schema(description = "Publisher contact email", example = "contact@publisher.com")
    private String publisherEmail;

    // Constructors, getters, setters...
}
```

---

## Authoring reminders

- Annotate the `@RestController` directly. Do not split into an API interface + implementation.
- Do not place any Javadoc (`/** */` or `///`) on `@RestController` classes. Intent for the HTTP layer goes in `@Operation(description = "...")` as Markdown.
- Use JSR-303 Bean Validation annotations (`@NotBlank`, `@Size`, etc.) on DTOs — SpringDoc auto-generates constraint metadata from them.
- For an `@ApiResponse` to render its body schema correctly, supply `content = @Content(schema = @Schema(implementation = ...))`.
- Group endpoints by domain with `@Tag` at the class level; reserve `@Tag` overrides at the method level for cross-cutting endpoints.
- For end-to-end errors and exception authoring, see [error-response-authoring.md](error-response-authoring.md).
- For the layer split (controller vs service), see [controller-javadoc-openapi-pattern.md](controller-javadoc-openapi-pattern.md).
- For attribute tables, see [openapi-annotation-reference.md](openapi-annotation-reference.md).
