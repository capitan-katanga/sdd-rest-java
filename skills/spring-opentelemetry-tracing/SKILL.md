---
name: spring-opentelemetry-tracing
description: "Distributed tracing for Spring Boot 4 with Micrometer Tracing + OpenTelemetry bridge: OTLP exporter, automatic spans for HTTP server/client, JDBC, Kafka producers/consumers, RestClient/WebClient and HTTP Interface clients; manual spans with Tracer / @Observed / ObservationRegistry; W3C Trace Context propagation across services; baggage; sampling strategies (parent-based, ratio, always-on); traceId/spanId in MDC for log correlation; exporter targets (Jaeger, Tempo, Grafana Cloud, Honeycomb). Triggers: micrometer-tracing-bridge-otel, opentelemetry-exporter-otlp, management.otlp.tracing.endpoint, management.tracing.sampling.probability, management.tracing.propagation.type, Tracer, Span, ObservationRegistry, @Observed, Observation.createNotStarted, ContextSnapshot, Baggage, BaggageInScope, W3CTraceContextPropagator, traceparent, tracestate, traceId, spanId, OtlpHttpSpanExporter, OtlpGrpcSpanExporter, otel.javaagent."
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
version: 0.1.0
license: Apache-2.0
---

# Distributed Tracing — OpenTelemetry for Spring Boot 4

End-to-end distributed tracing for Spring Boot 4 microservices using the Micrometer Tracing API with the OpenTelemetry bridge. Automatic spans for HTTP/JDBC/Kafka, manual spans via `Tracer` / `@Observed`, and OTLP export to any compatible backend.

## Tested With

- Spring Boot 4.0.x
- Spring Framework 7.x
- Micrometer Tracing 1.3+ with the **OpenTelemetry** bridge (`micrometer-tracing-bridge-otel`)
- OpenTelemetry Java 1.40+ (exporter + SDK)
- Java 25

## Do NOT Use This Skill When

- Looking for metrics / Prometheus / Actuator health → use `spring-boot-actuator` (different telemetry signal). Tracing and metrics share Micrometer but are configured separately.
- Configuring structured logs / MDC / correlation IDs → use `observability-logging`. This skill **feeds** that one (`traceId`/`spanId` land in MDC automatically once tracing is enabled).
- Routing telemetry from a managed cloud agent → out of scope. This skill covers SDK-level export. Agent / sidecar / collector deployment is infra.
- Picking a backend → out of scope. Pattern is the same whether you ship to Jaeger, Tempo, Honeycomb, or vendor X.

## When to Read References

| Situation | Read |
|-----------|------|
| Sampling strategies: parent-based, ratio, head vs tail sampling, per-route overrides | `references/sampling.md` |
| OTLP exporter knobs: HTTP vs gRPC, headers, compression, retries, batch size | `references/otlp-exporter.md` |
| Manual instrumentation cookbook: `Tracer`, `Span`, `@Observed`, `ObservationRegistry`, low/high cardinality keys | `references/manual-instrumentation.md` |
| Baggage: propagating tenant/user context across services, security considerations | `references/baggage.md` |
| Backend-specific config (Jaeger, Tempo, Grafana Cloud, Honeycomb) | `references/backend-targets.md` |
| Pure OpenTelemetry SDK without Micrometer bridge (when, why, trade-offs) | `references/otel-sdk-direct.md` |

## Quick Reference

**Dependencies (Micrometer + OTel bridge — recommended default):**
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

**Minimum config:**
```yaml
spring.application.name: orders-service

management:
  tracing:
    enabled: true
    sampling.probability: 1.0              # dev: 100% — drop in prod (see Sampling)
    propagation.type: w3c                  # default; also supports b3, b3_multi
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
      transport: http                      # or grpc → port 4317
      compression: gzip
      timeout: 10s
      headers:
        x-honeycomb-team: ${HONEYCOMB_API_KEY:}    # backend-specific auth
```

**Add `traceId`/`spanId` to logs (Logback / Boot 4 default pattern):**

