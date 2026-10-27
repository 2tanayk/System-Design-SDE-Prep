# Database Replication

## 1. The problem: one database is handling too much

A food-delivery system may send restaurant reads, menu reads, order-history queries, and writes for orders and inventory to the same PostgreSQL primary. When read traffic becomes high, the primary may spend a large share of its resources serving repetitive reads rather than handling writes.

**Database replication** means maintaining copies of data on multiple database nodes. Replicas can help distribute read traffic, support failover, and improve recovery options. Replication is not a complete backup strategy, and adding replicas does not automatically make an application highly available.

## 2. Primary–replica architecture

In a common **leader-follower** (primary–replica) design:

- **Primary:** The authoritative writer. It accepts writes and produces the changes that replicas follow.
- **Replica / standby / read replica:** Maintains a copy of the primary's data. It may serve suitable read-only queries and may be eligible for promotion during failover.
- **Replication:** The mechanism that transfers changes and applies them to replicas.

~~~text
                    Spring Boot API
                    /             \
             write path          read path
                  |                  |
                  v                  v
          +---------------+   +---------------+
          | Primary DB    |-->| Read replica  |
          | authoritative |   | copy of data  |
          +---------------+   +---------------+
~~~

This is a simplified illustration. Real deployments may have multiple replicas, separate read-routing logic, and replicas in different availability zones or regions.

## 3. How changes are replicated: WAL

In PostgreSQL, replication commonly uses the **Write-Ahead Log (WAL)**.

A WAL is a sequential log of database changes used for recovery and replication. The database records relevant changes in WAL before the corresponding data pages need to be written to their final locations.

A simplified streaming-replication flow:

1. The primary processes a transaction and records the necessary WAL.
2. WAL records are sent from the primary to a standby over the network.
3. The standby receives the records and replays them to update its local database state.
4. Once applied, the corresponding data is visible to reads on that standby, subject to transaction visibility and replication state.

~~~text
Application write
       |
       v
Primary PostgreSQL
       |
       v
WAL records generated
       |
       v
Stream WAL to standby
       |
       v
Standby replays WAL
       |
       v
Replica data catches up
~~~

The replica does not need the primary to copy its entire database after every write; it can process the change log. Database engines use different log terminology and replication mechanisms. PostgreSQL uses WAL; MySQL commonly uses a binary log (binlog).

## 4. Why replicas help reads more readily than writes

Suppose a workload receives 8,000 database operations per second: 7,000 reads and 1,000 writes. These numbers are illustrative.

Appropriate reads can be distributed across replicas, reducing the read workload on the primary. Writes in this leader-follower design still go to the authoritative primary, which orders and coordinates changes before replicating them.

A replica is not an independent writer for the same data. Letting two nodes update shared inventory without proper coordination can create conflicting outcomes. For example, two customers might both buy the last burger if each node independently believes one is available.

This does not mean writes can never scale horizontally. Partitioned or distributed database architectures can distribute write ownership, but those designs introduce other data-routing and coordination tradeoffs.

## 5. Asynchronous, synchronous, and semi-synchronous replication

Replication mode determines what the primary waits for before acknowledging a write. The exact guarantees depend on the database engine and configuration.

### Asynchronous replication

The primary generally acknowledges a committed write without waiting for a replica to confirm receipt or replay of the change. Replication continues in the background.

**Advantages**
- Lower write latency because replica acknowledgment is not on the commit path.
- The primary can often continue accepting writes when a replica is slow or unavailable.

**Tradeoffs**
- Replicas may be stale.
- If the primary fails before recent acknowledged changes reach the replica that is promoted, those changes may be lost from the new primary.
- Lag can grow under network or replica pressure.

This is common in PostgreSQL streaming replication by default.

### Synchronous replication

The primary waits for acknowledgment from a configured synchronous standby according to the selected acknowledgment rule. Depending on the configuration, acknowledgment may mean that WAL has been received, flushed to durable storage, or replayed.

**Advantages**
- Can provide stronger protection against loss of acknowledged changes during particular failover scenarios.
- Can ensure the required standby has received or applied changes to the configured level before the primary confirms completion.

**Tradeoffs**
- Adds network and replica acknowledgment latency to writes.
- A slow or unavailable required standby can delay or block commits, depending on configuration.
- “Synchronous” alone does not promise zero data loss in every imaginable failure scenario; the acknowledgment rule, number of standbys, failure independence, and failover procedure matter.

### Semi-synchronous replication

Some database systems offer a semi-synchronous mode in which the primary waits for acknowledgment from at least one replica while other replicas may continue asynchronously. Exact behavior and fallback rules are product-specific.

**Purpose:** balance some additional protection against data loss with lower latency than a design that waits for all required replicas to apply every change.

Do not assume every database supports the same semi-synchronous mode. In PostgreSQL, understand its actual synchronous-standby configuration rather than treating “semi-sync” as a universal setting.

| Consideration | Asynchronous | Synchronous / semi-synchronous, as configured |
|---|---|---|
| Primary waits for acknowledgment | Generally not for replica confirmation | Yes, from configured replica(s) |
| Commit latency | Usually lower | Usually higher |
| Stale reads | Possible | Depends on when and where replicas are read |
| Potential recent-write loss on failover | Possible if changes were not replicated to the promoted node | Reduced according to the acknowledgment and failover guarantees |
| Dependence on replica/network health | Less direct for write acknowledgment | More direct |

