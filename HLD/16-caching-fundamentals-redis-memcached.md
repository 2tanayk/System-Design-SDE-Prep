# Caching Fundamentals — Redis, Memcached, and Use Cases

## 1. Why do we cache?

Imagine an API endpoint repeatedly reads the same document from PostgreSQL:

    SELECT *
    FROM documents
    WHERE id = 501;

If document 501 is requested thousands of times and rarely changes, repeatedly executing the same database query is unnecessary work.

Caching keeps a temporary copy of frequently accessed data in a faster storage layer. Subsequent requests can use that copy instead of repeatedly querying the primary database.

Caching is commonly introduced to:
- Reduce read latency.
- Reduce repeated work on the primary database.
- Reduce database query volume and connection pressure.
- Help the system handle read-heavy traffic.

**Trade-off:** caching improves performance but introduces another copy of the data. That creates concerns around freshness, invalidation, memory usage, and cache failures.

## 2. What is a cache?

A **cache is a storage layer that keeps copies of data so that future requests can retrieve it faster.**

In a typical backend architecture:

    Client
      |
      v
    Application
      |
      +----> Redis cache
      |
      +----> PostgreSQL (source of truth)

PostgreSQL remains responsible for durable relational data and business transactions. Redis stores copies of selected data that the application can quickly retrieve.

A cache is usually not the authoritative source of business data. If a cached document disappears, the application should normally be able to load it again from PostgreSQL.

## 3. Cache hit and cache miss

### Cache hit

A **cache hit** occurs when the requested key already exists in the cache.

Example: a request arrives for document 501.

    1. Application receives GET /documents/501.
    2. Application checks Redis for document:501.
    3. Redis returns the cached document.
    4. Application returns the response.

PostgreSQL is not queried for this request.

### Cache miss

A **cache miss** occurs when the requested key is not present in the cache.

    1. Application receives GET /documents/502.
    2. Application checks Redis for document:502.
    3. Redis returns a miss.
    4. Application queries PostgreSQL.
    5. PostgreSQL returns the document.
    6. Application stores a copy in Redis.
    7. Application returns the response.

The next request for document 502 can be served from Redis.

This common pattern is called **Cache-Aside (Lazy Loading)**: the application checks the cache, loads from the database on a miss, and then populates the cache. It is a sensible default when the application wants to cache only data that is actually requested.

A cache miss is not an error by itself. It is a normal part of cache operation.

## 4. TTL and cache hit ratio

### TTL — Time To Live

TTL defines how long a cache entry remains valid before it expires.

Example Redis command:

    SET document:501 '{"name":"Architecture Design"}' EX 300

- Key: document:501
- Value: serialized document data
- EX 300: expire the key after 300 seconds (five minutes)

After expiry, a subsequent cache lookup returns a miss and the application can fetch a fresh copy from the primary database.

TTL bounds how long an untouched entry remains in the cache, but it does not by itself solve every stale-data problem. An application may also need to invalidate or update a key when its underlying data changes.

### Cache hit ratio

The hit ratio is the proportion of cache requests that find the requested data.

    Hit ratio = Cache hits / Total cache requests

Example:

    Total cache requests = 1,000
    Cache hits           =   900
    Cache misses         =   100

    Hit ratio = 900 / 1,000 = 90%

A higher hit ratio generally means fewer requests reach the database. However, a high hit ratio is useful only when the cached data is sufficiently fresh and the cache lookup itself is worthwhile.

## 5. Why caching improves performance

Consider an illustrative workload:

| Operation | Illustrative latency |
|---|---:|
| Redis lookup | 2 ms |
| PostgreSQL query | 40 ms |

Assume:
- 90% of requests are cache hits and take 2 ms.
- 10% are misses and take 2 ms for the Redis lookup plus 40 ms for the PostgreSQL query.

Simplified average latency:

    Average = 0.90 × 2 ms + 0.10 × (2 ms + 40 ms)
            = 1.8 ms + 4.2 ms
            = 6 ms

