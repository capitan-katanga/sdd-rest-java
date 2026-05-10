---
name: spring-cloud-gateway
description: "Spring Cloud Gateway as edge service for a microservices fleet: route definition (YAML and programmatic RouteLocator), predicates (Path, Host, Method, Header, Query), filters (RewritePath, AddRequestHeader, RemoveResponseHeader, CircuitBreaker, RequestRateLimiter, Retry), discovery-backed routing with lb:// URIs, JWT auth at the edge, CORS, and global filters. Targets Spring Boot 4 / Spring Cloud 2025.x. Triggers: spring-cloud-starter-gateway, spring-cloud-starter-gateway-mvc, RouteLocator, RouteLocatorBuilder, RouteDefinitionLocator, spring.cloud.gateway.routes, GatewayFilter, GlobalFilter, GatewayFilterFactory, lb://, @LoadBalanced, predicates: - Path=, filters: - RewritePath, AddRequestHeader, RequestRateLimiter, RedisRateLimiter, CircuitBreaker, SpringCloudCircuitBreakerFilterFactory, ServerWebExchange."
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
version: 0.1.0
license: Apache-2.0
---

# Spring Cloud Gateway

API gateway / edge service for Spring Cloud microservices. Sits in front of the fleet and handles routing, cross-cutting concerns (auth, CORS, rate limiting, retries), and protocol bridging.

## Tested With

- Spring Boot 4.0.x
- Spring Cloud 2025.x (release train aligned with Boot 4 — confirm via the Spring Cloud BOM)
- Java 25
- Reactor Netty (default server for the WebFlux flavor)
- Tomcat 11 (for the MVC flavor)

## Which flavor?

Spring Cloud Gateway ships two stacks. **Pick one per gateway service, do not mix dependencies.**

| Flavor | Starter artifact | Runtime stack | When to pick it |
|---|---|---|---|
| Reactive (default) | `spring-cloud-starter-gateway` | WebFlux + Netty | High concurrency, long-lived connections, streaming, websocket proxy. The historical and most-featured flavor. |
| Servlet (MVC) | `spring-cloud-starter-gateway-mvc` | Spring MVC + Tomcat | Existing Servlet/MVC ecosystem, simpler debugging, blocking filter logic. Newer; smaller filter surface than the reactive flavor. |

This skill leads with the **reactive** flavor (Netty). The MVC flavor uses the same predicates but exposes filters as `HandlerFilterFunction`. See `references/gateway-mvc.md` for the MVC-specific syntax.

## Do NOT Use This Skill When

- Building service-to-service HTTP clients **inside** a microservice — use `spring-http-interface-clients` (`@HttpExchange`) and the `@LoadBalanced` infrastructure from `spring-cloud-discovery-config`.
- Standing up the registry or Config Server → use `spring-cloud-discovery-config`.
- Configuring resilience patterns **inside** a service (circuit breaker around outbound DB / Kafka calls) → use `spring-boot-resilience4j`. Gateway has its own resilience filter that wraps inbound HTTP only.
- Implementing JWT auth **inside** a downstream service → use `spring-security-jwt`. (Gateway can do JWT validation at the edge, but downstream services still need their own SecurityFilterChain to trust the propagated identity.)

## When to Read References

| Situation | Read |
|-----------|------|
| Servlet (MVC) gateway syntax: `route()`, `route(GET("/api/**"), http(...))` | `references/gateway-mvc.md` |
| Custom `GatewayFilterFactory` / `GlobalFilter` with `ServerWebExchange` mutation | `references/custom-filters.md` |
| `RedisRateLimiter` setup: Redis keyspace, replenish-rate, burst-capacity, key resolver per principal | `references/rate-limiting.md` |
| Circuit-breaker filter wired to Resilience4j (`CircuitBreaker=fallbackUri`) | `references/circuit-breaker-filter.md` |
| JWT auth at the edge with `oauth2ResourceServer` and identity propagation to downstreams | `references/jwt-at-edge.md` |
| WebSocket / SSE proxying considerations | `references/websocket-proxy.md` |

