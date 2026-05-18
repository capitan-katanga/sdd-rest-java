---
name: spring-http-interface-clients
description: "Declarative HTTP clients with Spring Framework 7 HTTP Interface for Spring Boot 4.x services: define service-to-service contracts as @HttpExchange-annotated interfaces, generate proxies with HttpServiceProxyFactory + RestClientAdapter, configure the underlying RestClient with HttpComponentsClientHttpRequestFactory (Apache HttpClient 5) for connection pooling and explicit timeouts, propagate headers, handle errors, and unit-test the interfaces. Replaces OpenFeign on Boot 4. Triggers: @HttpExchange, @GetExchange, @PostExchange, @PutExchange, @DeleteExchange, @PatchExchange, HttpServiceProxyFactory, RestClientAdapter, RestClientHttpServiceGroupConfigurer, @ImportHttpServices, HttpServiceGroup, HttpComponentsClientHttpRequestFactory, PoolingHttpClientConnectionManager, CloseableHttpClient, RequestConfig, connectTimeout, socketTimeout, connectionRequestTimeout, ConnectionConfig, SocketConfig, @RequestParam, @PathVariable, @RequestHeader, @RequestBody, @CookieValue."
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
version: 0.2.0
license: Apache-2.0
---

# Spring HTTP Interface Clients (@HttpExchange)

Declarative, **blocking** HTTP clients native to Spring Framework 7 / Boot 4. You write a Java interface annotated with `@HttpExchange`, Spring generates a proxy backed by `RestClient`. **This is the supported replacement for OpenFeign on Boot 4** — no extra dependency beyond `spring-web` and `httpclient5`.

This skill targets **synchronous, virtual-thread-friendly** clients. WebFlux / `WebClient` / `Mono` / `Flux` are out of scope: the project runs on virtual threads, where blocking IO is cheap and `RestClient` is the right primitive.

## Tested With

- Spring Boot 4.0.x
- Spring Framework 7.x
- Java 25 (virtual threads enabled — see `spring-async-concurrency`)
- Apache HttpClient 5 (`httpclient5`) as the underlying transport
- `RestClient` with `HttpComponentsClientHttpRequestFactory`

## Do NOT Use This Skill When

- The client is one-off and you'd rather inline the call → use `RestClient` directly (no need for a declarative interface).
- The service-to-service call goes through an external API with hand-rolled auth flows / signing → use `RestClient` directly so you have full control over the request builder.
- Migrating an existing OpenFeign codebase → that's a refactor topic; the skill `spring-cloud-openfeign` does **not exist** in this plugin by design. Treat OpenFeign code as legacy to port to `@HttpExchange`.
- Reactive (`WebClient` / `Mono` / `Flux`) is needed → out of scope; this project does not target WebFlux.

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

**Wire the proxy (blocking — `RestClient` over Apache HttpClient 5):**
```java
@Configuration
public class InventoryClientConfig {

    @Bean
    InventoryClient inventoryClient(RestClient.Builder builder,
                                    ClientHttpRequestFactory requestFactory) {
        RestClient client = builder
            .baseUrl("http://inventory-service.internal")
            .requestFactory(requestFactory)
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

## Apache HttpClient 5 — Connection Pool & Timeouts

`RestClient`'s default factory is `JdkClientHttpRequestFactory`. **In this project use `HttpComponentsClientHttpRequestFactory` (Apache HttpClient 5)** because it exposes an explicit `PoolingHttpClientConnectionManager` and `RequestConfig` — both of which are needed to bound resources and surface failures fast.

Dependency:
```xml
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
</dependency>
```

**Pooled, timeout-aware request factory:**
```java
@Configuration
public class HttpClientConfig {

    @Bean(destroyMethod = "close")
    PoolingHttpClientConnectionManager connectionManager() {
        return PoolingHttpClientConnectionManagerBuilder.create()
            .setMaxConnTotal(200)                         // hard cap across all routes
            .setMaxConnPerRoute(50)                       // per target host:port
            .setDefaultConnectionConfig(ConnectionConfig.custom()
                .setConnectTimeout(Timeout.ofSeconds(2)) // TCP handshake
                .setSocketTimeout(Timeout.ofSeconds(5))  // inactivity on a socket
                .setTimeToLive(TimeValue.ofMinutes(10))  // total lifetime of a pooled conn
                .build())
            .setDefaultSocketConfig(SocketConfig.custom()
                .setSoKeepAlive(true)
                .build())
            .build();
    }

    @Bean(destroyMethod = "close")
    CloseableHttpClient httpClient(PoolingHttpClientConnectionManager pool) {
        return HttpClients.custom()
            .setConnectionManager(pool)
            .evictIdleConnections(TimeValue.ofMinutes(1))
            .evictExpiredConnections()
            .setDefaultRequestConfig(RequestConfig.custom()
                .setConnectionRequestTimeout(Timeout.ofSeconds(1))   // wait for a pool slot
                .setResponseTimeout(Timeout.ofSeconds(5))            // server response budget
                .build())
            .setRetryStrategy(new DefaultHttpRequestRetryStrategy(0, TimeValue.ZERO))
            // retries belong to Resilience4j at the call site, not the transport layer
            .build();
    }

