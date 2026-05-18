---
name: spring-kafka-advanced
description: "Advanced Spring for Apache Kafka patterns on Boot 4: Schema Registry (Confluent / Apicurio) with Avro or Protobuf, exactly-once semantics via KafkaTransactionManager and idempotent producer, partition strategy and custom Partitioner, batch listeners, manual offset management, DefaultErrorHandler + DeadLetterPublishingRecoverer for DLT, ConcurrentKafkaListenerContainerFactory tuning, Kafka Streams (KStream/KTable/topology), and integration testing with EmbeddedKafkaBroker or Testcontainers KafkaContainer. Targets spring-kafka 3.3+. Triggers: KafkaTemplate, @KafkaListener, ConcurrentKafkaListenerContainerFactory, KafkaTransactionManager, transactional.id, enable.idempotence, acks=all, DefaultErrorHandler, DeadLetterPublishingRecoverer, FixedBackOff, ExponentialBackOff, RetryTopicConfiguration, @RetryableTopic, SeekToCurrentErrorHandler, RecoveringBatchErrorHandler, KafkaAvroSerializer, KafkaProtobufSerializer, schema.registry.url, KStream, KTable, StreamsBuilder, Topology, processing.guarantee=exactly_once_v2, EmbeddedKafkaBroker, KafkaContainer."
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
version: 0.1.0
license: Apache-2.0
---

# Spring Kafka — Advanced Patterns

Production patterns for Spring for Apache Kafka on Boot 4: schemas, exactly-once delivery, partitioning, DLTs, listener tuning, Streams, and testing.

## Tested With

- Spring Boot 4.0.x
- Spring for Apache Kafka 3.3+
- Apache Kafka 3.7+ (broker)
- Confluent Schema Registry 7.x **or** Apicurio Registry 3.x
- Java 25
- Testcontainers `kafka` module (preferred for integration tests)

## Do NOT Use This Skill When

- Sagas spanning multiple services (compensating actions, orchestration vs choreography) — out of scope for the current plugin cut; treat as application-level orchestration.
- Inbound HTTP routing at the edge — out of scope. Kafka is not an HTTP gateway.

## When to Read References

| Situation | Read |
|-----------|------|
| Schema Registry: Avro vs Protobuf, schema evolution (backward / forward / full), serializer config | `references/schema-registry.md` |
| Exactly-once: `KafkaTransactionManager`, chained transactions with `@Transactional`, read-process-write loops | `references/exactly-once.md` |
| Partitioning: key strategy, custom `Partitioner`, sticky partitioner, ordering guarantees | `references/partitioning.md` |
| Error handling: `DefaultErrorHandler`, backoff strategies, `DeadLetterPublishingRecoverer`, non-blocking retries with `@RetryableTopic` | `references/error-handling-and-dlt.md` |
| Listener tuning: concurrency, `max.poll.records`, `fetch.min.bytes`, batch listeners, `RecoveringBatchErrorHandler` | `references/listener-tuning.md` |
| Kafka Streams: `StreamsBuilder` config, stateful joins, windowing, exactly-once_v2, `KafkaStreamsCustomizer` | `references/kafka-streams.md` |
| Testing: `EmbeddedKafkaBroker` vs Testcontainers `KafkaContainer` trade-offs | `references/testing.md` |

## Quick Reference

**Dependencies:**
```xml
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>
<!-- Schema Registry — Avro (Confluent) -->
<dependency>
  <groupId>io.confluent</groupId>
  <artifactId>kafka-avro-serializer</artifactId>
  <version>7.6.0</version>
</dependency>
<!-- Tests -->
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>kafka</artifactId>
  <scope>test</scope>
</dependency>
```

**Idempotent + transactional producer:**
```yaml
spring.kafka:
  producer:
    acks: all
    properties:
      enable.idempotence: true
      max.in.flight.requests.per.connection: 5
    transaction-id-prefix: orders-tx-
  consumer:
    isolation-level: read_committed
    properties:
      enable.auto.commit: false
```

**Avro producer/consumer config:**
```yaml
spring.kafka:
  properties:
    schema.registry.url: http://schema-registry:8081
  producer:
    value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
  consumer:
    value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
    properties:
      specific.avro.reader: true
```