Boot 4 includes traceId / spanId in the default console log pattern when tracing is on the classpath. To customize:
```yaml
logging.pattern.level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```
Pair with `observability-logging` for JSON / structured output.

**Manual span with `Tracer`:**
```java
@Service
public class PricingService {

    private final Tracer tracer;

    public Quote price(Order order) {
        Span span = tracer.nextSpan().name("pricing.calculate").start();
        try (Tracer.SpanInScope scope = tracer.withSpan(span)) {
            span.tag("order.id", order.id());
            span.tag("order.items", String.valueOf(order.items().size()));
            return doCalculate(order);
        } catch (Exception ex) {
            span.error(ex);
            throw ex;
        } finally {
            span.end();
        }
    }
}
```

**`@Observed` — the lighter-weight option:**
```java
@Observed(
    name = "pricing.calculate",
    contextualName = "calculate-price",
    lowCardinalityKeyValues = { "service.tier", "premium" }
)
public Quote price(Order order) { /* ... */ }
```
Requires `ObservationRegistry` on the classpath (pulled in by `actuator` + tracing bridge). `@Observed` creates the span and a metric simultaneously — useful when you want both signals from one annotation.

**Baggage (propagate custom data with the trace):**
```yaml
management.tracing:
  baggage:
    remote-fields: x-tenant-id, x-user-id
    correlation:
      fields: x-tenant-id, x-user-id        # also land in MDC for logs
```
```java
try (BaggageInScope b = tracer.createBaggageInScope("x-tenant-id", "acme-co")) {
    inventoryClient.fetch(sku);   // header propagates downstream
}
```

## Instructions

1. **Stick with the Micrometer + OpenTelemetry bridge as the default.** Spring Boot's `ObservationRegistry` integrates with `RestClient`, `WebClient`, `RestTemplate`, `KafkaTemplate`, `@KafkaListener`, JDBC, scheduled tasks, and HTTP server endpoints out of the box. The pure OTel SDK works too, but you lose the unified Observation API. Use the bridge unless you have a reason not to (see `references/otel-sdk-direct.md`).
2. **Set `spring.application.name` everywhere.** It becomes the `service.name` attribute on every span — the primary axis backends pivot on.
3. **Use OTLP, not Jaeger's native protocol.** OTLP is the OpenTelemetry standard; every modern backend speaks it. Don't pull in `opentelemetry-exporter-jaeger` unless targeting a very old Jaeger.
4. **Pick the propagation format intentionally.** `w3c` (default) is the cross-vendor standard. `b3`/`b3_multi` is needed only when integrating with legacy Zipkin-style services that don't speak W3C.
5. **Sampling is per-service but propagates head-decisions.** A request sampled at the gateway carries `sampled=1` in `traceparent`, so downstream services keep it — they only make a sampling decision when no parent span exists. Tune `sampling.probability` from the edge.
6. **Don't manually instrument what's already automatic.** HTTP server/client, JDBC, Kafka, scheduled methods, and `RestClient`/`WebClient` are auto-instrumented. Add manual spans only for business steps (e.g., "pricing.calculate", "fraud.check") that aren't HTTP/DB calls.
7. **Wire `traceId`/`spanId` into your logs.** Boot 4 does this by default with tracing on the classpath; verify the pattern in `application.yml`. For JSON logs, ensure the encoder includes MDC fields — see `observability-logging`.
8. **Use baggage sparingly.** Baggage is transmitted with **every** propagated request — keep it tiny. Sensitive data (auth tokens, PII) does **not** belong in baggage; it travels across every hop.
9. **Run an OpenTelemetry Collector** in dev and prod. Direct exporter-to-backend works but couples your service to the backend's protocol and credentials. The Collector buffers, batches, and lets you switch backends without redeploying services.

## Examples

### Service-to-service via `@HttpExchange` (W3C propagation, no manual code)
```yaml
# Both services use the same config:
management.tracing.propagation.type: w3c
management.otlp.tracing.endpoint: http://otel-collector:4318/v1/traces
```
A `RestClient` built through Boot's auto-configured `RestClient.Builder` carries `traceparent` on outbound calls. The downstream service's auto-instrumentation reads it, makes its own span a child, and the entire trace shows up end-to-end in your backend. No code required.

