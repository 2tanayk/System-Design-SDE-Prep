# Cache Write Policies — Write-Through, Write-Back, and Write-Around

## 1. The problem write policies solve

A cache stores a copy of data that also exists in the primary database. Reads can be served from the cache, but what should happen when the application changes the data?

Example: PostgreSQL and Redis both contain:

    document:501 = "Project Proposal"

A user renames the document:

    Project Proposal -> Final Project Proposal

PostgreSQL is updated, but Redis may still contain the old name. The write policy defines how the write path updates, bypasses, or defers updates to the cache and database.

The three policies covered here are:

1. **Write-Through:** update the cache and synchronously persist to the database before acknowledging success.
2. **Write-Back / Write-Behind:** update the cache, acknowledge quickly, and persist to the database asynchronously.
3. **Write-Around:** bypass the cache on writes; reads populate the cache later.

## 2. Write-Through

### Core idea

In a write-through design, the application writes through the cache layer. The cache layer synchronously propagates the write to the backing database and the application receives success only after the database write has completed successfully.

Conceptual flow:

    Application
        |
        | write new value
        v
    Cache layer
        |
        | synchronous database write
        v
    PostgreSQL
        |
        | database write succeeds
        v
    Cache holds new value
        |
        v
    Application receives success

Example: rename document 501.

    Old value: Project Proposal
    New value: Final Project Proposal

After a successful write:
- Redis contains Final Project Proposal.
- PostgreSQL contains Final Project Proposal.
- The application has received success.

### Advantages

- The cache is fresh immediately after a successful write.
- Subsequent reads can return the new value from cache.
- It can be useful when data is read frequently after being updated.

### Disadvantages

**Higher write latency:** the application waits for the synchronous backing-database write as well as the cache operation.

**Cache pollution:** a newly written value may be added to the cache even if nobody reads it.

**Dual-write failure problem:** the cache and database are separate systems. One operation may succeed while the other fails.

Example:

    Application -> Cache layer -> PostgreSQL
                                  |
                                  | commit succeeds
                                  v
                             Cache update fails

Now PostgreSQL has the new value, but Redis may still have the old value. The design needs retry, invalidation, reconciliation, or stronger coordination to recover safely.

### When it fits

- Frequently read configuration that is also updated.
- Data where reads immediately after a successful write should see the new value.
- Workloads where the additional write latency is acceptable.

**Mental model:** update the cache through a synchronous write path; don't acknowledge success until the backing database write has succeeded.

### Implementation caveat

Write-through is an architectural pattern, not something a plain Redis instance automatically performs against PostgreSQL. Redis does not natively know how to write a document row to your PostgreSQL database. The application, cache abstraction, or dedicated integration layer must implement and coordinate the write path.

## 3. Write-Back (Write-Behind)

### Core idea

In write-back, the application writes to the cache and the cache acknowledges the write immediately. The backing database is updated asynchronously, often by batching multiple changes.

Conceptual flow:

    Application
        |
        | write
        v
    Cache / write buffer
        |
        +----> acknowledge application quickly
        |
        v
    Background worker / flush process
        |
        | asynchronous, possibly batched
        v
    PostgreSQL

### Example: document view counter

Suppose a popular document receives a large number of views. Persisting every increment to PostgreSQL individually may create unnecessary write load.

Instead, the cache accumulates increments:

    Redis counter for document:501 = 10,000

A background process later flushes the accumulated amount:

    UPDATE documents
    SET view_count = view_count + 10000
    WHERE id = 501;

This can replace many individual database updates with one aggregated update.

### Advantages

- Very low write acknowledgement latency.
- Multiple updates can be combined into batches.
- Reduces the number of writes sent to the primary database.

### Disadvantages

**Risk of data loss:** acknowledged changes may exist only in the cache or write buffer. If the cache node fails before changes are safely persisted, some acknowledged data may be lost.

**Database lag:** PostgreSQL can temporarily contain an older value than the cache.

**Operational complexity:** the system must manage background flushes, batch sizes, retries, duplicate delivery, idempotency, recovery, and reconciliation.