**Exactly-once read-process-write:**
```java
@Bean
KafkaTransactionManager<String, OrderEvent> kafkaTm(
        ProducerFactory<String, OrderEvent> pf) {
    return new KafkaTransactionManager<>(pf);
}

@KafkaListener(topics = "orders.in", containerFactory = "txContainerFactory")
@Transactional("kafkaTm")
public void onOrder(ConsumerRecord<String, OrderEvent> record) {
    OrderEvent enriched = enrich(record.value());
    template.send("orders.out", record.key(), enriched);
    // commit is atomic across consumer offset + producer send
}
```

**Dead-Letter Topic with exponential backoff:**
```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<String, ?> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
        template,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

    ExponentialBackOffWithMaxRetries backoff = new ExponentialBackOffWithMaxRetries(5);
    backoff.setInitialInterval(500);
    backoff.setMultiplier(2.0);
    backoff.setMaxInterval(10_000);

    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backoff);
    handler.addNotRetryableExceptions(IllegalArgumentException.class);
    return handler;
}
```

**Non-blocking retries via `@RetryableTopic`:**
```java
@RetryableTopic(
    attempts = "5",
    backoff = @Backoff(delay = 500, multiplier = 2.0, maxDelay = 10_000),
    autoCreateTopics = "true",
    dltStrategy = DltStrategy.FAIL_ON_ERROR
)
@KafkaListener(topics = "payments.events")
public void onPayment(PaymentEvent evt) { /* ... */ }
```
This generates `payments.events-retry-0`, `…-retry-1`, …, `payments.events-dlt` automatically.

**Custom partitioner (key by tenant):**
```java
public class TenantPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        TenantKey tk = (TenantKey) key;
        int total = cluster.partitionCountForTopic(topic);
        return Math.floorMod(tk.tenantId().hashCode(), total);
    }
    @Override public void close() {}
    @Override public void configure(Map<String, ?> configs) {}
}
```
```yaml
spring.kafka.producer.properties.partitioner.class: com.example.TenantPartitioner
```

## Instructions

1. **Decide the schema serialization format first.** Avro is mature with strong tooling around the Confluent registry; Protobuf is preferred when you want shared schemas across non-JVM consumers. JSON Schema is supported but loses much of the typing benefit. Switching later forces a topic re-key — pick once.
2. **Enable idempotent producer (`enable.idempotence=true`) on every producer.** It's free at-most-once-per-partition protection and is required for transactional producers. Boot enables it by default but make it explicit so it survives config refactors.
3. **For exactly-once read-process-write, set `transaction-id-prefix`** and a `KafkaTransactionManager`. Mark the listener method `@Transactional("kafkaTm")` — the offset commit becomes part of the producer transaction.
4. **Always plan for the failure mode.** Default behavior on a `RuntimeException` is to redeliver indefinitely. Configure `DefaultErrorHandler` with a bounded retry and a `DeadLetterPublishingRecoverer` from day one — don't wait for a production incident.
5. **Pick blocking vs non-blocking retries.** Blocking (`DefaultErrorHandler` with backoff) pauses the partition during retries — fine for low-volume, latency-tolerant topics. Non-blocking (`@RetryableTopic`) routes retries to dedicated topics and unblocks the main flow — preferred for high-throughput.
6. **Tune `concurrency`, `max.poll.records`, and `fetch.min.bytes` together.** Increasing concurrency only helps if partitions ≥ concurrency. `max.poll.records` controls batch size per poll; pair with `RecoveringBatchErrorHandler` if using `@KafkaListener(batch=true)`.
7. **Set `auto.offset.reset` consciously.** `earliest` rewinds new consumers to the beginning of the topic; `latest` skips backlog. Default is `latest` — for analytics consumers you usually want `earliest`.
8. **For Kafka Streams use `processing.guarantee=exactly_once_v2`** (requires brokers 2.5+). The older `exactly_once` is deprecated.
9. **Test with Testcontainers `KafkaContainer`** for integration tests that exercise real broker semantics (transactions, compaction). Use `EmbeddedKafkaBroker` only when speed matters more than fidelity (e.g., simple serdes tests).

## Examples

### Batch listener with partial-batch recovery
```java
@KafkaListener(
    topics = "metrics.raw",
    containerFactory = "batchFactory",
    batch = "true"
)
public void onMetricsBatch(List<ConsumerRecord<String, Metric>> batch,
                           Acknowledgment ack) {
    metricsService.persist(batch.stream().map(ConsumerRecord::value).toList());
    ack.acknowledge();
}
```
Pair with:
```java
@Bean
ConcurrentKafkaListenerContainerFactory<String, Metric> batchFactory(
        ConsumerFactory<String, Metric> cf) {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, Metric>();
    factory.setConsumerFactory(cf);
    factory.setBatchListener(true);
    factory.setCommonErrorHandler(new RecoveringBatchErrorHandler(
        (record, ex) -> log.error("Batch poison record {}", record, ex),
        new FixedBackOff(1000L, 3L)));
    return factory;
}
```

