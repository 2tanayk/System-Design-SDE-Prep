# In-Memory Data Stores vs. Disk-Based Databases

## 1. The problem: repeated reads cost resources

A food-delivery application may receive thousands of requests for the same restaurant menu. If every request queries the primary database, the database repeatedly does work to return data that may barely have changed.

A fast data store can keep a copy of frequently accessed data so the application can reuse it. This can reduce repeated database work and improve latency, but adds another component and raises questions about freshness and recovery.

## 2. Core idea: RAM and persistent storage

- **RAM (Random Access Memory):** Working memory used by programs and the operating system. Accessing data already in RAM is generally much faster than reading the same data from SSD or hard disk.
- **Disk storage:** Non-volatile storage that retains data when power is lost.
- **In-memory data store:** A system designed to keep its working dataset in memory for fast access.
- **Disk-based database:** A database that maintains persistent data on storage while also using RAM for caching and query processing.

Do not think of PostgreSQL as reading everything from disk on every query. It uses memory extensively, including caching frequently accessed data pages. Performance also depends on the query, indexes, network latency, and the work required to produce the result.

### PostgreSQL

PostgreSQL keeps its data persistently on disk and uses RAM for cached pages and query processing.

### Redis

Redis is an in-memory data store that supports key-value operations and richer data structures. It can also be configured with persistence options, so “in-memory” does not mean that data can never survive a restart.

### Memcached

Memcached is primarily a volatile, in-memory caching system. Its cache entries are not intended to act as durable authoritative records.

The key distinction is how the system is designed to serve its working dataset and what durability guarantees it provides—not simply “fast RAM versus slow disk.”

## 3. Redis vs. Memcached

| Capability | Redis | Memcached |
|---|---|---|
| Primary role | In-memory data store, commonly used as a cache | In-memory cache |
| Data model | Strings, hashes, lists, sets, sorted sets, and other supported structures | Primarily values retrieved by key |
| Expiration | Supported | Supported |
| Persistence | Optional persistence mechanisms | Primarily volatile; cached entries are not a durable dataset |
| Operations | Rich data structures and atomic operations | Simple cache operations and basic atomic counters |
| Typical fit | Caching plus richer key-oriented data and coordination use cases | Straightforward distributed caching |

Neither is universally better. Memcached can fit a simple cache, while Redis is useful when its richer functionality is required. Extra functionality also introduces additional choices and operational considerations.

Redis can be used for sessions, counters, rate limiting, and other use cases, but each use case needs its own analysis of durability, correctness, and failure behavior.

## 4. The common production pattern: PostgreSQL + Redis

For a food-delivery application, PostgreSQL can remain the **source of truth** for restaurant menus. The source of truth is the authoritative place where canonical data is maintained.

Redis stores copies of frequently accessed menu data to accelerate reads.

~~~text
Customer request
      |
      v
Spring Boot API
      |
      v
  Check Redis
   /       \
 HIT       MISS
  |          |
  v          v
Return    Query PostgreSQL
data          |
              v
         Store in Redis
              |
              v
         Return data
~~~

This is called **cache-aside**:
1. The application checks the cache first.
2. If the key exists, it returns the cached value.
3. If the key is missing, it fetches the data from the primary database.
4. It stores the result in the cache when appropriate and returns it.

## 5. Dry run: retrieving a restaurant menu

Suppose PostgreSQL contains this menu:

| Item | Price |
|---|---:|
| Paneer Biryani | ₹250 |
| Raita | ₹50 |
| Mango Lassi | ₹100 |

### Request 1 — cache miss
1. Spring Boot asks Redis for the key restaurant:101:menu.
2. Redis has no entry.
3. Spring Boot fetches the menu from PostgreSQL.
4. The application stores the result in Redis.
5. The API returns the menu to the customer.

### Request 2 — cache hit
1. Spring Boot asks Redis for the same key.
2. Redis returns the cached menu.
3. The API returns the result without querying PostgreSQL for that data.

Caching can reduce repeated reads when data is frequently requested and the cache entry remains valid. The benefit depends on cache hit rate and the relative cost of the cache and database operations.

## 6. What if the menu changes?

Suppose the price changes from ₹250 to ₹280 in PostgreSQL, but Redis still contains ₹250. The cached copy is now **stale**—it no longer matches the authoritative value.

### Option A: TTL-based expiration

**TTL (Time to Live)** specifies how long a cache entry may remain valid before it expires.