Redis persistence and replication can reduce some failure risks, but they do not automatically make an asynchronous write-back design equivalent to synchronously committing every business write to PostgreSQL.

### When it fits

Write-back is appropriate when:
- Write volume is very high.
- Updates can be aggregated or batched.
- Immediate durable persistence is not required for every individual update.
- The business explicitly accepts the remaining data-loss risk and has a recovery strategy.

Examples:
- View counters.
- Analytics aggregation.
- Telemetry and metrics.
- Usage statistics where minor loss or delay is acceptable.

Do not use a naive write-back cache as the only durability mechanism for financial transactions, inventory reservations, or other critical business records.

**Mental model:** acknowledge the write from the cache quickly; persist it to the database later.

## 4. Write-Around

### Core idea

In write-around, writes bypass the cache and go directly to the database. The cache is populated only when the data is subsequently read.

Conceptual flow:

    Application
        |
        | write directly
        v
    PostgreSQL

    Redis is not populated by this write.

On a later read:

    Application
        |
        v
    Redis lookup
        |
       MISS
        |
        v
    PostgreSQL read
        |
        v
    Populate Redis
        |
        v
    Return result

### Example: newly created document

The application creates a document directly in PostgreSQL:

    INSERT INTO documents (id, name)
    VALUES (501, 'Project Proposal');

Redis is not populated just because the row was inserted.

When the document is requested for the first time:
1. Application checks Redis.
2. Redis returns a miss.
3. Application reads PostgreSQL.
4. Application stores the returned document in Redis.
5. Application returns the response.

Only data that is actually read is added to the cache.

### Advantages

- Avoids filling the cache with records that may never be read.
- Simple write path: write directly to the primary database.
- Useful for write-heavy data where only a small subset is read later.

### Disadvantages

**First read is slower:** because the write did not populate Redis, the first subsequent read is a cache miss.

**Existing cached entries can become stale:** if Redis already contains document:501 and the application updates PostgreSQL directly, Redis still has the old value unless it is invalidated or refreshed.

Example:

    Redis before write:
        document:501 = Project Proposal

    PostgreSQL update:
        Project Proposal -> Final Project Proposal

    Redis after database update, if untouched:
        document:501 = Project Proposal  (stale)

A production write-around flow commonly invalidates the corresponding cache key after the database transaction commits:

    1. Update PostgreSQL.
    2. Commit the database transaction.
    3. Delete document:501 from Redis.
    4. Next read misses the cache and loads the updated database value.
    5. Repopulate Redis.

Redis command:

    DEL document:501

Invalidation should be coordinated carefully with transaction completion and concurrent readers. A simple delete-after-commit pattern is common, but it does not eliminate every possible race where a concurrent miss loads an old value and writes it back into the cache.

### When it fits

- Data is written frequently but read infrequently.
- Many newly created records may never be accessed again.
- Large datasets where caching every written record would waste memory.
- Logs, archive records, or other write-heavy data where later reads are selective.

**Mental model:** write to the database directly; let a later read populate the cache.

## 5. Compare the three policies

| Property | Write-Through | Write-Back / Write-Behind | Write-Around |
|---|---|---|---|
| Write path | Through cache layer | Through cache/write buffer | Directly to database |
| Database write | Synchronous | Asynchronous | Synchronous |
| Cache populated/updated on write? | Yes | Yes | No |
| Application waits for database? | Yes | No | Yes |
| Write acknowledgement latency | Higher | Lower | Usually lower than write-through |
| Can batch database writes? | Not the defining behavior | Yes | Not the defining behavior |
| Cache pollution risk | Yes | Yes | Reduced |
| Risk before DB persistence | Depends on coordinating both writes | Higher: acknowledged data may not yet be durable in DB | Database commit has already occurred |
| First read after a write | Usually a cache hit | Usually a cache hit, though DB may lag | Cache miss if no entry exists |
| Main fit | Fresh cache after writes | High-throughput batchable writes | Write-heavy, infrequently read data |