### Kafka Streams — windowed aggregation
```java
@Bean
KStream<String, OrderEvent> ordersStream(StreamsBuilder builder) {
    KStream<String, OrderEvent> orders = builder.stream(
        "orders.events",
        Consumed.with(Serdes.String(), orderEventSerde()));

    orders
        .groupBy((k, v) -> v.tenantId())
        .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofMinutes(1)))
        .aggregate(
            OrderStats::empty,
            (key, evt, agg) -> agg.with(evt),
            Materialized.with(Serdes.String(), orderStatsSerde()))
        .toStream()
        .map((wk, stats) -> new KeyValue<>(wk.key(), stats))
        .to("orders.stats", Produced.with(Serdes.String(), orderStatsSerde()));

    return orders;
}

@Bean
KafkaStreamsConfiguration streamsConfig() {
    return new KafkaStreamsConfiguration(Map.of(
        StreamsConfig.APPLICATION_ID_CONFIG, "orders-aggregator",
        StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092",
        StreamsConfig.PROCESSING_GUARANTEE_CONFIG, "exactly_once_v2",
        StreamsConfig.REPLICATION_FACTOR_CONFIG, 3));
}
```

### Integration test with Testcontainers
```java
@SpringBootTest
@Testcontainers
class OrderPipelineTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("apache/kafka:3.7.0"));

    @DynamicPropertySource
    static void registerProps(DynamicPropertyRegistry reg) {
        reg.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Test
    void publishesEnrichedOrder(@Autowired KafkaTemplate<String, OrderEvent> template) {
        template.send("orders.in", "order-1", new OrderEvent("order-1", 100));
        // assert presence on orders.out via test consumer
    }
}
```

## Best Practices

- **Always pin `acks=all` + `enable.idempotence=true`** on producers handling business events. Cheap insurance against duplicates and silent data loss.
- **Use one consumer group per logical consumer**, not per service. Two services that should each see every event need two different group IDs.
- **Treat the DLT as production data.** Build alerting on DLT-write rate and a replay tool (re-publish to the source topic after fixing the root cause). DLT is not "dead", it's "needs human attention".
- **Keep schema evolution backward-compatible by default.** Add optional fields, never remove or rename. Configure Schema Registry compatibility mode to `BACKWARD` (or `BACKWARD_TRANSITIVE` for paranoid setups).
- **Don't share `application.id` across Streams instances of different services.** It's the cluster-wide identity of the topology; collisions are silent and corrupt state stores.
- **For request/response patterns over Kafka, prefer `ReplyingKafkaTemplate`** with correlation IDs. Don't reinvent it on top of plain `KafkaTemplate`.

## Anti-patterns

- Don't catch `Exception` in `@KafkaListener` and swallow it. The default error handler exists precisely so you don't have to. Throwing through the listener triggers retries / DLT routing.
- Don't `Thread.sleep` in a listener to "rate limit". Use `pauseConsumer`/`resumeConsumer` or a separate dedicated consumer with low concurrency. Sleeping blocks the partition.
- Don't rely on Kafka ordering across partitions. Ordering is per-partition only. Either route correlated records to the same partition (by key) or don't depend on order.
- Don't use `seek()` from inside a listener handler — race conditions with the container's polling thread. Use `ConsumerSeekAware` callbacks.
- Don't enable `auto.create.topics.enable=true` in production. Topic creation should be an explicit migration step (Terraform / `KafkaAdmin` bean) with the right partition count and replication factor for the workload.

## Related Skills

- `spring-boot-resilience4j` — Retries / circuit breakers around the **producer** side (e.g., wrapping `KafkaTemplate.send` calls into the broker).
- `spring-opentelemetry-tracing` — Kafka client instrumentation propagates W3C trace headers across topic hops.
- `spring-async-concurrency` — Listener container concurrency vs application-level virtual-thread executors.
- `core-setup` — Externalizing bootstrap servers, schema registry URLs, and credentials via env vars in `application.yml`.
