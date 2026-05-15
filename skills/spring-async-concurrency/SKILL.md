---
name: spring-async-concurrency
description: "Concurrency patterns for Spring Boot 4 on Java 25: @Async + @EnableAsync, TaskExecutor / ThreadPoolTaskExecutor sizing, AsyncConfigurer customization, CompletableFuture composition (thenApply / thenCompose / allOf / anyOf / exception handling), Java 21+ Virtual Threads (Executors.newVirtualThreadPerTaskExecutor, spring.threads.virtual.enabled, Thread.ofVirtual()), StructuredTaskScope for structured concurrency (Java 25), MDC and SecurityContext propagation across thread boundaries with TaskDecorator. Targets development of concurrent code — testing of @Async is covered by unit-test-scheduled-async. Triggers: @Async, @EnableAsync, AsyncConfigurer, TaskExecutor, ThreadPoolTaskExecutor, SimpleAsyncTaskExecutor, VirtualThreadTaskExecutor, spring.threads.virtual.enabled, Executors.newVirtualThreadPerTaskExecutor, Executors.newFixedThreadPool, CompletableFuture, supplyAsync, thenCompose, allOf, anyOf, exceptionally, handle, Thread.ofVirtual, Thread.startVirtualThread, StructuredTaskScope, StructuredTaskScope.ShutdownOnFailure, StructuredTaskScope.ShutdownOnSuccess, TaskDecorator, ContextSnapshot, MdcTaskDecorator, DelegatingSecurityContextExecutor."
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
version: 0.2.0
license: Apache-2.0
---

# Spring Async & Modern Java Concurrency

Concurrency for Spring Boot 4 services on Java 25. **Default to virtual threads.** Platform-thread pools are only justified for tight, sustained CPU-bound loops where pool isolation matters — for everything else (HTTP, DB, message brokers, fan-out IO) use virtual threads.

## Golden Rule

**Never open a platform thread when a virtual thread will do.** Java 25's virtual threads are cheap, schedulable, and observable. `Executors.newVirtualThreadPerTaskExecutor()` and `spring.threads.virtual.enabled=true` are the defaults — reach for `ThreadPoolTaskExecutor` only when you can justify the loss of elasticity (e.g. an isolated CPU pool with `Runtime.availableProcessors()` workers for crypto, image, or codec work).

## Tested With

- Spring Boot 4.0.x
- Spring Framework 7.x
- Java 25 (LTS) — virtual threads are stable, `StructuredTaskScope` finalized
- Hibernate 7.x (for `@Async` + `@Transactional` interplay notes)

## Do NOT Use This Skill When

- Writing tests for `@Async` / `@Scheduled` methods → use `unit-test-scheduled-async`.
- Building reactive pipelines (`Mono` / `Flux`, schedulers, backpressure) → out of scope. This skill is for **imperative** concurrency on virtual threads; WebFlux is not a target stack.
- Tuning Kafka consumer concurrency → use `spring-kafka-advanced` (different concern: container-level concurrency, not per-method async).
- Tuning Tomcat / Netty worker threads at the server level → that's HTTP server config, not application concurrency. Stay in `application.yml` (e.g., `server.tomcat.threads.max`).

## When to Read References

| Situation | Read |
|-----------|------|
| Pool sizing math: CPU-bound vs IO-bound formulas, queue capacity, rejection policies | `references/pool-sizing.md` |
| Virtual threads deep-dive: pinning, synchronized blocks, ThreadLocal cost, when **not** to use them | `references/virtual-threads.md` |
| `StructuredTaskScope` patterns: `ShutdownOnFailure`, `ShutdownOnSuccess`, custom policies, deadlines | `references/structured-concurrency.md` |
| Context propagation: MDC, `SecurityContext`, request-scoped beans across thread boundaries | `references/context-propagation.md` |
| `CompletableFuture` cookbook: timeouts, recovery, fan-out / fan-in, cancellation | `references/completablefuture-cookbook.md` |

## Quick Reference

**Enable virtual threads globally (default for new services):**
```yaml
spring:
  threads:
    virtual:
      enabled: true
```
That switches Tomcat workers, the `@Async` default executor, `@Scheduled`, and Spring's task executors to virtual threads in one move. Combine with `@EnableAsync` to use `@Async` annotations:
```java
@SpringBootApplication
@EnableAsync
public class Application { }
```

**Explicit virtual-thread executor (when you want a named bean for `@Async("name")`):**
```java
@Bean(name = "ioExecutor")
TaskExecutor virtualThreadExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

**Platform-thread pool — only for sustained CPU-bound work that needs isolation:**
```java
@Bean(name = "cpuExecutor")
TaskExecutor cpuExecutor() {
    int cores = Runtime.getRuntime().availableProcessors();
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(cores);
    exec.setMaxPoolSize(cores);
    exec.setQueueCapacity(50);
    exec.setThreadNamePrefix("cpu-");
    exec.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    exec.initialize();
    return exec;
}
```
Use this only when profiling shows a CPU-bound hotspot (hashing, image processing, codec work) that benefits from a bounded pool. The default answer is virtual threads.

**`@Async` with named executor and `CompletableFuture` return:**
```java
@Service
public class ReportService {

