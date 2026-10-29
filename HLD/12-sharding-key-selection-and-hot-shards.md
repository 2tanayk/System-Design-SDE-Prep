# Sharding — Key Selection and the Hot-Shard Problem

## 1. Why sharding exists

A single database server has finite storage, CPU, memory, and I/O capacity. Before introducing sharding, a team should consider query optimization, suitable indexes, partitioning, larger database resources, and read replicas. Those options may be enough.

**Sharding distributes a dataset across multiple database servers.** Each server owns only part of the dataset. The application or a routing layer determines which shard holds the data for a request.

Conceptually:

    Application
        |
        v
    Shard router
        |
        +------> Shard A: one subset of records
        +------> Shard B: another subset of records
        +------> Shard C: another subset of records

A shard is an independently managed database node or database partition of ownership. A shard may itself use table partitioning and indexes.

## 2. Partitioning versus sharding

- **Partitioning** splits a table into smaller physical partitions within one logical database. It can help PostgreSQL prune irrelevant partitions and manage large data ranges.
- **Sharding** splits data across separate database servers, allowing storage and workload to be distributed across independent resources.
- These concepts can be combined: each shard can contain partitioned tables.

Partitioning alone does not provide the same scale-out capacity as distributing data across database servers.

## 3. What is a sharding key?

A **sharding key** is the field, or combination of fields, used to determine where a record lives. The key should be chosen based on access patterns, data distribution, write/read load, stability, and the need to keep related data together.

Consider a hypothetical multi-tenant document platform:

| document_id | tenant_id | owner_user_id | document_name |
|---:|---:|---:|---|
| 501 | 100 | 21 | Contract A |
| 502 | 200 | 35 | Report B |
| 503 | 100 | 22 | Proposal C |
| 504 | 300 | 40 | Invoice D |
| 505 | 200 | 35 | Report E |

Potential keys include tenant_id, owner_user_id, or document_id. Each leads to a different data layout and different query costs.

## 4. Example: sharding by tenant_id

If records are assigned by tenant_id, records for a tenant are placed on the same shard. For example, documents for tenant 100 can be routed to one shard, tenant 200 to another, and so on. The exact mapping may use a hash or a routing directory; the example is illustrative rather than a fixed assignment.

A request such as “fetch all documents for tenant 100” can be routed to the shard responsible for tenant 100. This creates **data locality**: records commonly accessed together are stored together.

Benefits:
- Tenant-scoped queries can often go to one shard.
- Writes and transactions scoped to a tenant may remain on one shard.
- Queries spanning many tenants may need multiple shards and a merge of the results.

Trade-off:
- A very large or active tenant can overload its shard while other shards remain lightly used.

## 5. What is a hot shard?

A **hot shard** receives disproportionately high traffic or workload compared with other shards. It may have higher CPU use, I/O, lock contention, connection pressure, or query latency while other shards still have spare capacity.

A hot shard can result from:

### Uneven data volume
One tenant might hold tens of millions of records while most other tenants hold only a few thousand. If all data for a tenant stays together, that large tenant may dominate one shard's storage and maintenance workload.

### Uneven request volume
Two shards may contain roughly the same number of rows, but one may receive many more reads and writes. Equal data size does not imply equal workload.

### A hot key or hot range
If new records are continually written to the newest range in a range-based design, writes may concentrate on one shard. Similarly, a single extremely active tenant or user may send a large fraction of requests to one shard.

Adding more shards does not automatically fix a hot shard. The workload must be redistributed or the source of concentrated traffic must be addressed.

## 6. How to choose a sharding key

### Rule 1: Match common query patterns
If most requests are tenant-scoped, tenant_id may allow direct routing to one shard. If most are user-scoped, user_id may be more suitable. Queries across many tenants or users can become fan-out queries.

**Fan-out** means sending one logical query to multiple shards and combining their results.

### Rule 2: Consider cardinality, but do not rely on it alone
A key with many distinct values offers many candidates for distribution. A low-cardinality field such as status, with only values like ACTIVE, PENDING, and CLOSED, is often a poor choice because it can concentrate whole groups of records together.

High cardinality alone is not sufficient: one distinct key can still generate a disproportionate amount of traffic.

### Rule 3: Consider actual workload and skew
Estimate data volume per key, requests per second, read/write mix, query cost, and traffic concentration. A design that distributes the number of records evenly may still distribute requests badly.

