---
name: spring-data-jpa
description: "Persistence with Spring Data JPA + Hibernate 7 on Spring Boot 4.x: repositories (JpaRepository), entity relationships, derived queries and @Query, pagination, auditing (@CreatedDate/@LastModifiedDate), transactions, UUID PKs, Hibernate second-level cache (JCache / Ehcache / Caffeine, @Cache annotation, query cache), database indexing. Triggers: JpaRepository, @Entity, @Query, @EntityGraph, @Transactional, @Cache, CacheConcurrencyStrategy, hibernate.cache.use_second_level_cache, hibernate.cache.region.factory_class, spring.jpa.properties.hibernate.cache, JCacheRegionFactory, Ehcache, Caffeine, second-level cache, L2 cache, query cache."
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
version: 0.2.0
license: Apache-2.0
---

# Spring Data JPA

## Overview

Provides patterns for Spring Data JPA repositories, entity relationships, queries, pagination, auditing, and transactions.

## When to Use

Creating repositories with CRUD operations, entity relationships, `@Query` annotations, pagination, auditing, or UUID primary keys.

## Instructions

### Create Repository Interfaces

To implement a repository interface:

1. **Extend the appropriate repository interface:**
   ```java
   @Repository
   public interface UserRepository extends JpaRepository<User, Long> {
       // Custom methods defined here
   }
   ```

2. **Use derived queries for simple conditions:**
   ```java
   Optional<User> findByEmail(String email);
   List<User> findByStatusOrderByCreatedDateDesc(String status);
   ```

3. **Implement custom queries with `@`Query:**
   ```java
   @Query("SELECT u FROM User u WHERE u.status = :status")
   List<User> findActiveUsers(@Param("status") String status);
   ```

### Configure Entities

1. **Define entities with proper annotations:**
   ```java
   @Entity
   @Table(name = "users")
   public class User {
       @Id
       @GeneratedValue(strategy = GenerationType.IDENTITY)
       private Long id;

       @Column(nullable = false, length = 100)
       private String email;
   }
   ```

2. **Configure relationships using appropriate cascade types:**
   ```java
   @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
   private List<Order> orders = new ArrayList<>();
   ```
   **Validation:** Test cascade behavior with a small dataset before applying to production data. Verify delete operations don't cascade unexpectedly.

3. **Set up database auditing:**
   ```java
   @CreatedDate
   @Column(nullable = false, updatable = false)
   private LocalDateTime createdDate;
   ```

### Apply Query Patterns

1. **Use derived queries for simple conditions**
2. **Use `@`Query for complex queries**
3. **Return Optional<T> for single results**
4. **Use Pageable for pagination**
5. **Apply `@`Modifying for update/delete operations**

### Manage Transactions

1. **Mark read-only operations with `@`Transactional(readOnly = true)**
2. **Use explicit transaction boundaries for modifying operations**
3. **Specify rollback conditions when needed**

### Enable Hibernate Second-Level Cache (when read traffic warrants it)

The L2 cache lives **across sessions** — it caches entities by primary key for the whole `EntityManagerFactory`, not just one transaction. Use it when the same entities are loaded repeatedly across requests and the underlying data changes rarely. Skip it for write-heavy tables.

This project uses Hibernate's L2 cache as the **only** application-level caching mechanism. There is no `spring-boot-starter-cache` / `@Cacheable` story in this codebase — keep cache concerns inside the persistence layer.

1. **Add JCache + a provider (Ehcache or Caffeine) to `pom.xml`:**
   ```xml
   <dependency>
       <groupId>org.hibernate.orm</groupId>
       <artifactId>hibernate-jcache</artifactId>
   </dependency>
   <dependency>
       <groupId>org.ehcache</groupId>
       <artifactId>ehcache</artifactId>
       <classifier>jakarta</classifier>
   </dependency>
   ```
   For an in-memory single-JVM cache prefer Caffeine via `com.github.ben-manes.caffeine:jcache`. Use Ehcache when you need disk-tiered or off-heap regions.

2. **Wire Hibernate via `application.yml`:**
   ```yaml
   spring:
     jpa:
       properties:
         hibernate:
           cache:
             use_second_level_cache: true
             use_query_cache: false             # opt in per query only
             region:
               factory_class: jcache
           javax:
             cache:
               provider: org.ehcache.jsr107.EhcacheCachingProvider
               missing_cache_strategy: create   # create regions on demand in dev
   ```
   In production set `missing_cache_strategy: fail` so a typo in a `@Cache` region surfaces at startup.

3. **Annotate cacheable entities and associations:**
   ```java
   @Entity
   @Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "product")
   public class Product { /* ... */ }

   @Entity
   public class Order {
       @OneToMany(mappedBy = "order")
       @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)   // cache the collection
       private List<OrderLine> lines = new ArrayList<>();
   }
   ```
   Pick the concurrency strategy deliberately: `READ_ONLY` for immutable reference data, `READ_WRITE` for mutable data with strong consistency, `NONSTRICT_READ_WRITE` only when stale reads are acceptable.