## Quick Reference

**Maven (reactive flavor):**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

**YAML-defined routes:**
```yaml
spring:
  application.name: edge-gateway
  cloud:
    gateway:
      discovery.locator.enabled: false   # off in prod; explicit routes are clearer
      routes:
        - id: orders-api
          uri: lb://orders-service
          predicates:
            - Path=/api/orders/**
          filters:
            - RewritePath=/api/orders/(?<segment>.*), /orders/${segment}
            - AddRequestHeader=X-Forwarded-Prefix, /api/orders
        - id: inventory-api
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
            - Method=GET,POST
          filters:
            - name: CircuitBreaker
              args:
                name: inventoryCircuit
                fallbackUri: forward:/fallback/inventory
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY,GATEWAY_TIMEOUT
                methods: GET
                backoff:
                  firstBackoff: 50ms
                  maxBackoff: 500ms
                  factor: 2
```
`lb://orders-service` resolves through `spring-cloud-discovery-config` — the registry must be reachable.

**Programmatic routes (`RouteLocator` bean):**
```java
@Configuration
public class RouteConfig {
    @Bean
    RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("orders-api", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .rewritePath("/api/orders/(?<seg>.*)", "/orders/${seg}")
                    .addRequestHeader("X-Forwarded-Prefix", "/api/orders")
                    .circuitBreaker(c -> c.setName("ordersCircuit").setFallbackUri("forward:/fallback/orders")))
                .uri("lb://orders-service"))
            .build();
    }
}
```
Programmatic config wins when routes depend on Java logic (e.g., one route per tenant). YAML wins for simple static routing.

**CORS (global, applies to every route):**
```yaml
spring.cloud.gateway.globalcors.cors-configurations:
  '[/**]':
    allowed-origins: "https://app.example.com"
    allowed-methods: GET,POST,PUT,DELETE,OPTIONS
    allowed-headers: "*"
    allow-credentials: true
    max-age: 3600
```

**Rate limiter (Redis-backed, per principal):**
```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenish-rate: 20
      redis-rate-limiter.burst-capacity: 40
      key-resolver: "#{@principalKeyResolver}"
```
```java
@Bean
KeyResolver principalKeyResolver() {
    return exchange -> exchange.getPrincipal()
        .map(Principal::getName)
        .switchIfEmpty(Mono.just("anonymous"));
}
```

## Instructions

1. **Decide the flavor (reactive vs MVC) before adding any starter.** Mixing both starters or pulling Boot Web alongside Boot WebFlux causes startup failures.
2. **Register the gateway with the discovery service** (`spring-cloud-starter-netflix-eureka-client` or Consul) so it can resolve `lb://<service>` URIs. Without a registry client, the gateway can only proxy to absolute URIs.
3. **Define routes — YAML for static, Java for dynamic.** Don't mix the two for the same route ID. Java routes win on conflict but the override is silent and confusing.
4. **Add filters in order of dependency.** Rate limit → auth → routing → retries → circuit breaker is a typical chain; circuit breaker should generally wrap the actual downstream call (innermost).
5. **Enable Actuator's `gateway` endpoint** (`management.endpoint.gateway.access=read-only`) to inspect routes, filters, and global filters at runtime. Production-safe — it does not expose secrets.
6. **Wire JWT at the edge** if the downstream services trust the gateway. Use `spring-security-jwt` patterns adapted to WebFlux (`SecurityWebFilterChain` instead of `SecurityFilterChain`). See `references/jwt-at-edge.md`.
7. **For rate limiting, use Redis** (`spring-boot-starter-data-redis-reactive`). The in-memory rate limiter only works for single-node gateways and is unsafe to rely on under load balancers.
8. **Tune Netty timeouts** explicitly. Defaults are often too generous for client-facing edges:
   ```yaml
   spring.cloud.gateway.httpclient:
     connect-timeout: 2000
     response-timeout: 5s
     pool:
       max-idle-time: 30s
   ```

## Examples

