---
name: core-setup
description: "Spring Boot 4.x project configuration best practices: pom.xml structure, profile-aware application.yml, type-safe @ConfigurationProperties records, externalized config via env vars and Spring Cloud Config. Read before scaffolding a new Boot 4 service or adding configuration. Triggers: Spring Boot 4, spring-boot-starter-parent 4, Java 25, Jakarta EE 11, Spring Framework 7, application.yml, application-{profile}.yml, @ConfigurationProperties, @ConfigurationPropertiesScan, spring.profiles.active, spring.config.activate.on-profile."
version: 0.2.0
license: Apache-2.0
---

# Core Setup — Spring Boot 4.x

**Signals**: `pom.xml` referencing `spring-boot-starter-parent` 4.x, missing `application.yml`, no `@ConfigurationProperties`, scaffolding from scratch.

## Tested With

- Spring Boot 4.0.x
- Java 25 (LTS)
- Jakarta EE 11
- Spring Framework 7.x
- Maven 3.9+

## Do NOT Use This Skill When

- The project is already on Spring Boot 4 with proper `application.yml` and configuration → load topic-specific skills (e.g., `spring-boot-rest-api-standards`, `spring-data-jpa`)
- Building Docker images → use `containerization-docker`
- Configuring logging → use `observability-logging`
- Configuring security → use `spring-security-jwt`
- Setting up Spring Cloud Config Server → use `spring-cloud-discovery-config`
- Writing tests → use `spring-testing-fundamentals` and friends

## Quick Reference

**Boot 4 BOM in `pom.xml`:**
```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>4.0.0</version>
</parent>
<properties>
  <java.version>25</java.version>
</properties>
```

**Profile-aware `application.yml`:**
```yaml
spring:
  application:
    name: my-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

---
spring.config.activate.on-profile: dev
server.port: 8080

---
spring.config.activate.on-profile: prod
server.port: 80
```

**Type-safe configuration with records:**
```java
@ConfigurationProperties("app.payments")
public record PaymentsProperties(
    URI gatewayUrl,
    Duration timeout,
    int maxRetries
) {}

@SpringBootApplication
@ConfigurationPropertiesScan
public class Application { /* ... */ }
```

## Anti-patterns

- Don't carry over `javax.*` imports — Boot 3+ requires `jakarta.*`.
- Don't rely on `@Value` for groups of related properties — use `@ConfigurationProperties` records.
- Don't put secrets in `application.yml` — use environment variables or Spring Cloud Config with an encrypted backend.
- Don't pin to Java 17 / 21 for greenfield Boot 4 projects — Java 25 is the LTS aligned with Boot 4.

## References

- `references/configuration-best-practices.md` — full configuration cookbook: profile structure, externalized config hierarchy, `@ConfigurationProperties` validation, secrets handling for local + production, HikariCP / Jackson / CORS / Actuator presets, configuration metadata processor.
