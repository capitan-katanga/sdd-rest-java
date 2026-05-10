---
name: spring-http-interface-clients
description: "Declarative HTTP clients with Spring Framework 7 HTTP Interface: define service-to-service contracts as @HttpExchange-annotated interfaces, generate proxies with HttpServiceProxyFactory + RestClientAdapter (blocking) or WebClientAdapter (reactive), integrate with @LoadBalanced for service discovery, propagate headers, handle errors, set timeouts, and unit-test the interfaces. Replaces OpenFeign on Boot 4. Triggers: @HttpExchange, @GetExchange, @PostExchange, @PutExchange, @DeleteExchange, @PatchExchange, HttpServiceProxyFactory, RestClientAdapter, WebClientAdapter, RestClientHttpServiceGroupConfigurer, @ImportHttpServices, HttpServiceGroup, @RequestParam, @PathVariable, @RequestHeader, @RequestBody, @CookieValue."
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
version: 0.1.0
license: Apache-2.0
---

# Spring HTTP Interface Clients (@HttpExchange)

Declarative HTTP clients native to Spring Framework 6+ / Boot 3+. You write a Java interface annotated with `@HttpExchange`, Spring generates the proxy. **This is the supported replacement for OpenFeign on Boot 4** — no extra dependency beyond `spring-web` (or `spring-webflux` for reactive).

## Tested With

- Spring Boot 4.0.x
- Spring Framework 7.x
- Java 25
- `RestClient` (blocking) for MVC / synchronous services
- `WebClient` (reactive) for WebFlux services

## Do NOT Use This Skill When

- The client is one-off and you'd rather inline the call → use `RestClient` directly (no need for a declarative interface).
- The service-to-service call goes through an external API with hand-rolled auth flows / signing → use `RestClient`/`WebClient` directly so you have full control over the request builder.
- You're building **the gateway** (inbound routing) → use `spring-cloud-gateway`. This skill is for **outbound** calls from a service.
- Migrating an existing OpenFeign codebase → that's a refactor topic; the skill `spring-cloud-openfeign` does **not exist** in this plugin by design. Treat OpenFeign code as legacy to port to `@HttpExchange`.

## Quick Reference

**Define the contract (interface):**
```java
public interface InventoryClient {

    @GetExchange("/inventory/{sku}")
    Stock fetch(@PathVariable long sku);

    @PostExchange("/inventory/reserve")
    ReservationResult reserve(@RequestBody ReserveCommand cmd,
                              @RequestHeader("X-Tenant-Id") String tenant);

    @DeleteExchange("/inventory/reservations/{id}")
    void cancel(@PathVariable UUID id);
}
```
No annotation on the type itself is required, but you can hoist a common base path:
```java
@HttpExchange("/api/v1")
public interface InventoryClient { /* ... */ }
```

**Wire the proxy (blocking — `RestClient`):**
```java
@Configuration
public class InventoryClientConfig {

    @Bean
    InventoryClient inventoryClient(RestClient.Builder builder) {
        RestClient client = builder
            .baseUrl("http://inventory-service")  // resolved by @LoadBalanced
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .requestInterceptor(new TimingInterceptor())
            .build();

        return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(client))
            .build()
            .createClient(InventoryClient.class);
    }
}
```

**Wire the proxy (reactive — `WebClient`):**
```java
@Bean
InventoryClient reactiveInventoryClient(WebClient.Builder builder) {
    WebClient client = builder.baseUrl("http://inventory-service").build();
    return HttpServiceProxyFactory
        .builderFor(WebClientAdapter.create(client))
        .build()
        .createClient(InventoryClient.class);
}
```
Interface methods can return `Mono<T>` / `Flux<T>` when backed by `WebClientAdapter`.

**Load-balanced builder (requires `spring-cloud-discovery-config`):**
```java
@Configuration
public class ClientConfig {
    @Bean
    @LoadBalanced
    RestClient.Builder loadBalancedRestClientBuilder() {
        return RestClient.builder();
    }
}
```
After this, `baseUrl("http://inventory-service")` resolves via the registry.

**`@ImportHttpServices` (Boot 4 ergonomic shortcut):**

Boot 4 ships `@ImportHttpServices`, which lets you register multiple HTTP interfaces in one place and select the underlying client per group. Use it when you have several interfaces sharing the same base URL / timeouts / interceptors.

```java
@Configuration
@ImportHttpServices(
    group = "inventory",
    types = { InventoryClient.class, StockAuditClient.class }
)
public class InventoryGroupConfig { }
```
```yaml
spring.http.client.service:
  group:
    inventory:
      base-url: "http://inventory-service"
      default-headers:
        Accept: application/json
      connect-timeout: 2s
      read-timeout: 5s
      client-type: rest_client          # or web_client / rest_template
```
Customize a group programmatically via `RestClientHttpServiceGroupConfigurer`:
```java
@Bean
RestClientHttpServiceGroupConfigurer tracingConfigurer() {
    return groups -> groups
        .filterByName("inventory")
        .forEachClient((meta, builder) ->
            builder.requestInterceptor(new TraceContextPropagationInterceptor()));
}
```

## Instructions