### Route with circuit breaker + fallback
```java
@RestController
class FallbackController {
    @GetMapping("/fallback/orders")
    Mono<Map<String, String>> ordersFallback() {
        return Mono.just(Map.of(
            "status", "DEGRADED",
            "message", "Orders temporarily unavailable. Try again shortly."
        ));
    }
}
```
The `CircuitBreaker` filter routes to `forward:/fallback/orders` when the downstream call fails or the breaker is open.

### Header-stripping for sensitive internals
```yaml
filters:
  - RemoveRequestHeader=X-Internal-Auth
  - RemoveResponseHeader=Server
  - DedupeResponseHeader=Access-Control-Allow-Origin, RETAIN_UNIQUE
```
Defensive default. Add these globally if you don't want internal headers leaking to clients.

### `GlobalFilter` for request-scoped tracing
```java
@Component
public class CorrelationIdFilter implements GlobalFilter, Ordered {

    public static final String HEADER = "X-Correlation-Id";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String correlationId = Optional.ofNullable(exchange.getRequest().getHeaders().getFirst(HEADER))
            .orElseGet(() -> UUID.randomUUID().toString());

        ServerWebExchange mutated = exchange.mutate()
            .request(r -> r.header(HEADER, correlationId))
            .build();

        return chain.filter(mutated)
            .doFinally(s -> mutated.getResponse().getHeaders().add(HEADER, correlationId));
    }

    @Override public int getOrder() { return Ordered.HIGHEST_PRECEDENCE; }
}
```
Pairs naturally with `observability-logging` (MDC propagation) and `spring-opentelemetry-tracing` (span correlation).

## Best Practices

- **Pin route IDs.** The `id` field is what shows up in metrics, Actuator's `/actuator/gateway/routes`, and logs. Make them descriptive (`orders-api`, not `route1`).
- **Disable `discovery.locator.enabled` in production.** Auto-generated routes from the registry are convenient in dev but expose every registered service to the outside world. Define explicit routes.
- **Always set `response-timeout`** at the `httpclient` level. Without it, slow downstreams can pin Netty threads indefinitely.
- **Keep filter ordering deterministic.** Use the explicit `order` attribute on custom `GatewayFilter`s when chain order matters. Default ordering can shift between Spring Cloud releases.
- **Push tracing (`traceparent`) headers through.** Spring Cloud Gateway integrates with Micrometer Tracing — see `spring-opentelemetry-tracing`. Don't strip W3C trace headers.
- **Run the gateway as its own deployment.** Co-locating it with a business service breaks the "edge / not edge" boundary and complicates scaling.

## Anti-patterns

- Don't put business logic in `GlobalFilter`s. They run for every request; a `GatewayFilter` scoped to one route is cheaper and easier to reason about.
- Don't rely on the in-memory rate limiter in a load-balanced gateway deployment. Each instance has its own counter, so the actual rate is `(replenish-rate × instance-count)`.
- Don't proxy WebSocket / SSE without setting `httpclient.wiretap` off and increasing `pool.max-idle-time` — defaults can break long-lived streams.
- Don't add `spring-boot-starter-web` to a reactive gateway. It pulls Tomcat, which conflicts with Netty. The reactor stack is non-blocking by design.
- Don't terminate TLS at the gateway and then forward credentials in plaintext. Use mTLS or signed JWTs for the gateway → service hop.

## Related Skills

- `spring-cloud-discovery-config` — Registry + Config Server that `lb://` URIs depend on.
- `spring-http-interface-clients` — Downstream service-to-service calls (different concern: this skill is for inbound edge, that one for outbound clients).
- `spring-boot-resilience4j` — The library backing `CircuitBreaker` and `Retry` filters.
- `spring-security-jwt` — JWT patterns; adapt to `SecurityWebFilterChain` for the reactive gateway.
- `spring-opentelemetry-tracing` — End-to-end trace propagation across the gateway boundary.
- `observability-logging` — Correlation IDs (the `GlobalFilter` example above feeds MDC).