### Rule 4: Prefer a reasonably stable key
Changing a record's sharding key may require moving it to a different shard and updating routing. Frequently changing keys can add significant complexity.

### Rule 5: Keep commonly related records together when useful
Keeping all records for one tenant together can simplify tenant-wide operations. But it also means a very large tenant can become a hot shard. Keeping records together is a trade-off, not an absolute rule.

There is no universally best shard key. Pick one that fits the expected workload and understand the operations it makes expensive.

## 7. Compare tenant_id versus user_id

### Design A: Shard by tenant_id
- Advantage: tenant-wide reads and related operations can often stay on one shard.
- Disadvantage: one enormous tenant may overwhelm its shard.
- Good candidate when most access is tenant-scoped and tenant sizes and workloads are manageable.

### Design B: Shard by user_id
- Advantage: users from one very large tenant can be spread across multiple shards.
- Disadvantage: a tenant-wide query may need to visit many shards and merge results.
- Good candidate when requests are primarily user-scoped and spreading a large tenant's load matters more than tenant-wide locality.

If a few tenants are exceptionally large, the system may need a hybrid placement policy rather than one uniform rule for every tenant.

## 8. Strategies for handling a hot shard

### Option A: Give a large tenant its own shard
Keep ordinary tenants on shared shards but isolate unusually large or active tenants on dedicated shards. A routing directory can map tenant IDs to shard IDs.

This can isolate heavy tenants, but it requires data migration, routing changes, and operational support for dedicated placements.

### Option B: Split one tenant into buckets
If one tenant is too large or active for one shard, distribute that tenant's records using a secondary key. For example, a conceptual bucket formula is:

    bucket = hash(document_id) % 4

The tenant's documents could then be spread across four buckets. This distributes load but means a query for all documents of that tenant may have to query all four buckets. Bucket routing and future rebalancing must be designed carefully.

### Option C: Cache repeated reads
Caching can reduce load when the hot shard is dominated by repeated reads of the same data. It does not, by itself, solve a write hotspot and introduces invalidation and freshness concerns.

### Option D: Rebalance data
Move data from an overloaded shard to less-loaded shards and update the routing scheme. Rebalancing helps when placement is the cause, but it cannot solve a single hot key if all its requests still go to one destination.

The correct remedy depends on whether the bottleneck is one tenant, one hot key, a hot write range, or the general workload.

## 9. Costs and complexity introduced by sharding

| Concern | Why it becomes harder |
|---|---|
| Cross-shard queries | Multiple shards must be queried and results combined. |
| Transactions | Atomic changes across multiple shards require more coordination or application-level design. |
| Unique constraints | A local unique constraint does not automatically enforce uniqueness across every shard. |
| Joins | Related rows on separate shards may require distributed query support or application-level aggregation. |
| Resharding | Moving records needs migration, routing changes, and consistency planning. |
| Operations | Capacity, replication, availability, monitoring, and failures must be managed per shard. |

These costs are why sharding should not be introduced merely because a table is large. Indexes, partitioning, vertical scaling, and read replicas may be simpler and sufficient.

## 10. A practical decision process

1. Measure the bottleneck: determine whether the issue is storage capacity, write throughput, read traffic, CPU, or a specific slow query.
2. Optimize the existing database first: inspect query plans, fix inefficient SQL, and create workload-appropriate indexes.
3. Consider partitioning if it allows query pruning or makes retention and maintenance easier.
4. Consider read replicas if read capacity is the main problem and replica consistency is acceptable for the workload.
5. Consider sharding when a single database remains insufficient and the workload can be distributed across independent servers.
6. Choose a key using real access patterns and workload distribution. Estimate skew and identify likely hot keys before migrating data.
7. Design routing, rebalancing, cross-shard operations, uniqueness, and failure handling as first-class parts of the system.

## 11. Interview summary

**Sharding distributes data across database servers; the sharding key decides where records live.** A good key matches common access patterns, keeps related data together when useful, and distributes workload reasonably well. A hot shard is an overloaded shard caused by skewed data, concentrated traffic, or hot keys and ranges. Possible remedies include dedicated shards, sub-sharding, caching for repeated reads, and rebalancing. Sharding increases capacity but complicates cross-shard queries, transactions, uniqueness, migration, and operations, so it should follow measurement and simpler optimizations.