1. **Define one interface per downstream service** (`InventoryClient`, `OrdersClient`). One method per endpoint. Avoid god-interfaces with 30 methods — split by aggregate / use case.
2. **Use HTTP method annotations** (`@GetExchange`, `@PostExchange`, etc.) rather than the generic `@HttpExchange(method=…)`. The specific annotations are clearer and produce better IDE hints.
3. **Parameter binding mirrors Spring MVC's:** `@PathVariable`, `@RequestParam`, `@RequestHeader`, `@RequestBody`, `@CookieValue`. Any parameter without an annotation defaulting to body in POST/PUT is **not** safe — annotate everything explicitly.
4. **For service discovery, mark the `RestClient.Builder` bean as `@LoadBalanced`** and use the logical service name in `baseUrl`. Without `@LoadBalanced`, `http://inventory-service` will fail DNS resolution.
5. **Pin timeouts.** `RestClient` has no useful default read timeout. Set them at the underlying `ClientHttpRequestFactory` (`JdkClientHttpRequestFactory`, `ReactorClientHttpConnector`, etc.) or via `@ImportHttpServices` group config.
6. **Add interceptors for cross-cutting concerns** (auth header, correlation ID, tracing). Don't bake them into every interface method.
7. **Handle errors with `RestClient.defaultStatusHandler`** or `WebClient.defaultStatusHandler`. The proxy will propagate the underlying client's exception — use Problem Details parsing (`RFC 7807`) where applicable.
8. **Unit-test the contract** with `MockRestServiceServer` (for `RestClient`) or `MockWebServer` / `WebTestClient` (for `WebClient`). The proxy is a real client invocation, so tests must mock the HTTP layer.

## Examples

### Reactive interface with `Mono`/`Flux`
```java
public interface CatalogClient {

    @GetExchange("/products/{id}")
    Mono<Product> fetch(@PathVariable String id);

    @GetExchange("/products")
    Flux<Product> stream(@RequestParam(required = false) String category);
}
```
Backed by `WebClientAdapter`. Backpressure and cancellation propagate end-to-end.

### Authentication header from a request-scoped principal
```java
@Bean
ClientHttpRequestInterceptor tenantHeaderInterceptor() {
    return (request, body, execution) -> {
        SecurityContext ctx = SecurityContextHolder.getContext();
        if (ctx.getAuthentication() instanceof JwtAuthenticationToken jwt) {
            request.getHeaders().set("X-Tenant-Id", jwt.getToken().getClaimAsString("tenant_id"));
        }
        return execution.execute(request, body);
    };
}
```
Apply the interceptor on the `RestClient.Builder`. Every interface method sees the header automatically.

### Status handling — surface `RFC 7807` Problem Details
```java
RestClient client = RestClient.builder()
    .baseUrl("http://orders-service")
    .defaultStatusHandler(
        HttpStatusCode::is4xxClientError,
        (req, resp) -> {
            ProblemDetail problem = new ObjectMapper().readValue(resp.getBody(), ProblemDetail.class);
            throw new OrdersClientException(problem);
        })
    .build();
```

### Test against `MockRestServiceServer`
```java
@Test
void fetchHitsTheRightUrl() {
    RestClient.Builder builder = RestClient.builder();
    MockRestServiceServer server = MockRestServiceServer.bindTo(builder).build();
    InventoryClient client = HttpServiceProxyFactory
        .builderFor(RestClientAdapter.create(builder.baseUrl("http://stub").build()))
        .build()
        .createClient(InventoryClient.class);

    server.expect(requestTo("http://stub/inventory/42"))
          .andExpect(method(HttpMethod.GET))
          .andRespond(withSuccess("""
              {"sku":42,"available":7}
              """, MediaType.APPLICATION_JSON));

    Stock stock = client.fetch(42);

    assertThat(stock.available()).isEqualTo(7);
    server.verify();
}
```

## Best Practices

- **One interface, one collaborator.** Don't merge `OrdersClient` and `InventoryClient` into a `BackendClient`. Service boundaries map to interface boundaries.
- **Return DTOs, not domain entities.** Define a `record` per response type close to the interface. Keeps the contract explicit and decoupled from the producer's persistence model.
- **Methods that don't return data return `void`.** Don't return `ResponseEntity<Void>` for cosmetic reasons; the proxy handles it. If you need status codes, return `ResponseEntity<T>` explicitly.
- **Use `@ImportHttpServices` once you have 3+ interfaces against the same target.** Single interfaces don't justify a group.
- **Pair with `spring-boot-resilience4j`** for retry/circuit-breaker. Annotate methods at the interface call site, not inside the interface. The proxy plays nicely with AOP wrappers.
- **Propagate tracing headers automatically** by enabling Micrometer Tracing instrumentation for `RestClient` / `WebClient`. See `spring-opentelemetry-tracing`.

## Anti-patterns

- Don't put `@RequestMapping` on `@HttpExchange` interfaces. `@RequestMapping` is for Spring MVC controllers (incoming); `@HttpExchange` is for clients (outgoing). Mixing them confuses the framework and breaks generation.
- Don't add OpenFeign (`spring-cloud-starter-openfeign`) "just in case". This skill exists precisely so you don't need it. OpenFeign duplicates the abstraction with its own annotations and lifecycle.
- Don't share a single `RestClient.Builder` bean across services with different target URLs by mutating `baseUrl` per call. Build one `RestClient` per target — they're cheap.
- Don't rely on Jackson defaults silently. Set `defaultMessageConverter` or configure the `ObjectMapper` explicitly when downstream APIs use snake_case or custom date formats.
- Don't catch `RestClientResponseException` and swallow it. Either propagate or translate to a domain-specific exception (`OrdersClientException`) so callers don't depend on Spring types.

## Related Skills

- `spring-cloud-discovery-config` — `@LoadBalanced` infrastructure for `lb://`-style resolution.
- `spring-boot-resilience4j` — Circuit breaker / retry / timeout decorators around interface calls.
- `spring-opentelemetry-tracing` — Automatic span creation for `RestClient` / `WebClient` outbound calls.
- `spring-security-jwt` — Propagating JWTs to downstream services via interceptors.
- `unit-test-wiremock-rest-api` — Integration testing interface clients against a stubbed HTTP server.
