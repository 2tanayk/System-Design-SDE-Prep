# Application and Database Optimizations Before Scaling

## 1. The problem: avoid scaling away inefficiency

Before upgrading a server or adding instances, investigate whether the current application is doing unnecessary work. An inefficient query, repeated database access, or excessive object creation may be consuming resources that could be saved without adding compute capacity.

Example, using illustrative timings:

| Operation | Time |
|---|---:|
| Application validation and processing | 10 ms |
| PostgreSQL query | 180 ms |
| Serialization and other overhead | 20 ms |
| Total | 210 ms |

Reducing application processing from 10 ms to 5 ms barely changes total latency (about 205 ms). Reducing the slow query from 180 ms to 30 ms brings total latency to about 60 ms. **Optimize the actual bottleneck, not the most convenient component.**

## 2. Measure before making changes

Establish a baseline and observe the endpoint or workflow under representative load.

| Metric | What it helps identify |
|---|---|
| Request rate (RPS) | Incoming workload |
| Latency percentiles (p50, p95, p99) | Typical and slow request behavior |
| CPU utilization | CPU saturation or heavy computation |
| Memory and garbage collection | Heap pressure, allocation, excessive temporary objects |
| Query latency and execution plans | Slow SQL, scans, expensive joins/sorts |
| Connection-pool utilization and wait time | Requests waiting for a database connection |
| Cache hit rate | Whether a cache is reducing origin requests |
| Error rate | Failures under load |

**p95 latency** is the response-time threshold within which 95% of measured requests complete. Average latency alone can hide slow requests. Metrics narrow down the search; they do not automatically prove the root cause.

## 3. Application-level optimizations

### 3.1 Eliminate unnecessary work and round trips

If a request fetches the same restaurant record three times to obtain its name, address, and opening hours, retrieve the required information once and reuse it where appropriate.

Look for:
- Repeated database or external API calls.
- Redundant calculations and parsing.
- Recomputing unchanged values.
- Repeated transformations that can be safely reused.

A **round trip** is the request-and-response communication between components. Round trips add network latency, resource usage, and processing overhead.

Don't combine every query just to reduce the query count: a large complicated query can be worse than a few small independent ones. Optimize total work and measured latency.

### 3.2 Use suitable algorithms and data structures

For repeated membership checks in a large collection, scanning a list may take O(n) time per lookup in the worst case. A hash-based set usually provides O(1) average-case membership lookup, with different memory and ordering tradeoffs and potentially worse worst-case behavior.

Look for expensive nested loops, repeated sorting, unnecessary scans, and repeated computation. Choose structures to fit access patterns rather than converting everything to a hash set.

### 3.3 Avoid blocking resources unnecessarily

Threads can be occupied while waiting for a database, payment service, or other network dependency. If many requests wait on slow dependencies, a request-thread pool can be exhausted even when CPU utilization is low.

Investigate dependency latency, appropriate timeouts, redundant sequential calls, and whether asynchronous processing suits the workflow. Asynchronous execution does not make a slow dependency faster and must not compromise business correctness. For example, an order should not be reported as paid before payment confirmation is obtained.

### 3.4 Reduce unnecessary object creation and memory usage

Loading full entities and large relationships when an endpoint needs only a few fields can create unnecessary objects and allocation pressure.

Consider:
- Selecting only required data.
- Avoiding unnecessary object copies.
- Streaming large results where appropriate.
- Investigating high heap usage and garbage-collection activity.

The JVM manages memory and performs garbage collection for objects that are no longer reachable. Excessive temporary allocation can create additional work and latency. Measure before attempting low-level micro-optimizations.

### 3.5 Return appropriately sized responses

Do not load and serialize 100,000 records if the user needs only 20. Use pagination and select only the required fields. For large datasets, keyset pagination can be preferable to deep offset pagination when the sort order and query support it.

## 4. Database optimizations

### 4.1 Add appropriate indexes

An index is a data structure that helps PostgreSQL locate rows without necessarily scanning the whole table.

For example, if queries frequently filter by customer_id:

~~~sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
~~~

An index may make those queries more efficient, depending on selectivity, table size, and the execution plan.

Indexes have costs:
- They use disk space and may consume memory when cached.
- Inserts, updates, and deletes may need to maintain them.
- Too many unnecessary indexes increase write work and maintenance.

Choose indexes based on actual query patterns, filters, joins, and ordering requirements. Inspect the execution plan and test with representative data. A sequential scan is not inherently bad; it can be optimal when a query reads a large fraction of a table.

PostgreSQL's **EXPLAIN** shows the planned execution strategy. **EXPLAIN ANALYZE** executes the query and reports actual statistics, so use it carefully for statements that may have side effects or significant production impact.

### 4.2 Fix N+1 queries (especially with JPA/Hibernate)

Suppose one query fetches 100 orders, then the application triggers a separate customer query for each order. This can become 101 queries for a single request: the **N+1 query problem**.

Possible remedies, depending on the desired result shape:
- JPQL JOIN FETCH.
- Spring Data JPA @EntityGraph.
- Batch fetching.
- DTO projections that select only needed fields.

Avoid making every relationship globally eager. Fetch joins can also produce duplicate rows or large result sets, particularly across multiple collection associations. Choose a fetch strategy suited to the actual use case.

