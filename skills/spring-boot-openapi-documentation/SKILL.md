---
name: spring-boot-openapi-documentation
description: Wiring and configuration for SpringDoc OpenAPI 3 (springdoc-openapi-starter-webmvc-ui 3.0.2) in Spring Boot 4.x applications — Maven dependency setup, application.yml configuration, Swagger UI customization, security scheme plumbing, Pageable @ParameterObject support, build plugins, API groups, troubleshooting. Use when installing SpringDoc, configuring Swagger UI access paths, wiring SecurityScheme beans, plumbing pagination, integrating build pipelines, or diagnosing SpringDoc runtime issues. For ANNOTATION AUTHORING on @RestController and DTO classes (how to write @Tag, @Operation, @ApiResponse, @Parameter, @Schema, error responses, custom exception intent), use java-code-documentation-conventions instead. Targets Boot 4 / Java 25 / Jakarta EE 11.
version: 1.0.0
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Spring Boot OpenAPI Documentation — Wiring and Configuration

## Overview

SpringDoc OpenAPI automates generation of OpenAPI 3.0 documentation for Spring Boot projects and exposes a Swagger UI web interface for exploring and testing APIs.

This skill covers **only the wiring side**: dependency installation, `application.yml` configuration, Swagger UI customization, security scheme bean configuration, pagination plumbing, build integration, API grouping, and troubleshooting. It deliberately does not cover how to *write* OpenAPI annotations on classes — that lives in `java-code-documentation-conventions`.

## When to Use

- Set up SpringDoc OpenAPI in Spring Boot 4.x projects
- Configure and customize Swagger UI access paths and features
- Wire `SecurityScheme` beans for JWT, OAuth2, Basic Auth, or API key authentication
- Configure pagination support via `@ParameterObject` and custom `Page` DTOs
- Set up multiple API groups and version segmentation
- Customize the OpenAPI bean programmatically (info, contact, license, servers)
- Integrate OpenAPI generation into Maven build plugins or CI pipelines
- Diagnose SpringDoc runtime issues (missing endpoints, schema errors, Swagger UI failures)
- Hide internal endpoints from the generated spec

## Do NOT Use This Skill When

- Writing or reviewing OpenAPI annotations on `@RestController` classes (`@Tag`, `@Operation`, `@ApiResponse`, `@Parameter`) → use `java-code-documentation-conventions`
- Writing or reviewing `@Schema` annotations on DTO records and entities → use `java-code-documentation-conventions`
- Documenting error response codes, constraint violations, or custom exception intent → use `java-code-documentation-conventions`
- Looking up attribute tables for individual annotations → see `java-code-documentation-conventions/references/openapi-annotation-reference.md`
- Designing endpoints, DTOs, error envelopes, or HATEOAS contracts → use `spring-boot-rest-api-standards`

## Quick Reference

| Concept | Description |
|---------|-------------|
| **Dependencies** | `springdoc-openapi-starter-webmvc-ui` for WebMvc, `springdoc-openapi-starter-webflux-ui` for WebFlux |
| **Configuration** | `application.yml` with `springdoc.api-docs.*` and `springdoc.swagger-ui.*` properties |
| **Access Points** | OpenAPI JSON: `/v3/api-docs`, Swagger UI: `/swagger-ui/index.html` |
| **Security wiring** | Configure `SecurityScheme` in `OpenAPI` bean; controllers reference it via `@SecurityRequirement` |
| **Pagination** | Use `@ParameterObject` with Spring Data `Pageable` |

## Instructions

### 1. Add Dependencies

Add the SpringDoc starter for your application type (WebMvc or WebFlux). See [dependency-setup.md](references/dependency-setup.md) for Maven configuration.

### 2. Configure SpringDoc

Set basic configuration in `application.yml`:

```yaml
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
    operationsSorter: method
```

See [configuration.md](references/configuration.md) for advanced options.

### 3. Authoring annotations on controllers and models

This skill does not cover how to write `@Tag`, `@Operation`, `@ApiResponse`, `@Parameter`, or `@Schema` annotations. For controller and DTO authoring patterns, see [`java-code-documentation-conventions/references/openapi-authoring-patterns.md`](../java-code-documentation-conventions/references/openapi-authoring-patterns.md). For the attribute-by-attribute reference, see [`openapi-annotation-reference.md`](../java-code-documentation-conventions/references/openapi-annotation-reference.md).

