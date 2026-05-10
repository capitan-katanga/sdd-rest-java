---
name: spring-cloud-discovery-config
description: "Spring Cloud microservices base: service discovery with Eureka/Consul, centralized configuration with Spring Cloud Config Server, and client-side load balancing with Spring Cloud LoadBalancer. Read before standing up a service registry, bootstrapping a Config Server, refreshing properties at runtime, or wiring a load-balanced client. Triggers: @EnableEurekaServer, @EnableDiscoveryClient, eureka.client.service-url.defaultZone, spring.cloud.consul.host, spring.cloud.consul.discovery, @EnableConfigServer, spring.cloud.config.server.git.uri, spring.config.import=configserver:, @RefreshScope, /actuator/refresh, /actuator/busrefresh, @LoadBalanced, LoadBalancerClient, ServiceInstanceListSupplier, ReactiveLoadBalancer."
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
version: 0.1.0
license: Apache-2.0
---

# Spring Cloud — Service Discovery & Centralized Configuration

Foundation for Spring Cloud microservices on Spring Boot 4.x: a registry where services advertise themselves, a Config Server that hands out per-service configuration from a Git backend, and client-side load balancing for service-to-service calls.

## Tested With

- Spring Boot 4.0.x
- Spring Cloud release train aligned with Boot 4 (Spring Cloud 2025.x — check the BOM against your Boot version)
- Java 25
- Netflix Eureka 4.x (server + client)
- HashiCorp Consul 1.18+ (as alternative registry)
- Spring Cloud LoadBalancer (Ribbon is gone — do not bring it back)

## Do NOT Use This Skill When

- Routing HTTP traffic from an edge gateway → use `spring-cloud-gateway`.
- Building declarative HTTP clients for service-to-service calls → use `spring-http-interface-clients`. (This skill only covers the `@LoadBalanced` infrastructure that those clients sit on top of.)
- Configuring resilience around outbound calls (circuit breakers, retries) → use `spring-boot-resilience4j`.
- Bootstrapping a brand-new Boot 4 service from scratch → use `core-setup` first, then come back here.

## When to Read References

| Situation | Read |
|-----------|------|
| Standing up an Eureka **server** (HA pair, peer awareness, self-preservation tuning) | `references/eureka-server.md` |
| Registering a **client** against Eureka or Consul, health check integration | `references/discovery-clients.md` |
| Bootstrapping a **Config Server** with Git backend, native profile, encryption (`{cipher}`) | `references/config-server.md` |
| Config **client** side: `spring.config.import=configserver:`, fail-fast, retry, `@RefreshScope` | `references/config-client.md` |
| Spring Cloud **Bus** for broadcasting `/actuator/busrefresh` across the fleet | `references/config-bus.md` |
| Custom `ServiceInstanceListSupplier` (weighted, zone-aware, hint-based) | `references/loadbalancer-customization.md` |

> References are not auto-loaded. Cite the relevant one explicitly before reaching for it.

## Quick Reference

**Eureka server (single node, dev profile):**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```
```java
@SpringBootApplication
@EnableEurekaServer
public class RegistryApplication { /* ... */ }
```
```yaml
server.port: 8761
spring.application.name: discovery-server
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

**Eureka client (registering a service):**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
```yaml
spring.application.name: orders-service
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${random.value}
```

**Consul client (alternative registry):**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```
```yaml
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        prefer-ip-address: true
        health-check-path: /actuator/health
        health-check-interval: 10s
```

**Config Server (Git backend):**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { /* ... */ }
```
```yaml
server.port: 8888
spring:
  application.name: config-server
  cloud.config.server.git:
    uri: https://github.com/my-org/config-repo
    default-label: main
    search-paths: '{application}'
```

**Config client (Boot 3+/4 style — no more `bootstrap.yml` required):**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```
```yaml
spring:
  application.name: orders-service
  config.import: "optional:configserver:http://localhost:8888"
  cloud.config:
    fail-fast: true
    retry:
      initial-interval: 1000
      max-attempts: 6
```

**`@RefreshScope` for hot-reloading beans:**
```java
@RefreshScope
@Component
public class FeatureFlags {
    @Value("${features.checkout.v2:false}")
    private boolean checkoutV2Enabled;
}
```
After editing the Git repo: `POST /actuator/refresh` on the client (or `/actuator/busrefresh` if Spring Cloud Bus is wired).

**Load-balanced `RestClient`:**
```java
@Configuration
public class ClientConfig {
    @Bean
    @LoadBalanced
    RestClient.Builder restClientBuilder() {
        return RestClient.builder();
    }
}

@Service
public class InventoryGateway {
    private final RestClient client;

    InventoryGateway(RestClient.Builder builder) {
        this.client = builder.baseUrl("http://inventory-service").build();
    }