    @Bean
    ClientHttpRequestFactory clientHttpRequestFactory(CloseableHttpClient httpClient) {
        return new HttpComponentsClientHttpRequestFactory(httpClient);
    }
}
```

Pin **every** timeout — Apache's defaults are "wait forever":

| Timeout | Where | Meaning |
|---|---|---|
| `connectTimeout` | `ConnectionConfig` | TCP handshake budget. 1–3 s. |
| `connectionRequestTimeout` | `RequestConfig` | How long to wait for a slot from the pool when it's saturated. Keep it small (≤ 1 s) so saturation fails fast. |
| `responseTimeout` | `RequestConfig` | Total time to read a response. Sized to the downstream SLO. |
| `socketTimeout` | `ConnectionConfig` | Inactivity between packets on an established socket. Usually equal to `responseTimeout`. |
| `timeToLive` | `ConnectionConfig` | Hard ceiling for a pooled connection's lifetime. Defends against load-balancer staleness and DNS changes. |

Pool sizing rules of thumb:
- `maxConnPerRoute` ≈ steady-state concurrency you expect to that host. Too low → contention on `connectionRequestTimeout`. Too high → wasted descriptors and risk of overwhelming the downstream.
- `maxConnTotal` ≥ sum of per-route caps with headroom. The cap also bounds your blast radius if one downstream goes slow.

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
      client-type: rest_client          # this project uses RestClient only
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
4. **Always back `RestClient` with `HttpComponentsClientHttpRequestFactory`.** The JDK factory has no per-request configurability and no shared pool. Use Apache HttpClient 5 with a `PoolingHttpClientConnectionManager` defined as a singleton bean.
5. **Pin every timeout** (`connectTimeout`, `connectionRequestTimeout`, `responseTimeout`, `socketTimeout`, `timeToLive`). Apache's defaults are "wait forever" — a slow downstream will hang virtual threads indefinitely otherwise.
6. **Add interceptors for cross-cutting concerns** (auth header, correlation ID, tracing). Don't bake them into every interface method.
7. **Handle errors with `RestClient.defaultStatusHandler`**. The proxy propagates the underlying client's exception — parse Problem Details (`RFC 7807`) where applicable and translate to a domain exception.
8. **Unit-test the contract** with `MockRestServiceServer`. The proxy is a real client invocation, so tests must mock the HTTP layer.
9. **Delegate retries to Resilience4j**, not the HttpClient retry strategy. Transport-level retries are blind to idempotency and semantics; call-site retries can be selective. See `spring-boot-resilience4j`.

## Examples

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
- **Propagate tracing headers automatically** by enabling Micrometer Tracing instrumentation on the `RestClient` builder. See `spring-opentelemetry-tracing`.
- **Reuse one `PoolingHttpClientConnectionManager` across all clients to the same backend tier.** Multiple managers fragment the pool and double the descriptor count for no benefit.
- **Expose pool stats** via Micrometer (`PoolingHttpClientConnectionManager#getTotalStats`) — saturation should page someone before it propagates as user-facing timeouts.

## Anti-patterns

- Don't put `@RequestMapping` on `@HttpExchange` interfaces. `@RequestMapping` is for Spring MVC controllers (incoming); `@HttpExchange` is for clients (outgoing). Mixing them confuses the framework and breaks generation.
- Don't add OpenFeign (`spring-cloud-starter-openfeign`) "just in case". This skill exists precisely so you don't need it. OpenFeign duplicates the abstraction with its own annotations and lifecycle.
- Don't share a single `RestClient.Builder` bean across services with different target URLs by mutating `baseUrl` per call. Build one `RestClient` per target — they're cheap.
- Don't rely on Jackson defaults silently. Set `defaultMessageConverter` or configure the `ObjectMapper` explicitly when downstream APIs use snake_case or custom date formats.
- Don't catch `RestClientResponseException` and swallow it. Either propagate or translate to a domain-specific exception (`OrdersClientException`) so callers don't depend on Spring types.

## Related Skills

- `spring-boot-resilience4j` — Circuit breaker / retry / timeout decorators at the interface call site (preferred over Apache's transport-level retries).
- `spring-opentelemetry-tracing` — Automatic span creation for `RestClient` outbound calls.
- `spring-security-jwt` — Propagating JWTs to downstream services via interceptors.
- `spring-async-concurrency` — Virtual threads make this blocking client design viable; ensure `spring.threads.virtual.enabled: true`.
- `unit-test-wiremock-rest-api` — Integration testing interface clients against a stubbed HTTP server.
