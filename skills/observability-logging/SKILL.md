---
name: observability-logging
description: "Production logging for Spring Boot 4: Logback config, structured JSON logs, MDC for correlation IDs, async appenders, log levels per profile. Read before designing log output, adding correlation context, or wiring logs to OpenTelemetry. Triggers: Logback, logback-spring.xml, SLF4J, MDC, MDC.put, correlation ID, traceId, spanId, async appender, AsyncAppender, JsonLayout, LogstashEncoder, logging.pattern, logging.structured.format, OpenTelemetry logs."
version: 0.2.0
license: Apache-2.0
---

# Observability — Logging

**Signals**: `logback-spring.xml`, custom `logging.pattern.*` properties, MDC usage, JSON encoder configuration, attempts to correlate logs across services with a `traceId`/`requestId`.

## Tested With

- Spring Boot 4.x (built-in structured logging with `logging.structured.format`)
- Logback 1.5+
- SLF4J 2.x
- OpenTelemetry Java agent (auto-injects `trace_id` / `span_id` into MDC)

## Do NOT Use This Skill When

- Adding metrics or distributed tracing — covered by `spring-boot-actuator` (Micrometer, observation API)
- Configuring Spring profiles or properties in general → use `core-setup`
- Configuring distributed tracing spans / OTLP exporters → use `spring-opentelemetry-tracing`

## When to Read References

| Situation | Read |
|-----------|------|
| Boot 4 native structured logging (`logging.structured.format=ecs|gelf|logstash`) | `references/logging-configuration.md` |
| Custom `logback-spring.xml` (encoder, appenders, profile-aware config) | `references/logging-configuration.md` |
| MDC propagation across `@Async`, reactive contexts, scheduled tasks | `references/logging-configuration.md` |
| Correlation ID filter (HTTP `X-Request-Id` / W3C traceparent) | `references/logging-configuration.md` |
| Async appender sizing, queue tuning, dropped-event semantics | `references/logging-configuration.md` |
| OpenTelemetry log signal export (OTLP) | `references/logging-configuration.md` |

## Anti-patterns

- Don't `System.out.println` — always use SLF4J (`private static final Logger log = LoggerFactory.getLogger(Foo.class)`).
- Don't log secrets, JWTs, full request bodies, or PII — sanitize at the source.
- Don't construct strings with `+` for log arguments — use SLF4J parameterized form (`log.info("user={} order={}", userId, orderId)`).
- Don't ship logs with synchronous file appenders in high-throughput services — use `AsyncAppender` with bounded queue.
- Don't lose MDC across thread boundaries — use `MDCContext` (Reactor) or `MdcTaskDecorator` for `@Async`.