### Kafka — trace continues across topics
With `micrometer-tracing-bridge-otel` and `spring-kafka` on the classpath, `KafkaTemplate.send` injects `traceparent` into record headers and `@KafkaListener` reads it back. The span graph spans producer → broker → consumer naturally.

### Custom `Observation` with key-values
```java
public Outcome process(Payment p) {
    return Observation.createNotStarted("payment.process", observationRegistry)
        .lowCardinalityKeyValue("payment.gateway", p.gateway())
        .highCardinalityKeyValue("payment.id", p.id().toString())
        .observe(() -> doProcess(p));
}
```
Low-cardinality keys end up as span attributes **and** metric tags (so use them for things with bounded values, like `payment.gateway`). High-cardinality go on the span only.

### Async + tracing context propagation
`Observation` automatically captures the snapshot when you wrap a `Runnable`/`Callable`:
```java
Runnable wrapped = observationRegistry.observationConfig().getObservationConvention()
    .toString();   // illustrative — real usage:

var snapshot = ContextSnapshotFactory.builder().build().captureAll();
executor.submit(snapshot.wrap(() -> doWork()));
```
For Spring's `@Async` executor, configure a `TaskDecorator` that calls `ContextSnapshot.wrap`. See `spring-async-concurrency` for the pattern.

## Best Practices

- **One Collector per environment**, not one per service. Services point at `otel-collector:4318`; the Collector handles routing, retries, and credential management.
- **Sample aggressively in prod**, generously in dev. `1.0` (100%) is fine locally; in prod `0.1`–`0.05` (5–10%) is typical for high-volume services. Adjust per `service.name` if needed.
- **Drop noisy spans at the Collector**, not in code. Add filter/processor stages in the Collector config to discard `/actuator/health`, static asset traces, etc. Keeps your application code clean.
- **Use `@Observed` for business operations**, manual `Tracer` only when you need fine-grained control (e.g., span attributes computed from a partial result).
- **Always end spans in `finally`.** Forgotten `span.end()` leaks spans and corrupts trace duration. Prefer try-with-resources / `Observation.observe(Supplier)` patterns.
- **Tag with stable, low-cardinality keys.** Span attributes like `user.id` blow up the tag cardinality and break aggregation in metrics backends. Put user IDs on `Span` only, not on `Observation` low-cardinality keys.

## Anti-patterns

- Don't ship to two backends directly from the SDK. Either dual-export through the Collector or pick one backend. The SDK can only have one OTLP exporter per signal.
- Don't put auth tokens, JWTs, or PII in baggage or span attributes. Span data is often retained for weeks at the backend; treat it as low-trust storage.
- Don't disable the Boot-default log pattern when you turn tracing on. The `[%X{traceId},%X{spanId}]` correlation in logs is half the value of tracing.
- Don't manually create spans inside an `@Observed` method. You'll get nested spans with awkward names and double-billing on metrics.
- Don't use `sampling.probability: 0` in dev — you'll never see traces while debugging. Use `1.0` locally, lower in prod.
- Don't pull in the OTel Java agent (`-javaagent:opentelemetry-javaagent.jar`) **and** the Micrometer bridge in the same JVM. They both instrument the same libraries; you get double spans and broken propagation. Pick the bridge **or** the agent.

## Related Skills

- `observability-logging` — `traceId`/`spanId` MDC integration; structured JSON logs that include trace context.
- `spring-boot-actuator` — Metrics side of Micrometer; same `ObservationRegistry`, different signal.
- `spring-async-concurrency` — `ContextSnapshot` / `TaskDecorator` for propagating trace context across thread boundaries.
- `spring-http-interface-clients` — Outbound HTTP calls are auto-instrumented when `RestClient` is built through Boot's auto-configured builder.
- `spring-kafka-advanced` — Trace propagation across Kafka topic hops.
