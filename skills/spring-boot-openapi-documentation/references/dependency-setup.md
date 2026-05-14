# SpringDoc Dependency Setup

## Maven Dependencies

```xml
<!-- SWAGGER -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>3.0.2</version>
</dependency>

<!-- WebFlux alternative -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
    <version>3.0.2</version>
</dependency>
```

## Version Selection

| Spring Boot | SpringDoc OpenAPI |
|---|---|
| 4.x (Java 25, Jakarta EE 11) | `3.0.2` |

Spring Boot 3.x / 2.x are out of scope for this plugin. Always check [Maven Central](https://mvnrepository.com/artifact/org.springdoc) for newer 3.x releases when upgrading.