## 6. Same update, three different flows

Initial state:

    PostgreSQL: document:501 = "Old Name"
    Redis:      document:501 = "Old Name"

The user changes the value to New Name.

### Write-Through

- Write path goes through the cache layer.
- The backing database is updated synchronously.
- The cache holds the new value.
- The application receives success after the database write succeeds.

### Write-Back

- Cache is updated and acknowledges immediately.
- PostgreSQL is updated later by a background process.
- Redis may show New Name while PostgreSQL still has Old Name.
- A cache failure before persistence can lose acknowledged changes.

### Write-Around

- PostgreSQL is updated directly.
- Redis is not populated by the write.
- If the old value was already cached, it must be invalidated or refreshed.
- The next read misses and repopulates the cache from PostgreSQL.

## 7. Choosing a policy for different workloads

### Scenario A: Tenant configuration

A configuration record is read frequently and updated occasionally. Reads immediately after a successful update should observe the new configuration.

Write-Through is one possible fit because synchronously updating the cache as part of a successful write is useful. The dual-write failure case still needs a recovery strategy.

### Scenario B: Document view counters

A popular document receives a very high volume of views. Persisting every increment individually may overload PostgreSQL, and a small amount of metric loss may be acceptable.

Write-Back can be a fit because increments can be accumulated and persisted in batches. Use it only with an explicit durability, retry, and recovery strategy.

### Scenario C: Large document archive

Users continuously create archive records, but only a small subset is read afterwards.

Write-Around can be a fit because it avoids populating Redis for every newly created record. Only later reads populate the cache.

## 8. Cache-Aside vs. Write-Around

These terms are related but describe different sides of caching.

| Cache-Aside | Write-Around |
|---|---|
| Primarily a read strategy | A write strategy |
| Application checks cache before reading | Application bypasses cache for writes |
| On a miss, application reads DB and populates cache | A later read can miss, load DB, and populate cache |
| Describes how reads are handled | Describes how writes are handled |

A common combination is:

**Cache-Aside for reads + Write-Around for writes.**

The application:
1. Checks Redis before a read.
2. Reads PostgreSQL on a cache miss.
3. Populates Redis after the miss.
4. Writes updates directly to PostgreSQL.
5. Invalidates the relevant Redis key after the database transaction commits.

This is a practical design for many PostgreSQL-backed applications.

### Invalidation race to be aware of

Even delete-after-commit can have a race:

1. Reader A misses Redis and starts fetching the old database value.
2. Writer B updates PostgreSQL and commits.
3. Writer B deletes the Redis key.
4. Reader A finishes its earlier database read and writes the old value into Redis.

The cache is stale again.

Possible mitigations include versioning, compare-and-set style population, CDC-driven invalidation, or other coordination approaches. The right choice depends on how serious stale reads are and will be explored further in cache-consistency discussions.

## 9. Interview summary

- **Write-Through:** writes go through the cache layer and are synchronously persisted to the database before success is acknowledged. The cache is fresh after a successful write, but write latency and dual-write failure complexity increase.
- **Write-Back / Write-Behind:** cache acknowledges immediately and database persistence happens asynchronously, often in batches. This improves write throughput but introduces durability and recovery risks.
- **Write-Around:** writes bypass the cache and go directly to the database. Later reads populate the cache. It avoids cache pollution but can cause first-read misses and requires handling existing stale cache entries.
- **Cache-Aside + Write-Around:** a common combination: check/populate cache on reads, write directly to the database, then invalidate the affected key after commit.
- Always explain what happens when the cache or database fails, and whether acknowledged data can be lost.

**Remember:** Write-Through = synchronous database persistence; Write-Back = asynchronous persistence; Write-Around = bypass cache on writes.

## References

- [User-provided source: Caching in System Design Interviews](https://www.youtube.com/watch?v=1NngTUYPdpI)
- [Redis — Cache-Aside Pattern](https://redis.io/docs/latest/develop/use-cases/cache-aside/)
- [Redis — Data Types](https://redis.io/docs/latest/develop/data-types/)