    public Stock fetch(long sku) {
        return client.get().uri("/stock/{sku}", sku).retrieve().body(Stock.class);
    }
}
```
`http://inventory-service` resolves via the registry — no hardcoded host/port.

## Instructions

1. **Pick a registry** (Eureka or Consul). Eureka is simpler when the team already runs JVM infrastructure; Consul is preferable when the org runs a multi-language fleet or already has HashiCorp tooling.
2. **Stand up the registry first.** It must be reachable before any client starts, otherwise clients will fail-fast on registration (or retry on a backoff). For dev, run it via the same `compose.yaml` your services use (see `containerization-docker`).
3. **Register clients with `spring.application.name`** — that name is what other services look up via load balancer URIs. Treat it as a stable contract, not a debug string.
4. **Wire `/actuator/health` as the registry health check** so the registry can deregister unhealthy instances quickly.
5. **Stand up the Config Server** once the registry is healthy. The Config Server itself can be a regular client of the registry (so other services can discover it instead of hardcoding `http://localhost:8888`). Set `spring.cloud.config.discovery.enabled=true` on the client to enable that.
6. **Move per-service configuration into the Git repo** with one file per service (`orders-service.yml`, `inventory-service.yml`) and one `application.yml` for shared defaults. Profiles add suffixes (`orders-service-prod.yml`).
7. **Mark hot-reloadable beans with `@RefreshScope`.** Static beans (datasources, security filters) cannot refresh — they require a restart. Plan around that.
8. **Encrypt secrets** in the Git repo using `{cipher}…` syntax and a configured encryption key on the Config Server. Never commit plaintext credentials.
9. **Annotate the `RestClient.Builder` / `WebClient.Builder` bean with `@LoadBalanced`** so URIs like `http://orders-service` resolve through the registry. Without it, those hostnames will fail DNS lookup.

## Examples

### Load-balanced reactive `WebClient`
```java
@Configuration
public class ReactiveClientConfig {
    @Bean
    @LoadBalanced
    WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}
```
Reactive callers (WebFlux, Gateway) get the same `http://<service-name>` resolution.

### Custom `ServiceInstanceListSupplier` (zone-aware)
```java
@Configuration
public class LoadBalancerConfig {
    @Bean
    ServiceInstanceListSupplier zoneAwareSupplier(ConfigurableApplicationContext ctx) {
        return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()
            .withCaching()
            .withZonePreference()
            .build(ctx);
    }
}
```
See `references/loadbalancer-customization.md` for hint-based and weighted variants.

## Best Practices

- **Pin `spring.application.name`** before anything else. Renaming it later breaks every consumer using `http://<service-name>` URIs.
- **Always set `eureka.instance.prefer-ip-address: true`** in containerized environments. Hostnames inside containers rarely match what other services can reach.
- **Set `eureka.instance.lease-renewal-interval-in-seconds: 10`** for faster failure detection in dev (default 30s is too slow when debugging). Tune up in prod to reduce registry load.
- **Use one Config Server profile per environment** (`dev`, `staging`, `prod`) and bind the client to its environment via `spring.profiles.active`. Don't fork the Git repo by branch.
- **`@RefreshScope` is opt-in.** Apply it to beans that hold config values you want to mutate at runtime — feature flags, throttling limits, external endpoints. Don't blanket-annotate everything; rescoped beans are recreated on each refresh and lose internal state.
- **Health checks belong on the registry, not the client.** Let the registry deregister instances that fail their `/actuator/health` poll. Clients should not maintain their own liveness map.

## Anti-patterns

- Don't hardcode `http://localhost:8888` in services for production. Use `spring.cloud.config.discovery.enabled=true` and let the Config Server itself be a registry client.
- Don't ship plaintext secrets in the config Git repo. Either use `{cipher}` envelope encryption (built into Config Server) or front the repo with HashiCorp Vault as a backend.
- Don't keep `bootstrap.yml` if you're on Boot 3+/4 — `spring.config.import` is the supported import mechanism. The legacy bootstrap context still works but adds startup complexity for no benefit.
- Don't reintroduce Netflix Ribbon. It's been removed from Spring Cloud since the 2020 release train. Use Spring Cloud LoadBalancer.
- Don't share the Config Server's encryption key across environments. Per-env keys mean compromise of dev doesn't leak prod secrets.

## Related Skills

- `spring-cloud-gateway` — Edge gateway that uses this skill's discovery client + LoadBalancer.
- `spring-http-interface-clients` — Declarative HTTP clients built on the `@LoadBalanced` infrastructure here.
- `spring-boot-resilience4j` — Wrap load-balanced calls with circuit breaker / retry / timeout.
- `spring-boot-actuator` — `/actuator/refresh`, `/actuator/busrefresh`, registry health probes.
- `core-setup` — Profile-aware `application.yml`, `@ConfigurationProperties` patterns that consume Config Server values.
