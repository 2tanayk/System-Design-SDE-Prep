# Cache Replacement & Eviction Policies — LRU, LFU, SLRU

## 1. Why do we need an eviction policy?

A cache is usually backed by RAM, and RAM is finite. As the application keeps adding cached entries, the cache eventually reaches its configured memory limit.

At that point, it needs a decision:

**Which existing entry should be removed to make room for a new one?**

A cache replacement or eviction policy defines that decision. The goal is not simply to remove the oldest entry; it is to remove an entry whose absence is least likely to hurt future cache hits.

Example: a cache can hold only three documents:

    document:A
    document:B
    document:C

A new document D needs to be cached. The cache must choose one of A, B, or C to evict.

Different policies make different choices based on different signals:
- LRU looks at how recently an entry was accessed.
- LFU looks at how frequently an entry was accessed.
- SLRU separates entries that have only been seen once from entries that have demonstrated reuse.
- FIFO looks at insertion order.

No policy is universally best. The right choice depends on the application's access pattern.

## 2. LRU — Least Recently Used

### Core idea

LRU evicts the entry that has not been accessed for the longest time.

It assumes that something accessed recently is more likely to be accessed again soon. This is called **temporal locality**.

### Dry run

Assume the cache capacity is three entries.

1. Insert A, B, C.
2. Access A.
3. Insert D.

Before inserting D, the recency order from least recent to most recent is:

    B -> C -> A

A was accessed recently, so it moves to the most-recent position. B is now the least recently used entry.

When D is inserted:

    Evict B
    Cache: C, A, D

The important detail is that an access changes the recency order. LRU does not simply evict the oldest inserted entry.

### How can LRU be implemented efficiently?

A typical exact LRU implementation combines:

**HashMap**
- Maps a key to its node in the list.
- Allows the node to be found in O(1) average time.

**Doubly linked list**
- Stores entries in recency order.
- Most recently used entry is at the head.
- Least recently used entry is at the tail.
- Allows a node to be removed and moved to the head in O(1) time.

Operations:
- GET: use the map to find the node, then move it to the head.
- PUT existing key: update the value and move the node to the head.
- PUT new key: add it at the head; if capacity is exceeded, remove the tail and its map entry.

This gives O(1) average-time GET and PUT operations, assuming the HashMap operations are O(1) on average.

### Where LRU works well

- General-purpose application caches.
- Workloads where recently accessed entries are likely to be accessed again.
- Systems where a small, active working set is repeatedly used.

### Limitation: scan pollution

Suppose a cache contains frequently requested configuration entries. Then a one-time report scans thousands of old documents.

A pure LRU cache may keep admitting those one-time documents and evict the genuinely popular entries because each scanned document is more recent than the older hot entries.

LRU tracks recency, not popularity. A single access can make an entry look important.

**Mental model:** keep recently used entries; evict the entry that has been idle the longest.

## 3. LFU — Least Frequently Used

### Core idea

LFU evicts the entry with the lowest access frequency.

Instead of asking, "When was this entry last accessed?", it asks, "How often has this entry been accessed?"

### Dry run

Assume a cache contains three entries:

| Entry | Access count |
|---|---:|
| A | 5 |
| B | 3 |
| C | 1 |

A new entry D needs to be inserted. LFU removes C because it has the lowest access count.

This can make LFU a good fit when a relatively stable set of items receives a disproportionately large share of requests.

Example: a media platform may have a small collection of videos that remain popular for days or weeks. LFU can keep those frequently requested videos cached even when other, newer videos are accessed occasionally.

### Why frequency can be better than recency

Imagine:
- A has been accessed 1,000 times, but its last access was a little while ago.
- B has been accessed only once, very recently.

LRU may prefer B because it is more recent. LFU prefers A because it has a much stronger history of reuse.

### Limitation: stale popularity

Frequency can remember the past for too long.

Suppose a product was extremely popular last month but is no longer being requested. Its high historical frequency may keep it in the cache even though another product is becoming popular now.

A practical LFU design therefore needs some form of **frequency decay or aging**, so old popularity gradually matters less.

Redis uses an approximate LFU algorithm with probabilistic counters and a decay mechanism rather than maintaining an exact, unbounded counter for every key.

**Mental model:** keep frequently accessed entries; evict the entry with the weakest access frequency.

## 4. SLRU — Segmented Least Recently Used

### Core idea

SLRU is designed to reduce a weakness of plain LRU: a one-time access should not necessarily displace an entry that has demonstrated repeated usefulness.

It divides the cache into two LRU segments:

1. **Probationary segment:** newly inserted entries start here.
2. **Protected segment:** entries that have demonstrated reuse are promoted here.

Each segment maintains its own LRU order.

### The rules

1. A newly inserted entry enters the probationary segment.
2. If an entry is accessed again while in probationary, promote it to protected.
3. If an entry in protected is accessed again, move it to the most-recently-used position within protected.
4. If protected exceeds its configured capacity, demote its least-recently-used entry to the most-recently-used position in probationary.
5. When the total cache is full and a new entry must be admitted, evict the least-recently-used entry in probationary first.

The protected segment is therefore reserved for entries that have survived at least one reuse check.

### Dry run

Assume:
- Total cache capacity: 4 entries.
- Protected segment capacity: 2 entries.
- New entries enter probationary.

Request sequence:

    A, B, A, B, C, D, E

Walk through it:

**A**
- A is new, so it enters probationary.

**B**
- B enters probationary and becomes more recent than A.

    Probationary: A, B
    Protected: empty

**A**
- A is accessed again while in probationary.
- Promote A to protected.

    Probationary: B
    Protected: A

**B**
- B is accessed again while in probationary.
- Promote B to protected.

    Probationary: empty
    Protected: A, B

**C, then D**
- Both are new, so they enter probationary.

    Probationary: C, D
    Protected: A, B

**E**
- The cache is full.
- The probationary LRU entry is C.
- Evict C and insert E into probationary.

    Probationary: D, E
    Protected: A, B

A and B remain protected because they demonstrated reuse. A one-time scan of new entries is more likely to be evicted from probationary instead.

### Why SLRU is useful

SLRU provides a balance:
- It uses recency within each segment.
- It requires evidence of reuse before an entry enters protected.
- It helps prevent one-time scans from immediately displacing established hot entries.

### Is SLRU the same as LFU?

No.

SLRU does not maintain an exact lifetime frequency count. An entry is promoted when it is accessed again while in probationary. After promotion, SLRU still uses recency ordering inside the protected segment.

**Mental model:** new entries must prove they are worth keeping; repeated-use entries receive stronger protection.

## 5. FIFO — First In, First Out

### Core idea

FIFO evicts the entry that was inserted earliest, regardless of how often or how recently it has been accessed.

Example:

    Insert A
    Insert B
    Insert C
    Access A
    Insert D

FIFO still evicts A because A was inserted first. The access to A does not change its position in the insertion queue.

### Advantage

- Simple to understand and implement.
- Does not need to maintain recency or frequency metadata on every access.

### Limitation

It can evict a hot entry simply because it is old. That makes it less adaptive than LRU, LFU, or SLRU for many application-cache workloads.

**Mental model:** evict the oldest inserted entry, even if it is popular.

## 6. TTL — Time To Live is different

TTL defines how long an entry is allowed to remain valid. It is an expiration rule, not a popularity-based replacement policy.

Example:

    tenant:42:configuration
    TTL = 10 minutes

After the TTL expires, the entry is no longer considered valid and the cache treats it as expired. A later read may have to reload the value from the primary database.

The distinction:

| Eviction policy | Expiration / TTL |
|---|---|
| Chooses what to remove when memory is under pressure | Defines when an entry becomes expired |
| Usually based on recency, frequency, or insertion order | Based on elapsed time |
| Example: evict the LRU key | Example: expire a key after 10 minutes |

TTL can coexist with LRU or LFU. An entry can expire because its time is up, or be evicted earlier because the cache needs memory.

A TTL does not guarantee that an entry will remain in memory until the TTL ends.

## 7. Choosing between the policies

| Policy | What it measures | Good fit | Main weakness |
|---|---|---|---|
| LRU | Time since last access | General workloads with temporal locality | One-time scans can displace useful entries |
| LFU | Access frequency | Skewed workloads with durable popular items | Old popularity can linger without decay |
| SLRU | Recency across probationary and protected segments | Workloads vulnerable to scan pollution | More policy and bookkeeping complexity |
| FIFO | Insertion order | Simple workloads where recency/frequency tracking is unnecessary | Can evict a frequently used old entry |
| TTL | Age / expiry deadline | Data with a natural freshness limit | Does not by itself choose the best entry under memory pressure |

A practical starting point:
- Start with LRU for a general cache when recently used data is likely to be reused.
- Consider LFU when request frequency is highly skewed and popularity persists over time.
- Consider SLRU when one-time scans or bursts of unique keys are pushing out reusable entries.
- Use TTL when cached data has a freshness requirement, regardless of the replacement policy.