4. **Enable strict mode** so unmapped entities don't silently bypass the cache:
   ```yaml
   spring:
     jpa:
       properties:
         jakarta:
           persistence:
             sharedCache:
               mode: ENABLE_SELECTIVE
   ```

5. **Verify cache effectiveness** with statistics in dev:
   ```yaml
   spring:
     jpa:
       properties:
         hibernate:
           generate_statistics: true
   logging:
     level:
       org.hibernate.stat: DEBUG
   ```
   Look for the hit/miss ratio in logs. A near-zero hit rate means the cache is the wrong tool — the access pattern probably doesn't repeat enough to amortize the cache.

### Validate and Optimize

**1. Verify entity configuration:**
- Test cascade behavior in a transaction before production deployment
- Validate bidirectional relationships sync correctly

**2. Optimize query performance:**
- Run `EXPLAIN ANALYZE` on queries against large tables
- If performance issues detected: add indexes → verify with EXPLAIN → repeat
- Use `@EntityGraph` to prevent N+1 queries

**3. Validate pagination:**
- Ensure indexed columns support pagination queries
- Test with large datasets to verify cursor stability

## Examples

### Basic CRUD Repository

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    // Derived query
    List<Product> findByCategory(String category);

    // Custom query
    @Query("SELECT p FROM Product p WHERE p.price > :minPrice")
    List<Product> findExpensiveProducts(@Param("minPrice") BigDecimal minPrice);
}
```

### Pagination Implementation

```java
@Service
public class ProductService {
    private final ProductRepository repository;

    public Page<Product> getProducts(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
        return repository.findAll(pageable);
    }
}
```

### Entity with Auditing

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdDate;

    @LastModifiedDate
    private LocalDateTime lastModifiedDate;

    @CreatedBy
    @Column(nullable = false, updatable = false)
    private String createdBy;
}
```

## Best Practices

### Entity Design
- Use constructor injection exclusively (never field injection)
- Prefer immutable fields with `final` modifiers
- Use Java records (16+) or `@Value` for DTOs
- Always provide proper `@Id` and `@GeneratedValue` annotations
- Use explicit `@Table` and `@Column` annotations

### Performance Optimization
- Use appropriate fetch strategies (LAZY vs EAGER)
- Implement pagination for large datasets
- Use database indexes for frequently queried fields
- Consider using `@EntityGraph` to avoid N+1 query problems

### Reference Documentation

For comprehensive examples, detailed patterns, and advanced configurations, see:

- [Examples](references/examples.md) - Complete code examples for common scenarios
- [Reference](references/reference.md) - Detailed patterns and advanced configurations

## Constraints and Warnings

- Never expose JPA entities directly in REST APIs; always use DTOs to prevent lazy loading issues.
- Avoid N+1 query problems by using `@EntityGraph` or `JOIN FETCH` in queries.
- Be cautious with `CascadeType.REMOVE` on large collections as it can cause performance issues.
- Do not use `EAGER` fetch type for collections; it can cause excessive database queries.
- Avoid long-running transactions as they can cause database lock contention.
- Use `@Transactional(readOnly = true)` for read operations to enable optimizations.
- Be aware of the first-level cache; entities may not reflect database changes within the same transaction.
- UUID primary keys can cause index fragmentation; consider using sequential UUIDs or Long IDs.
- Pagination on large datasets requires proper indexing to avoid full table scans.

### Second-Level Cache

- Use Hibernate's L2 cache for read-heavy reference data; do not introduce `@Cacheable` / `CacheManager` at the service layer in parallel — caching lives in the persistence layer in this project.
- Pick `CacheConcurrencyStrategy` deliberately: `READ_ONLY` (immutable), `READ_WRITE` (strong consistency, default for mutable data), `NONSTRICT_READ_WRITE` (eventual, narrow use cases).
- Set `hibernate.javax.cache.missing_cache_strategy: fail` in production so typos in region names break the build instead of silently disabling caching.
- Use `ENABLE_SELECTIVE` shared-cache mode so only entities marked `@Cache` participate — opt-in, not opt-out.
- Enable `hibernate.generate_statistics` in dev/staging to confirm hit/miss ratios; if the hit rate is low, the access pattern doesn't justify the cache.
- The query cache (`use_query_cache: true`) is off by default. Opt in per query via `setHint("org.hibernate.cacheable", true)` only after entity-level caching is proven insufficient — it has more pitfalls (invalidation on any table write).
- Caffeine for single-JVM, in-memory; Ehcache when you need off-heap or disk tiers; switch to a distributed cache (e.g. Hazelcast, Infinispan) only when you actually have multiple JVMs needing shared state.