The database-only query was 40 ms in this illustration; the cache reduces the average request latency to 6 ms.

These are illustrative values, not a benchmark. Actual latency depends on network placement, payload size, serialization, query plans, contention, and cache hit ratio.

### Reducing database load

Imagine 10,000 requests for the same document.

Without caching, all 10,000 requests may query PostgreSQL.

With a warm cache and a 90% hit ratio:
- 9,000 requests are served by the cache.
- 1,000 requests miss and query PostgreSQL.

That is a 90% reduction in database reads for this particular workload.

Reducing repeated queries can also reduce connection-pool pressure and free database resources for writes and queries that genuinely need PostgreSQL.

## 6. What should we cache?

Caching is most useful when data is:
- Read frequently.
- Relatively expensive to retrieve or compute repeatedly.
- Not changing so frequently that maintaining freshness becomes harder than the performance benefit.
- Small or valuable enough to justify keeping a copy in memory.

Examples:

| Data | Cache suitability | Reason |
|---|---|---|
| Public document metadata | High | Repeated reads, relatively infrequent changes |
| Tenant configuration | High | Read frequently, changed occasionally |
| User profile | High | Many reads, comparatively fewer updates |
| Product/category catalog | High | Read-heavy and predictable |
| Search suggestions | High | Repeated lookups; short staleness may be acceptable |
| Live account balance | Requires caution | Stale data can lead to incorrect decisions |
| One-time computation used once | Low | Cache overhead may exceed the benefit |

The objective is to identify **hot data**: data accessed disproportionately often. Do not cache everything by default.

## 7. Where can caching exist?

Caching can be introduced at several layers. The right location depends on what data is being cached and who needs to share it.

| Layer | Examples | Good fit | Main caveat |
|---|---|---|---|
| External/distributed cache | Redis, Memcached | Shared data across multiple application instances | Network hop and another service to operate |
| CDN/edge cache | Cloudflare, Fastly, Akamai | Static images, videos, JavaScript and CSS near users | Cache invalidation and content freshness |
| Client-side cache | Browser HTTP cache, local storage, mobile storage | Avoiding network calls; offline or user-local data | Backend has limited control; data can become stale |
| In-process cache | Application RAM, Caffeine, Guava | Very small, frequently read, rarely changing data | Fastest lookup, but not shared across instances |

For a high-traffic backend with multiple stateless application instances, an external cache such as Redis is a common starting point because all application instances can use the same cache.

In-process caching can be a secondary optimization for tiny, very hot values, but each application instance maintains its own copy.

CDNs and client-side caching are particularly useful for content that can be cached closer to the user. Their detailed behavior can be studied separately.

## 8. Redis fundamentals

Redis is an in-memory data structure store commonly used as a cache, database, and messaging/streaming component.

It exposes a key-based interface, but unlike a basic object cache, it supports multiple native data structures and atomic operations.

Simple example:

    SET user:101:name "Tanay"
    GET user:101:name

Result:

    "Tanay"

A Redis key can store different kinds of values, depending on the data type chosen.

### Common Redis data structures

| Data type | Mental model | Example use |
|---|---|---|
| String | Single value or byte sequence | Cached JSON, tokens, counters |
| Hash | Field-value map/object | User metadata, configuration |
| List | Ordered sequence | Simple queues, recent items |
| Set | Collection of unique members | Permissions, membership |
| Sorted Set | Members ordered by score | Leaderboards, rankings |
| Stream | Append-oriented event log | Event processing |

### Examples

**String**

    SET document:501 '{"name":"Architecture Design"}'
    GET document:501

Useful for serialized API responses, simple values and counters.

**Hash**

    HSET user:101 name "Tanay" city "Pune" role "USER"
    HGET user:101 city
    HGETALL user:101

Useful when fields need to be read or updated individually.

**List**

    LPUSH notifications:user:101 "New document shared"
    LRANGE notifications:user:101 0 -1

Useful for ordered collections and simple queue-like workflows.