    @Async("ordersExecutor")
    public CompletableFuture<Report> generate(long orderId) {
        Report r = buildReport(orderId);          // blocking work
        return CompletableFuture.completedFuture(r);
    }
}
```
Returning `CompletableFuture<T>` lets callers compose. `void` returns are fire-and-forget.

**`CompletableFuture` composition with timeout & recovery:**
```java
CompletableFuture<Report> future = reportService.generate(42L)
    .orTimeout(5, TimeUnit.SECONDS)
    .exceptionally(ex -> Report.failed(ex.getMessage()));
```

**Fan-out / fan-in:**
```java
List<CompletableFuture<LineItem>> futures = ids.stream()
    .map(id -> itemService.fetchAsync(id))
    .toList();

CompletableFuture<List<LineItem>> all = CompletableFuture
    .allOf(futures.toArray(CompletableFuture[]::new))
    .thenApply(v -> futures.stream().map(CompletableFuture::join).toList());
```

**`StructuredTaskScope` (Java 25) — cancel siblings on failure:**
```java
public Report buildReport(long orderId) throws InterruptedException {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        StructuredTaskScope.Subtask<Order>   order   = scope.fork(() -> orders.find(orderId));
        StructuredTaskScope.Subtask<Customer> cust   = scope.fork(() -> customers.find(orderId));
        StructuredTaskScope.Subtask<Stock>    stock  = scope.fork(() -> inventory.fetch(orderId));

        scope.join().throwIfFailed();             // all three or fail fast
        return new Report(order.get(), cust.get(), stock.get());
    }
}
```
Use a virtual-thread `ThreadFactory` (the default for `StructuredTaskScope`) — the three forks become three virtual threads, scoped to this method's stack frame.

**Propagate MDC across `@Async`:**
```java
@Bean
TaskDecorator mdcTaskDecorator() {
    return runnable -> {
        Map<String, String> mdc = MDC.getCopyOfContextMap();
        return () -> {
            Map<String, String> previous = MDC.getCopyOfContextMap();
            if (mdc != null) MDC.setContextMap(mdc); else MDC.clear();
            try { runnable.run(); }
            finally {
                if (previous != null) MDC.setContextMap(previous);
                else MDC.clear();
            }
        };
    };
}

@Bean(name = "ordersExecutor")
TaskExecutor ordersExecutor(TaskDecorator decorator) {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setTaskDecorator(decorator);
    // ...rest of config
    exec.initialize();
    return exec;
}
```

## Instructions

1. **Default to virtual threads.** Enable `spring.threads.virtual.enabled=true` for new services. For explicit `@Async` executors, use `Executors.newVirtualThreadPerTaskExecutor()` wrapped in `TaskExecutorAdapter`. Reach for a `ThreadPoolTaskExecutor` only when a profiler shows sustained CPU contention that benefits from bounded parallelism.
2. **Audit `synchronized` and `ThreadLocal` before enabling virtual threads.** Virtual threads pin to their carrier inside `synchronized` blocks — long sections (file IO, network calls) erase the benefit. Replace with `ReentrantLock`. `ThreadLocal` still works but loses some of the lightweight-thread advantage.
3. **Never use the default `SimpleAsyncTaskExecutor` in production.** With virtual threads enabled it's fine for `@Async`; without them, it creates an unbounded supply of platform threads and can OOM the service.
4. **Always name your `@Async` executors** (`@Async("ioExecutor")`) and bean names (`name = "ioExecutor"`). Spring picks executors by name; silent name collisions are surprising in production.
5. **If you do create a `ThreadPoolTaskExecutor`, set `setRejectedExecutionHandler` explicitly.** `AbortPolicy` (default) throws; `CallerRunsPolicy` gives natural backpressure for HTTP-driven workloads. Pick deliberately.
6. **Use `StructuredTaskScope` for fan-out within a single request.** It binds child tasks' lifetime to the method's stack frame and uses virtual threads by default — cancellation, exception propagation, and resource cleanup all become local concerns.
7. **Propagate MDC and `SecurityContext` deliberately.** Async tasks don't inherit them automatically. Use `TaskDecorator` (Spring) or `ContextSnapshot` (Micrometer Context Propagation) — pick one approach per service to avoid double wrapping.
8. **Mind `@Async` + `@Transactional`.** A `@Transactional` method called from `@Async` starts a **new** transaction; the caller's transaction has nothing to do with the async work. Don't expect a single atomic boundary.
9. **Test concurrency with `Awaitility`** rather than `Thread.sleep`. See `unit-test-scheduled-async` for the testing recipes.

## Examples

### Two executors — virtual default + isolated CPU pool
```java
@Bean(name = "ioExecutor")
TaskExecutor ioExecutor() {
    // Default for everything IO-shaped: HTTP, DB, message brokers, file IO
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}