For example, an entry with a five-minute TTL expires after five minutes. If the next request finds the key missing, the application can reload the value from PostgreSQL.

Choose TTL when some staleness is acceptable and simple cache management is valuable. TTL alone does not guarantee that a cached value is always current; a change can happen before the entry expires.

### Option B: Invalidate the cache after updates

After updating PostgreSQL, the application deletes the corresponding Redis key. The next request misses the cache and fetches fresh data.

Choose invalidation when changes should become visible sooner and the application can manage the failure cases.

Invalidation can fail after the database commits. For example, PostgreSQL may have the new price while Redis still contains the old one. Combining invalidation with TTL gives a fallback, while higher-reliability designs may use dependable event delivery and reconciliation.

For critical operations—such as charging a customer—do not blindly trust a potentially stale cache. Validate prices and business rules at the appropriate authoritative boundary.

## 7. Why not store everything in Redis?

### 7.1 RAM is limited and can be expensive

Keeping a large historical dataset entirely in memory may cost more than using persistent disk storage. The machine also needs RAM for the operating system, applications, and other data.

Use memory where its latency benefit is worth the capacity and cost. Do not cache data indiscriminately.

### 7.2 Durability requirements differ

A cached menu can often be reconstructed from PostgreSQL if Redis loses the entry. The same assumption is unsafe for the only copy of a payment record.

Redis offers persistence options, but you must deliberately evaluate durability, recovery, consistency, and failure guarantees for the chosen use case. For critical relational business records, PostgreSQL is a natural default in this stack.

### 7.3 Query and data-model capabilities differ

PostgreSQL provides relational queries, joins, constraints, and transactions. Redis is optimized for key-based access and supported data structures rather than acting as a drop-in replacement for PostgreSQL's relational query model.

Choose each system for the access patterns and guarantees it supports well.

### 7.4 A cache introduces consistency work

Once data exists in PostgreSQL and Redis, the application must decide how cached copies are refreshed or invalidated. That creates additional failure scenarios and operational complexity.

## 8. Cache stampede and cache failures

A **cache stampede** happens when many requests miss the cache simultaneously and all hit the underlying database for the same data. This can occur when a popular entry expires or is evicted.

Possible mitigations:
- Refresh frequently used entries before expiry.
- Let one request refresh a key while others wait for or reuse the result.
- Add jitter to TTLs so many keys do not expire together.
- Limit the concurrency sent to PostgreSQL.

Redis itself can also become unavailable. A fallback to PostgreSQL may be appropriate, but without protective limits a cache outage can overwhelm the database. Use timeouts, concurrency controls, and monitoring that fit the failure requirements.

## 9. Practical choices

| Scenario | Starting choice | Why |
|---|---|---|
| Orders, payments, and inventory | PostgreSQL | Durable relational business data, constraints, and transactions |
| Frequently accessed restaurant menus | Redis + PostgreSQL | Redis accelerates repeated reads; PostgreSQL remains the source of truth |
| Straightforward cached API responses | Redis or Memcached | Fast key-based lookup with expiration |
| Shared session data | Redis may fit | Shared access across application instances and TTL support |
| Complex reports over related entities | PostgreSQL | SQL joins and aggregation |
| Large historical dataset | Suitable persistent database or storage system | Keeping everything in RAM may be costly |

These are starting points, not rigid rules. Redis can be configured for persistent use cases when its guarantees fit, and PostgreSQL can serve many workloads directly without a separate cache.

## 10. Summary card

- **Idea:** In-memory data stores serve a working dataset from RAM; disk-based databases persist data on storage and also use RAM to improve performance.
- **Roles:** PostgreSQL is a natural source of truth for relational business data; Redis and Memcached commonly speed up repeated reads.
- **Project connection:** Use cache-aside for frequently accessed restaurant menus—check Redis, fetch PostgreSQL on a miss, and cache the result.
- **Tradeoff:** RAM can reduce access latency, but capacity is limited; caching introduces stale data, invalidation, and failure-handling concerns.
- **Watch-outs:** Cache stampedes, Redis outages, low cache hit rates, and unbounded fallback traffic can shift load back to the database.
- **Interview one-liner:** “I'd use PostgreSQL for durable relational data and Redis or Memcached to accelerate frequently accessed data. The choice depends on access patterns, durability, latency targets, memory cost, and how cached data is kept sufficiently fresh.”