**Set**

    SADD user:101:permissions "READ" "WRITE" "SHARE"
    SISMEMBER user:101:permissions "WRITE"

Useful for unique membership and set operations.

**Sorted Set**

    ZADD leaderboard 1500 "user:101"
    ZADD leaderboard 2200 "user:102"
    ZADD leaderboard 1800 "user:103"
    ZREVRANGE leaderboard 0 2 WITHSCORES

Useful for score-based rankings and ordered results.

Redis also supports structures such as bitmaps, HyperLogLog, geospatial indexes and time series. The key point for this topic is that Redis can do more than store serialized objects.

## 9. Memcached fundamentals

Memcached is a distributed in-memory key-value cache designed primarily for storing and retrieving serialized objects.

Its model is intentionally simple:

    KEY:   document:501
    VALUE: {"name":"Architecture Design","status":"PUBLISHED"}

A client asks Memcached for a key. If it exists, Memcached returns the stored value; otherwise, the application loads the data from its primary store.

Memcached focuses on simple object caching:
- Keys map to serialized values.
- Expiration is supported.
- It uses an LRU-oriented eviction approach by default.
- Servers are independent; clients commonly distribute keys across servers using client-side hashing.
- It does not provide Redis-style built-in server replication or native rich data structures.

## 10. Redis vs. Memcached

| Feature | Redis | Memcached |
|---|---|---|
| Primary purpose | In-memory data structure store, often used as a cache | Simple in-memory object cache |
| Data model | Strings, hashes, lists, sets, sorted sets, streams, etc. | Keys mapped to serialized values |
| TTL | Supported | Supported |
| Eviction | Configurable policies | LRU-oriented eviction by default |
| Atomic operations | Rich operations on native data structures | Basic operations, counters and CAS support |
| Persistence | Optional RDB/AOF persistence | Primarily ephemeral cache |
| Replication | Supported | No built-in server replication |
| Clustering/distribution | Redis Cluster distributes keys across nodes | Client-side distribution across independent servers |
| Additional capabilities | Pub/Sub, Streams, scripting, transactions | Intentionally focused on caching |
| Good fit | Cache plus sessions, counters, rate limits and other structures | Simple, disposable object caching |

### Which one is faster?

Do not claim Redis is always faster than Memcached.

Both are designed for fast in-memory access. Actual performance depends on:
- Workload and data size.
- Serialization and payload size.
- Network topology.
- Concurrency.
- Data structure and operation.
- Client implementation and configuration.

For simple serialized-object caching, Memcached may be perfectly sufficient. Redis is attractive when the application needs more than a basic key-value object cache.

**Redis is more versatile; that does not mean it is automatically the better choice for every workload.**

## 11. Why Redis alongside PostgreSQL?

PostgreSQL and Redis solve complementary problems.

### PostgreSQL remains the source of truth

Use PostgreSQL for:
- Durable business records.
- Relational queries and joins.
- Transactions and constraints.
- Foreign-key relationships.
- Data that must survive cache eviction or cache restarts.

### Redis provides fast access and operational data structures

A PostgreSQL-backed backend might use Redis for:

| Requirement | Redis capability |
|---|---|
| Frequently requested document metadata | String or Hash cache entry |
| Shared session state | Keys/Hashes with expiry |
| API request counters | Atomic increment operations |
| User permissions or membership checks | Sets |
| Leaderboards | Sorted Sets |
| Temporary verification codes | Strings with TTL |
| Event or work-stream use cases | Streams or an application-specific queue |

Redis can also support Pub/Sub, Streams and optional persistence. These features are useful when the workload requires them, but they do not turn Redis into a replacement for a relational source of truth.

### Why shared Redis helps horizontally scaled applications

Suppose multiple application instances run behind a load balancer:

    Load Balancer
       |     |     |
       v     v     v
     App A  App B  App C
       |     |     |
       +-----+-----+
             |
           Redis
             |
         PostgreSQL

If App A caches document 501, App B and App C can access the same external cache entry. The cache is not tied to the memory of one application instance.