@Bean(name = "cpuExecutor")
TaskExecutor cpuExecutor() {
    // Reserved for sustained CPU work (hashing, image, codec). Profile first.
    int cores = Runtime.getRuntime().availableProcessors();
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(cores);
    exec.setMaxPoolSize(cores);
    exec.setQueueCapacity(50);
    exec.setThreadNamePrefix("cpu-");
    exec.initialize();
    return exec;
}
```
`@Async("ioExecutor")` for external calls (the common case), `@Async("cpuExecutor")` only for documented CPU hotspots.

### Pipeline composition
```java
CompletableFuture<Order> order = orderClient.fetchAsync(id);
CompletableFuture<Customer> customer = order.thenCompose(o -> customerClient.fetchAsync(o.customerId()));
CompletableFuture<Bill> bill = customer.thenCombine(order, (c, o) -> new Bill(c, o));

bill.thenAccept(b -> notifier.send(b))
    .exceptionally(ex -> { log.error("Bill failed", ex); return null; });
```

### Deadline-aware structured scope
```java
public Snapshot snapshot() throws InterruptedException, TimeoutException {
    Instant deadline = Instant.now().plus(Duration.ofMillis(750));
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        var a = scope.fork(() -> svcA.read());
        var b = scope.fork(() -> svcB.read());
        scope.joinUntil(deadline);
        scope.throwIfFailed();
        return new Snapshot(a.get(), b.get());
    }
}
```
Past the deadline, `joinUntil` throws `TimeoutException` and the scope shuts down both subtasks.

### Java 21+ ergonomics — `Thread.ofVirtual()`
```java
Thread t = Thread.ofVirtual()
    .name("retry-worker-")
    .start(() -> retryLoop());
```
Useful for the occasional one-off; for collections of work use an executor.

## Best Practices

- **Virtual threads first.** Enable `spring.threads.virtual.enabled=true`. Only introduce a `ThreadPoolTaskExecutor` after profiling proves a CPU-bound hotspot needs bounded parallelism.
- **Name threads with a meaningful prefix.** `orders-async-3` in a log line is gold; `pool-2-thread-1` is noise. Virtual threads support naming via `Thread.ofVirtual().name(...)` and `VirtualThreadTaskExecutor("prefix-")`.
- **Prefer `StructuredTaskScope` to manual `CompletableFuture.allOf` when forks share a lifetime.** Structured concurrency catches "leaked" tasks at runtime; `allOf` does not. The scope's default factory is virtual threads.
- **Always set timeouts on `CompletableFuture` chains** with `orTimeout` / `completeOnTimeout`. Long-tail latencies on external services will pile up otherwise.
- **Avoid leaking `Future` references past method boundaries.** A `Future` returned to a caller is a hidden subscription — easy to forget to consume. Either `join()` synchronously or compose further before returning.

## Anti-patterns

- Don't open a platform thread when a virtual thread will do. `new Thread(...)`, `Executors.newFixedThreadPool`, `Executors.newCachedThreadPool` should not appear in new code. Use `Thread.ofVirtual()` or `Executors.newVirtualThreadPerTaskExecutor()`.
- Don't call `@Async` methods from within the same class — Spring's AOP proxy is bypassed and the call runs synchronously. Inject the bean or split the methods.
- Don't enable `spring.threads.virtual.enabled=true` without auditing locks. Heavy `synchronized` use will pin virtual threads to carriers and erase the benefit. Replace with `ReentrantLock`.
- Don't `Thread.sleep` on a virtual thread inside a `synchronized` block. The carrier thread is pinned for the duration. Use `LockSupport.parkNanos` or restructure.
- Don't share a `ThreadPoolTaskExecutor` across unrelated concerns when you do need one (e.g., HTTP outbound + background batch + email). One slow concern starves the others. Define one pool per workload.
- Don't ignore `RejectedExecutionException`. It's the JVM telling you a bounded pool is saturated — log it, alert on it, and reconsider whether a virtual-thread executor fits better.
- Don't rely on `ThreadLocal` for request-scoped state in async paths. The `TaskDecorator` boundary is where context lives or dies.

## Related Skills

- `unit-test-scheduled-async` — Testing `@Async` and `@Scheduled` methods (sister skill).
- `observability-logging` — MDC propagation patterns; the `TaskDecorator` example above feeds it.
- `spring-opentelemetry-tracing` — Tracing context propagates the same way MDC does; Micrometer's `ContextSnapshot` covers both.
- `spring-boot-resilience4j` — Bulkhead patterns (thread-pool isolation) complement explicit `TaskExecutor` definitions.
- `spring-http-interface-clients` — IO-bound consumers that benefit from virtual-thread executors.