## 6. Replication lag and stale reads

**Replication lag** is the delay between a change being committed on the primary and reaching the relevant visibility point on a replica. This may be measured by time or by log position.

Common contributors:
1. **Network latency or limited bandwidth:** Replication data takes longer to transfer.
2. **Replica resource bottlenecks:** The replica may have insufficient CPU, memory, or disk throughput.
3. **Large transactions:** Large changes can generate substantial WAL that takes time to transfer or replay.
4. **Apply/replay throughput:** A replica may receive changes faster than it can apply them.

Example: a customer places an order. The primary commits order 101, but the replica has not applied that transaction yet. If the application immediately reads order history from the replica, order 101 may be missing. This is a **stale read**: the result does not yet reflect a recent relevant write.

## 7. Mitigating stale reads and handling read-your-own-writes

**Read-your-own-writes consistency** means that after a client successfully writes data, a subsequent read by that client reflects the write.

Possible strategies:

### A. Route a short period of reads to the primary

After a write, route that user's relevant follow-up reads to the primary for a defined period (for example, several seconds). This is a pragmatic heuristic, not a mathematical guarantee: replication lag can exceed the chosen period.

### B. Track a WAL Log Sequence Number (LSN)

A **Log Sequence Number (LSN)** identifies a position in PostgreSQL's WAL stream.

A system can record the relevant commit LSN and direct subsequent reads only to a replica that has replayed at least that position. Otherwise, route to the primary or wait according to the latency policy.

This requires correct LSN tracking and a way to determine replica replay progress.

### C. Route using lag and health metrics

Monitor replica lag and availability. Route freshness-sensitive reads to the primary or to replicas that meet the required freshness threshold. If no replica qualifies, the application may wait, fall back to the primary, or return a controlled failure depending on the product requirement.

### D. Session stickiness (with caution)

Pinning a session to a particular replica can provide a more stable source during an interaction, but **sticky sessions alone do not guarantee read-your-own-writes**: the chosen replica can still be behind. Use it only when its weaker guarantee is acceptable or combine it with explicit lag/LSN-aware routing.

## 8. Which reads should go to replicas?

| Use case | Possible routing |
|---|---|
| Popular restaurant listings | Replica if brief staleness is acceptable |
| Menu browsing | Replica if freshness requirements permit it |
| Historical reports | Replica if the report can tolerate replication delay |
| Immediately viewing a newly placed order | Primary or a replica with an appropriate freshness guarantee |
| Authoritative inventory decision | Use the primary/write path and correct concurrency control in this leader-follower design |

Do not route every HTTP GET to a replica without considering its business semantics. “Read” does not automatically mean “staleness is acceptable.”

## 9. Failover, availability, and backups

**Failover** means switching the workload to another node when the current primary becomes unavailable. A replica may be promoted, and application connections or routing must be redirected.

Promotion requires care:
- Determine that the primary is actually unavailable.
- Avoid two nodes both acting as writable primary (**split brain**).
- Understand whether acknowledged changes had reached the replica.
- Reconfigure routing and reconnect clients.
- Monitor the promoted primary and rebuild replicas as needed.

Replication may support high availability and recovery, but it is not a substitute for backups. Replication can copy accidental deletions or corrupted data to replicas. Backups and tested restore procedures address different failure scenarios.

## 10. Geo-distributed replication

Replicas may live in different availability zones or regions. Replication across distant regions incurs unavoidable network latency and may be affected by bandwidth limits.

A common design keeps a primary in one region and asynchronous read replicas in other regions to serve local reads with lower latency. The tradeoff is that those local replicas may be stale. Writes may still need to reach the primary or be coordinated by a more complex distributed-write design.

Do not assume a distant replica can deliver both local latency and immediate global visibility of every write without coordination.

## 11. Decision checklist

Before introducing replication, ask:

1. **Goal:** Are we reducing read pressure, preparing for failover, or improving regional read latency?
2. **Read freshness:** Which requests can tolerate stale data?
3. **Write guarantees:** What acknowledged-write loss is acceptable in a failure?
4. **Mode:** Is asynchronous replication enough, or do we need synchronous acknowledgment?
5. **Routing:** How will the application choose a replica and handle lag?
6. **Failure:** How is primary promotion performed, and how do we prevent split brain?
7. **Recovery:** Do we also have backups and tested restore procedures?

## 12. Summary card

- **Idea:** Replication maintains copies of database data on multiple nodes.
- **Mechanics:** PostgreSQL streaming replication sends WAL records from primary to standby, which replays them.
- **Tradeoff:** Asynchronous replication generally has lower write latency but can lose recent unreplicated writes on failover; synchronous acknowledgment adds latency and has configuration-specific guarantees.
- **Watch-out:** Replication lag can cause stale reads. Use primary routing, lag-aware routing, or LSN-aware read-your-own-writes handling when required.
- **Project connection:** Use PostgreSQL replicas for suitable restaurant/menu reads while routing freshness-sensitive order reads and writes appropriately.
- **Interview one-liner:** “I’d use primary–replica replication to offload eligible reads or support failover, and choose sync mode, read routing, and recovery procedures based on consistency, latency, availability, and tolerated data-loss requirements.”
