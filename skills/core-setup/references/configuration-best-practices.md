# Configuration Best Practices — Spring Boot 4.x

## Contents
- [Configuration Philosophy](#configuration-philosophy)
- [File Format and Layout](#file-format-and-layout)
- [Profiles — Behavior, Not Values](#profiles--behavior-not-values)
- [Environment Variables — Values, Not Behavior](#environment-variables--values-not-behavior)
- [Externalized Configuration Hierarchy](#externalized-configuration-hierarchy)
- [`@ConfigurationProperties` Records](#configurationproperties-records)
- [Validation](#validation)
- [Secrets Handling](#secrets-handling)
- [Common Presets](#common-presets)
- [Testing Configuration](#testing-configuration)
- [Configuration Metadata Processor](#configuration-metadata-processor)
- [Best Practices Checklist](#best-practices-checklist)
- [References](#references)

## Configuration Philosophy

Two axes — never mix them:

| What changes | Mechanism |
|---|---|
| **Structure / behavior** — which beans, which implementations, which auto-config flips on | **Spring Profiles** (`@Profile`, `@ConditionalOnProperty`, `spring.config.activate.on-profile`) |
| **Values** — credentials, URLs, ports, sizes, timeouts | **Environment variables** referenced from `application.yml` via `${VAR:default}` |

If `application-prod.yml` only differs from `application-dev.yml` in literal values, you do not have a profile difference — you have an environment difference. Collapse to one file with `${ENV_VAR}` placeholders so the same JAR / container image runs everywhere.

Profiles legitimately change behavior:

- `dev` → simple in-memory cache, H2, `ddl-auto: update`, `show-sql: true`
- `prod` → Hibernate L2 cache, Postgres, `ddl-auto: validate`, `show-sql: false`
- `test` → TestContainers wiring, no scheduled jobs

Profiles **should not** carry values like `prod.host=postgres-prod.internal`. That goes in the deployment's environment variables.

## File Format and Layout

`application.yml` is the single source of truth. Do not introduce `.properties` files alongside it.

```
src/main/resources/
├── application.yml                # Common config + profile blocks via ---
└── application-test.yml           # Test-only overrides (test profile)
```

Boot also accepts `application-{profile}.yml` as separate files if a profile gets large. Prefer multi-document blocks (`---`) inside one `application.yml` until size makes that painful.

**Single-file layout with profile blocks:**

```yaml
spring:
  application:
    name: ${SPRING_APPLICATION_NAME:my-service}
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}
    hikari:
      maximum-pool-size: ${HIKARI_MAX_POOL_SIZE:10}
      minimum-idle: ${HIKARI_MIN_IDLE:5}
  threads:
    virtual:
      enabled: true
  jpa:
    open-in-view: false

server:
  port: ${SERVER_PORT:8080}
  compression:
    enabled: true
    mime-types: text/html,text/xml,text/plain,text/css,application/javascript,application/json
  shutdown: graceful

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true

logging:
  level:
    root: INFO

---
spring.config.activate.on-profile: dev
spring.jpa.hibernate.ddl-auto: update
spring.jpa.show-sql: true
logging.level.com.example: DEBUG

---
spring.config.activate.on-profile: prod
spring.jpa.hibernate.ddl-auto: validate
spring.jpa.show-sql: false
logging.level.com.example: INFO
```

Notes:

- Values like the DB URL, secrets, and pool sizes use `${VAR}` — they are env-driven, not profile-driven.
- Profile blocks change behavior (`ddl-auto`, `show-sql`, log levels) — not literal values.

## Profiles — Behavior, Not Values

### Activation

Via environment variable (preferred):

```bash
export SPRING_PROFILES_ACTIVE=prod
```

Via JVM arg:

```bash
java -jar app.jar --spring.profiles.active=prod
```

Via Docker:

```bash
docker run -e SPRING_PROFILES_ACTIVE=prod -p 8080:8080 my-service:1.0.0
```

### Conditional beans

Use `@Profile` to switch implementations:

```java
@Configuration
public class NotificationConfig {

    @Bean
    @Profile("!prod")
    NotificationGateway loggingGateway() {
        return new LoggingNotificationGateway();
    }

    @Bean
    @Profile("prod")
    NotificationGateway sendgridGateway(SendgridProperties props) {
        return new SendgridNotificationGateway(props);
    }
}
```

Use `@ConditionalOnProperty` when the switch is feature-flag-shaped rather than environment-shaped:

```java
@Bean
@ConditionalOnProperty(name = "app.features.audit", havingValue = "true")
AuditListener auditListener() { return new AuditListener(); }
```

## Environment Variables — Values, Not Behavior

Reference them with placeholders. Always provide a default for safe local startup, except for true secrets:

```yaml
server:
  port: ${SERVER_PORT:8080}                  # default 8080 locally
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL}            # no default — must be supplied
    password: ${SPRING_DATASOURCE_PASSWORD}  # no default — must be supplied
```

**Naming convention:** Spring Boot maps `UPPER_SNAKE_CASE` env vars to dotted property keys. `SPRING_DATASOURCE_URL` maps to `spring.datasource.url`. You usually don't need to write the placeholder — Spring binds the env var automatically — but explicit `${...}` placeholders make the contract visible in the YAML.

**Docker Compose:**

```yaml
services:
  my-service:
    image: my-service:1.0.0
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/mydb
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}  # from .env, never committed
    env_file:
      - .env
```

**`.env` (gitignored):**

```
DB_PASSWORD=super-secret
JWT_SECRET=another-super-secret
```

## Externalized Configuration Hierarchy

Spring Boot reads configuration in this order (later sources override earlier):

1. Default properties (`SpringApplication.setDefaultProperties`)
2. `@PropertySource` annotations
3. Config data files (`application.yml`)
4. Profile-specific config (`application-{profile}.yml` or multi-doc blocks)
5. OS environment variables
6. Java system properties (`-Dkey=value`)
7. Command-line arguments

In practice: a literal in `application.yml` is the default; env vars and CLI args override at runtime.

## `@ConfigurationProperties` Records

Use immutable Java records — no setters, no `@Value` fan-out.

```java
@ConfigurationProperties("app.payments")
public record PaymentsProperties(
    URI gatewayUrl,
    Duration timeout,
    int maxRetries,
    Security security
) {
    public record Security(boolean enabled, String apiKey) {}
}
```

Enable scanning once at the main class:

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Bind from YAML:

```yaml
app:
  payments:
    gateway-url: ${PAYMENTS_GATEWAY_URL}
    timeout: 5s
    max-retries: 3
    security:
      enabled: true
      api-key: ${PAYMENTS_API_KEY}
```

Inject by type:

```java
@Service
public class PaymentsService {
    private final PaymentsProperties props;
    public PaymentsService(PaymentsProperties props) { this.props = props; }
}
```

## Validation

Apply Jakarta Validation annotations on record components. Spring runs them at startup when `@Validated` is present.

```java
@ConfigurationProperties("app.payments")
@Validated
public record PaymentsProperties(
    @NotNull URI gatewayUrl,
    @NotNull @DurationMin(seconds = 1) Duration timeout,
    @Min(0) @Max(10) int maxRetries
) {}
```

Dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

A bad value fails the application context at startup with a precise error — far better than discovering it via a `NullPointerException` three calls deep.

## Secrets Handling

**Never commit secrets.** Always reference them via env vars:

```yaml
spring:
  datasource:
    password: ${SPRING_DATASOURCE_PASSWORD}
app:
  security:
    jwt-secret: ${JWT_SECRET}
```

**Local development:**

- `.env` file with Docker Compose `env_file:` (gitignore the `.env`)
- Shell exports in `.bashrc` / `.zshrc`
- IDE run-configuration env vars

**Production:**

- Kubernetes Secrets mounted as env vars
- Spring Cloud Config Server with an encrypted backend (Vault, JCE)
- Cloud secret managers (AWS Secrets Manager, GCP Secret Manager) via sidecars or init containers

## Common Presets

### HikariCP

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: ${HIKARI_MAX_POOL_SIZE:20}
      minimum-idle: ${HIKARI_MIN_IDLE:5}
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      pool-name: ${spring.application.name}-hikari
```

### Jackson

```yaml
spring:
  jackson:
    date-format: yyyy-MM-dd'T'HH:mm:ss.SSSZ
    time-zone: UTC
    default-property-inclusion: non_null
    deserialization:
      fail-on-unknown-properties: true
      # Required when JPA entities have primitive fields. Hibernate's bytecode
      # enhancer generates constructors that Jackson 3 uses for deserialization,
      # which fails when primitive fields are absent from the payload.
      fail-on-null-for-primitives: false
```

### CORS (global)

```yaml
spring:
  web:
    cors:
      allowed-origins: ${CORS_ALLOWED_ORIGINS:https://example.com}
      allowed-methods: GET,POST,PUT,DELETE
      allowed-headers: "*"
      allow-credentials: true
      max-age: 3600
```

For fine-grained per-route CORS, use `WebMvcConfigurer#addCorsMappings` in Java config.

### Actuator (minimal production-safe set)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active:default}
```

### Server (graceful shutdown + compression)

```yaml
server:
  port: ${SERVER_PORT:8080}
  shutdown: graceful
  compression:
    enabled: true
    mime-types: text/html,text/xml,text/plain,text/css,application/javascript,application/json

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

## Testing Configuration

Create `src/test/resources/application-test.yml` for integration-test overrides:

```yaml
spring:
  datasource:
    url: jdbc:tc:postgresql:16:///testdb  # TestContainers-managed
  jpa:
    hibernate:
      ddl-auto: create-drop
  docker:
    compose:
      enabled: false                       # don't auto-start compose in tests
logging:
  level:
    com.example: DEBUG
```

Test class:

```java
@SpringBootTest
@ActiveProfiles("test")
class OrdersIntegrationTest {
    // ...
}
```

Ad-hoc overrides per test class:

```java
@SpringBootTest
@TestPropertySource(properties = {
    "app.payments.max-retries=0"
})
class PaymentsNoRetryTest { /* ... */ }
```

## Configuration Metadata Processor

Generate IDE autocomplete metadata for your custom properties:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

Combined with JEP 467 Markdown Javadoc on record components, IDEs show descriptions inline:

```java
@ConfigurationProperties("app.payments")
public record PaymentsProperties(
    /// The payments gateway base URL.
    URI gatewayUrl,

    /// Timeout for a single request. Defaults to 5s.
    Duration timeout,

    /// Number of retries on transient failures (network, 5xx).
    int maxRetries
) {}
```

## Best Practices Checklist

- [ ] One `application.yml`, YAML only — no `.properties`.
- [ ] Profiles change **behavior** (beans, conditions, auto-config). Env vars change **values** (URLs, secrets, sizes).
- [ ] All credentials and per-environment values use `${ENV_VAR}` placeholders.
- [ ] No secrets committed; `.env` and `*.local.*` are gitignored.
- [ ] Custom config uses `@ConfigurationProperties` records, not `@Value`.
- [ ] `@Validated` on properties records that have required or bounded fields.
- [ ] `@ConfigurationPropertiesScan` on the main class.
- [ ] Sensible defaults in placeholders for local startup (`${SERVER_PORT:8080}`).
- [ ] `spring-boot-configuration-processor` on the classpath for IDE metadata.
- [ ] `spring.threads.virtual.enabled: true` for new services (see `spring-async-concurrency`).
- [ ] `management.endpoints.web.exposure.include` lists only what you need — never `*` in production.

## References

- [Spring Boot Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [`@ConfigurationProperties`](https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config.typesafe-configuration-properties)
- [Profiles](https://docs.spring.io/spring-boot/reference/features/profiles.html)
- [Configuration Metadata](https://docs.spring.io/spring-boot/specification/configuration-metadata/)