The access pattern and observed hit ratio matter more than choosing the most sophisticated algorithm.

## 8. How Redis handles eviction

Redis is an in-memory data store, so it needs a policy for what happens when its configured memory limit is reached.

### maxmemory

The maxmemory configuration sets the memory limit used by Redis to decide when eviction is needed.

### maxmemory-policy

The maxmemory-policy setting selects the eviction behavior.

Common policies include:

| Policy | Behavior |
|---|---|
| allkeys-lru | Evict least-recently-used keys across all keys |
| allkeys-lfu | Evict least-frequently-used keys across all keys |
| allkeys-random | Evict random keys |
| volatile-lru | Evict least-recently-used keys only among keys with a TTL |
| volatile-lfu | Evict least-frequently-used keys only among keys with a TTL |
| volatile-random | Evict random keys only among keys with a TTL |
| volatile-ttl | Evict keys with a TTL, preferring the shortest remaining TTL |
| noeviction | Do not evict existing keys; reject writes that require more memory |

Important distinction:

- allkeys policies can consider any key.
- volatile policies can consider only keys that have an expiration set.
- If a volatile policy has no eligible expiring keys to evict, Redis behaves like noeviction for that situation.
- noeviction does not mean reads stop working; writes that need additional memory can fail.

Redis also provides Least Recently Modified (LRM) policies in newer versions. LRM tracks modification time rather than access time, so it can be useful when the desired behavior is to evict keys that have not been written recently, even if they are frequently read. This is distinct from the core LRU/LFU/SLRU comparison above.

### Redis LRU and LFU are approximate

Redis does not maintain a perfect, globally exact LRU list for every key. It samples keys and chooses eviction candidates from those samples. This reduces the memory and processing overhead of maintaining exact recency for every access.

Redis LFU is also approximate. It uses compact probabilistic counters and frequency decay so that old popularity gradually loses influence.

Therefore, Redis allkeys-lru is not the same implementation as the textbook HashMap + doubly linked list exact-LRU cache.

### Example configuration

For a general cache where any key can be evicted:

    maxmemory 2gb
    maxmemory-policy allkeys-lru

For a cache where only keys with an expiry are eligible for eviction:

    maxmemory-policy volatile-lru

The correct policy depends on what keys live in the Redis instance and how the application accesses them. Be careful when mixing cache data with important non-cache data in the same instance.

## 9. Monitoring eviction behavior in Redis

Useful commands:

    INFO memory
    INFO stats
    CONFIG GET maxmemory
    CONFIG GET maxmemory-policy

Useful fields include:

- used_memory: memory currently used by Redis.
- evicted_keys: number of keys removed to make room under the memory policy.
- expired_keys: number of keys removed after their TTL expired.
- keyspace_hits: successful key lookups.
- keyspace_misses: lookups where the key was not found.

A simple cache hit-ratio calculation is:

    hit ratio = keyspace_hits / (keyspace_hits + keyspace_misses)

Interpretation:
- Rising evictions combined with a falling hit ratio may indicate that useful entries are being displaced, the cache is undersized, or the chosen policy does not fit the access pattern.
- A high eviction count is not automatically bad. Eviction is expected in a bounded cache; evaluate it alongside hit ratio, latency, memory use, and database load.
- High expired_keys may indicate that many keys are reaching their TTL. Check whether the TTL values match the data's freshness requirements.

## 10. Interview summary

- **LRU:** evicts the least recently accessed entry. A good general-purpose starting point when recent access predicts future access.
- **LFU:** evicts the least frequently accessed entry. Useful when a stable set of keys is consistently popular; frequency decay prevents old popularity from dominating forever.
- **SLRU:** separates probationary and protected entries. A new entry must demonstrate reuse before being protected, reducing scan pollution.
- **FIFO:** evicts the oldest inserted entry, regardless of usage.
- **TTL:** controls expiration by time; it is not a replacement policy.
- **Redis:** uses maxmemory and maxmemory-policy to control memory pressure. Its LRU and LFU implementations are approximate rather than textbook exact implementations.

**Remember:** LRU asks "When was it last used?", LFU asks "How often is it used?", SLRU asks "Has it demonstrated reuse?", FIFO asks "When was it inserted?", and TTL asks "Has its time expired?"

## References

- [User-provided source: Caching in System Design Interviews](https://www.youtube.com/watch?v=1NngTUYPdpI)
- [Redis documentation — Key eviction policies](https://redis.io/docs/latest/reference/eviction/)