### 4. Configure Security

Set up security schemes in the OpenAPI bean:

```java
@Bean
public OpenAPI customOpenAPI() {
    return new OpenAPI()
        .components(new Components()
            .addSecuritySchemes("bearer-jwt", new SecurityScheme()
                .type(SecurityScheme.Type.HTTP)
                .scheme("bearer")
                .bearerFormat("JWT")
            )
        );
}
```

Controllers reference the scheme by name with `@SecurityRequirement(name = "bearer-jwt")`. See [security-configuration.md](references/security-configuration.md) for full wiring.

### 5. Configure Pagination

Spring Data `Pageable` is surfaced to the OpenAPI spec via `@ParameterObject`:

```java
@GetMapping("/paginated")
public Page<Book> findAll(@ParameterObject Pageable pageable) {
    return repository.findAll(pageable);
}
```

See [pagination-support.md](references/pagination-support.md).

### 6. Test Documentation

Access Swagger UI at `/swagger-ui/index.html` to verify the wiring is correct and the endpoints render.

### 7. Customize for Production

Configure API grouping, versioning, and build plugins. See [advanced-configuration.md](references/advanced-configuration.md) and [build-integration.md](references/build-integration.md).

## Best Practices

- **Hide internal endpoints appropriately**: Use `@Hidden`, `@Operation(hidden = true)`, or a separate API group rather than relying on the absence of annotations.
- **Customize Swagger UI for better UX**: Enable filtering, sorting, try-it-out, and request duration display via the `springdoc.swagger-ui.*` properties.
- **Version your API documentation**: Include the API version in the `OpenAPI` bean's `Info` block and align it with the deployed artifact version.
- **Use API groups to segment large APIs**: A single OpenAPI document with hundreds of operations degrades Swagger UI performance — group by domain or version.

## References

- **[dependency-setup.md](references/dependency-setup.md)** — Maven dependencies and version selection
- **[configuration.md](references/configuration.md)** — Basic and advanced configuration options
- **[security-configuration.md](references/security-configuration.md)** — JWT, OAuth2, Basic Auth, API key configuration
- **[pagination-support.md](references/pagination-support.md)** — Pageable, Slice, and custom pagination plumbing
- **[advanced-configuration.md](references/advanced-configuration.md)** — API groups, customizers, OpenAPI bean configuration
- **[exception-handling.md](references/exception-handling.md)** — `@RestControllerAdvice` handler wiring (the authoring side lives in `java-code-documentation-conventions/references/error-response-authoring.md`)
- **[build-integration.md](references/build-integration.md)** — Maven plugins and CI/CD integration
- **[springdoc-official.md](references/springdoc-official.md)** — Official SpringDoc documentation
- **[troubleshooting.md](references/troubleshooting.md)** — Common runtime issues and solutions

## Constraints and Warnings

- Large API definitions can impact Swagger UI performance; consider grouping APIs by domain or version
- Schema generation may not work correctly with complex generic types; the wiring side surfaces the limitation, the authoring side (in `java-code-documentation-conventions`) decides how to express the missing schema
- Security schemes must be configured in the `OpenAPI` bean **before** controllers reference them by name via `@SecurityRequirement`
- Hidden endpoints (`@Operation(hidden = true)`) are still visible in code and may leak through other documentation tools

## Related Skills

- `java-code-documentation-conventions` — Authoring guide for OpenAPI annotations (`@Tag`, `@Operation`, `@ApiResponse`, `@Parameter`, `@Schema`) and JEP 467 `///` Javadoc on services, repositories, and custom exceptions. This skill handles only the SpringDoc mechanics; that skill handles the in-code documentation contract.
- `spring-boot-rest-api-standards` — REST API design standards (endpoints, DTOs, error envelopes, status codes)
- `spring-boot-dependency-injection` — Dependency injection patterns
- `spring-mvc-testing` — Testing REST controllers
- `spring-boot-actuator` — Production monitoring and management

## External Resources

- [SpringDoc Official Documentation](https://springdoc.org/)
- [OpenAPI 3.0 Specification](https://swagger.io/specification/)
- [Swagger UI Configuration](https://swagger.io/docs/open-source-tools/swagger-ui/usage/configuration/)
