---
name: core-setup
description: "Spring Boot 4.x foundation: project init, application.yml profiles, @ConfigurationProperties, dotfiles. Read before scaffolding a new Boot 4 service or migrating from Boot 3.x. Triggers: Spring Boot 4, spring-boot-starter-parent 4, Java 25, Jakarta EE 11, Spring Framework 7, Hibernate 7.1, application.yml, application-{profile}.yml, @ConfigurationProperties, @ConfigurationPropertiesScan, spring.profiles.active, .editorconfig, .gitignore Java, start.spring.io, spring init."
version: 0.1.0
license: Apache-2.0
---

# Core Setup — Spring Boot 4.x

**Signals**: `pom.xml` / `build.gradle.kts` referencing `spring-boot-starter-parent` 4.x, missing `application.yml`, no `@ConfigurationProperties`, scaffolding from scratch, migrating Boot 3 → Boot 4.

## Tested With

- Spring Boot 4.0.x (GA Nov 2025)
- Java 25 (LTS)
- Jakarta EE 11
- Spring Framework 7.x
- Hibernate ORM 7.1.x
- Maven 3.9+ / Gradle 8.10+

## Do NOT Use This Skill When

- The project is already on Spring Boot 4 with proper `application.yml` and configuration → load topic-specific skills (e.g., `spring-boot-rest-api-standards`, `spring-data-jpa`)
- Building Docker images → use `containerization-docker`
- Configuring logging → use `observability-logging`
- Configuring security → use `spring-security-jwt`
- Writing tests → use `spring-testing-fundamentals` and friends

## When to Read References

| Situation | Read |
|-----------|------|
| Migrating from Boot 3.x to Boot 4.x: deprecated APIs, config keys, dependencies | `references/spring-boot-4-migration.md` |
| New project scaffold, `start.spring.io` parameters, recommended starters | `references/spring-boot-4-migration.md` |
| `application.yml` structure, profiles, externalized config, `@ConfigurationProperties` records | `references/configuration-best-practices.md` |
| Property precedence, secrets handling, env-var binding, Spring Cloud Config | `references/configuration-best-practices.md` |
| Repo dotfiles (`.editorconfig`, `.gitignore`, `.gitattributes`), formatting (Spotless), pre-commit hooks | `references/dotfiles-and-project-init.md` |
| GitHub Actions CI baseline for a Java service | `references/dotfiles-and-project-init.md` |

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

**Type-safe configuration with records (Boot 3+, idiomatic in Boot 4):**
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
- Don't put secrets in `application.yml` — use Spring Cloud Config, AWS Secrets Manager, or env vars.
- Don't pin to Java 17 / 21 for greenfield Boot 4 projects — Java 25 is the LTS aligned with Boot 4.