### 4.3 Select only necessary columns and rows

For a restaurant-list endpoint, returning IDs, names, ratings, and cuisines may be sufficient. Do not retrieve every column and full relationships if they are not needed.

Push simple filters to the database when appropriate, rather than retrieving a huge result and filtering it in Java. This can reduce data transferred, object materialization, and application memory usage.

### 4.4 Rewrite expensive queries based on execution plans

Inspect slow queries for unnecessary joins, huge sorts, redundant subqueries, unexpected scans, or inaccurate row-count estimates. For example, a timestamp range may use a normal index more effectively than wrapping an indexed timestamp column in a date-conversion function, depending on the query and index definitions.

Do not force index usage without evidence; the optimizer may correctly prefer a sequential scan. Optimize the observed plan and verify the result under representative conditions.

### 4.5 Tune connection pools at system level

Opening a database connection has overhead. A connection pool such as HikariCP lets Spring Boot reuse connections.

However, increasing pool size without evidence can overload PostgreSQL. If 10 application instances each have a maximum pool size of 30, the service may collectively request up to 300 connections.

Investigate:
- Active and idle connections.
- Threads waiting for a connection.
- Connection acquisition latency.
- Query duration and database utilization.
- The total connection budget across application instances and other clients.

A pool wait may be caused by slow queries or long-held connections, not simply a pool that is too small.

### 4.6 Keep transactions appropriately scoped

A transaction groups database operations into a logical unit. Spring's @Transactional is commonly used to define transactional boundaries.

Long transactions can hold connections and, depending on the operations, locks and other resources while doing unrelated work such as waiting for an external HTTP call. Keep transactions as short as correctness allows, but do not split them in a way that breaks atomicity or business rules.

Workflows involving payments, orders, or inventory may need explicit state transitions and careful failure handling rather than one long database transaction spanning an external service.

### 4.7 Reduce redundant reads and writes without breaking correctness

Fetching inventory rows together or using suitable bulk operations can reduce round trips. But inventory is shared mutable state: concurrent requests may try to purchase the last item.

A conditional atomic update is one option:

~~~sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 123
  AND quantity > 0;
~~~

Check the affected-row count to determine whether inventory was available. Other workloads may need optimistic locking, pessimistic locking, or appropriate transaction isolation.

**An optimization is not acceptable if it breaks the application's correctness invariants.**

## 5. Caching: avoid repeated expensive reads

A **cache** stores a copy of data so that future requests can retrieve it without repeating an expensive operation.

A common flow is:
1. Check Redis for the requested data.
2. On a cache hit, return the cached result.
3. On a cache miss, query PostgreSQL and, when appropriate, populate Redis.

Caching frequently accessed data such as restaurant details may reduce query load and latency. But cache entries can become stale. Design for data freshness, expiration, invalidation, cache failures, and the impact of many cache misses.

Choose caching when repeated reads are expensive and freshness requirements make it worthwhile. Do not cache data indiscriminately; caching adds operational and consistency complexity.

## 6. Diagnose symptoms rather than blindly applying fixes

| Observation | Investigate |
|---|---|
| CPU is high and profiling shows heavy computation | Algorithms, redundant work, serialization, allocation/GC |
| CPU is low but SQL dominates request time | Query plans, indexes, lock waits, network latency |
| One request executes hundreds of SQL queries | N+1 queries, repeated data access |
| Threads wait for database connections | Pool sizing, slow queries, long transactions |
| The same read-heavy data is fetched repeatedly | Cache suitability, hit rate, freshness/invalidation |
| Responses are huge | Field selection, pagination, payload serialization |
| Locks are held for too long | Transaction scope, contention, external calls inside transactions |
| Requests wait for an external service | Dependency latency, timeouts, concurrency, async workflow design |

These are diagnostic clues, not automatic prescriptions. High CPU or low CPU can each have multiple causes.

## 7. Practical order of operations

1. **Measure:** establish a baseline for latency, throughput, errors, CPU, memory, database queries, and connection pools.
2. **Remove waste:** eliminate redundant application work and excessive data retrieval.
3. **Optimize database access:** inspect SQL plans, indexes, N+1 patterns, connection use, transactions, and contention.
4. **Cache selectively:** only when repeated reads justify the tradeoffs.
5. **Load-test again:** verify the improvement and make sure correctness remains intact.
6. **Scale when needed:** if an efficient system still cannot meet performance targets, increase capacity vertically, horizontally, or both.

Do not delay emergency scaling when production is failing. Scaling out temporarily can restore service while the root cause is investigated.

## 8. Summary card

- **Measure first:** identify the actual bottleneck instead of guessing.
- **Application:** eliminate redundant work, use appropriate data structures, and avoid unnecessary allocation or blocking.
- **Database:** use evidence-based indexes, fix N+1 queries, select only needed data, inspect execution plans, and tune connection pools.
- **Protect correctness:** keep transactions appropriate and preserve concurrency invariants when optimizing writes.
- **Cache selectively:** repeated expensive reads may be cached when freshness and invalidation requirements permit it.
- **Interview one-liner:** “Before scaling, I'd measure the bottleneck and remove waste from the application and data-access paths. If the optimized system still cannot meet its latency or throughput target, I'd add capacity at the constrained tier.”