This is also useful for shared sessions and counters that must work across instances.

### Reusing an existing Redis deployment

If Redis already supports another workload, such as a background-job queue, it may be convenient to reuse the technology for caching or counters.

However, do not blindly mix workloads:
- A memory-evicting cache policy can be dangerous for important queue or session state.
- Cache traffic can compete with queue operations for memory and CPU.
- Separate deployments or carefully isolated capacity and policies may be necessary.
- Separate logical Redis databases do not isolate CPU, RAM or eviction pressure.

## 12. Why not use PostgreSQL for everything?

PostgreSQL already uses memory extensively:
- shared_buffers keeps database pages in memory.
- The operating system can cache file data.
- Indexes help locate rows efficiently.

So a repeated query may already be fast because the relevant data pages are in memory.

But there is a difference:

**PostgreSQL's buffer cache accelerates database operations. Redis can avoid executing the database operation altogether.**

Without an application-level cache:

    Request
       |
       v
    PostgreSQL
       |
       +-- Use an index to locate the row
       +-- Read relevant pages from memory/disk cache
       +-- Execute the query and construct the result
       |
       v
    Return result

With Redis:

    Request
       |
       v
    Redis key lookup
       |
       v
    Return cached result

The Redis path avoids PostgreSQL query processing, even when PostgreSQL already has the data pages in memory.

However, Redis introduces a network hop, serialization work and another infrastructure component. It is not automatically better for every query.

## 13. When should we avoid caching?

Caching may not be worthwhile when:
- Data is rarely read.
- The database query is already extremely cheap.
- Data changes constantly.
- Stale data would violate a critical business rule.
- The payload is too large or expensive to keep in RAM.
- Cache invalidation and operational complexity outweigh the performance benefit.

**Golden rule:** Add a cache when it removes more cost than it creates.

## 14. Common problems to anticipate

These are introduced here at a high level and will be covered in dedicated lessons.

| Problem | What happens | Example mitigation to explore later |
|---|---|---|
| Stale data / invalidation | Database changes but cached copy remains old | Invalidate or refresh the key; TTL as a backstop |
| Cache stampede | Many requests miss the same expired key and overload the database | Request coalescing, proactive refresh, TTL jitter |
| Hot key | One popular key overloads a cache node | Key replication/splitting, in-process cache |
| Eviction | Memory pressure forces the cache to remove entries | Choose an appropriate eviction policy and capacity |
| Cache outage | Application cannot reach Redis | Defined fallback, timeouts, load protection and failure policy |

A Redis cache should usually be treated as a recoverable performance layer. But this fallback is not appropriate for every Redis use case: sessions, rate-limit state and queues may have different correctness and failure requirements.

## 15. Interview summary

- **Cache:** temporary storage of frequently accessed data to reduce latency and repeated work on the primary data store.
- **Cache hit:** requested key exists. **Cache miss:** key is absent, so the application loads from the source and may populate the cache.
- **TTL:** how long an entry remains valid before expiry.
- **Hit ratio:** cache hits divided by total cache requests.
- **Cache-Aside:** application checks the cache, reads the database on a miss and populates the cache. A common default.
- **Redis:** in-memory data structure store with strings, hashes, lists, sets, sorted sets, streams and atomic operations.
- **Memcached:** simple in-memory object cache focused on key-value storage.
- **Why Redis with PostgreSQL?** PostgreSQL remains the durable relational source of truth; Redis provides fast shared lookups and temporary operational data structures.
- **Do not cache everything:** cache data that is frequently read and sufficiently stable, and account for staleness, invalidation, memory and failures.

## References

- [Redis — Cache-Aside Pattern](https://redis.io/docs/latest/develop/use-cases/cache-aside/)
- [Redis — Data Types](https://redis.io/docs/latest/develop/data-types/)
- [Redis — Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Memcached Documentation](https://docs.memcached.org/)
- [User-provided source: Caching in System Design Interviews](https://www.youtube.com/watch?v=1NngTUYPdpI)
