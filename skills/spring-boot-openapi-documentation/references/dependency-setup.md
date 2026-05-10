# SpringDoc Dependency Setup

## Maven Dependencies

```xml
<!-- Standard WebMVC support -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.13</version>
</dependency>

<!-- Optional: therapi-runtime-javadoc for JavaDoc support -->
<dependency>
    <groupId>com.github.therapi</groupId>
    <artifactId>therapi-runtime-javadoc</artifactId>
    <version>0.15.0</version>
    <scope>provided</scope>
</dependency>

<!-- WebFlux support -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
    <version>2.8.13</version>
</dependency>
```

## Version Selection

- **Spring Boot 3.x**: Use SpringDoc 2.x (e.g., 2.8.13)
- **Spring Boot 2.x**: Use SpringDoc 1.x
- Always check for the latest stable version at [Maven Central](https://mvnrepository.com/artifact/org.springdoc)
