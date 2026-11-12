# CAP Theorem — Consistency, Availability, and Partition Tolerance

## 1. Why CAP matters

Distributed systems often replicate data across multiple nodes or regions. Replication improves resilience and can reduce read latency, but replicas need to exchange updates.

Imagine two database replicas:

    USA replica:    display_name = "Tanay"
    Europe replica: display_name = "Tanay"

A user changes the name to "Tanay Kumar" in the USA. Before the update reaches Europe, the network link between the regions fails.

    USA replica                         Europe replica
    display_name = "Tanay Kumar"        display_name = "Tanay"
              |                                  |
              +--------------- X ----------------+
                         Network partition

Both databases are still running, but they cannot communicate. A read request reaches Europe. Europe cannot know whether its value is still the latest.

It now faces a choice:
- Return the value it has ("Tanay"), keeping the request available but possibly returning stale data.
- Refuse or delay the request until it can establish the latest value, preserving strong consistency but sacrificing availability for that request.

This is the central problem described by CAP.

## 2. The three properties

### C — Consistency

In CAP, consistency means **strong consistency in the linearizability sense**: every successful read must return the most recent completed write, or a newer value. Operations must appear to happen in one global order that respects real-time ordering.

Example:
- A write changes the profile name to "Tanay Kumar" and completes.
- A later read must not return "Tanay" as though it were current.
- If a node cannot establish the latest value, it must not return a stale value as a successful strongly consistent read.

**Do not confuse CAP consistency with ACID consistency.**

| ACID consistency | CAP consistency |
|---|---|
| A transaction preserves database constraints and invariants, moving the database from one valid state to another. | Reads and writes follow a strong, linearizable view across replicas. |
| Example: an account balance cannot violate a required database constraint. | Example: after a completed balance update, a subsequent read cannot return the old balance as current. |

### A — Availability

In CAP, availability means every request received by a non-failing node must receive a successful response. The response may contain stale data.

It is stronger and more specific than saying "the service has good uptime":
- The node receiving the request is operational.
- It must return a successful response rather than reject the request because another node is unreachable.
- A stale result can satisfy availability, but it does not satisfy strong consistency.

### P — Partition Tolerance

A network partition occurs when nodes cannot communicate because messages are lost, delayed indefinitely, or a network link is broken.

Partition tolerance means the system is designed to continue operating in the presence of this communication failure.

A partition is not the same as a database process crashing. The nodes may all be healthy individually while being unable to exchange messages.

## 3. What CAP actually says

The simplified textbook phrase is:

> A distributed system can guarantee at most two of Consistency, Availability, and Partition Tolerance.

The more useful interpretation is:

**When a network partition occurs, a distributed system must choose between strong consistency and availability for operations affected by that partition.**

Why?
1. The replicas cannot exchange information.
2. One side may have accepted a write that the other side has not received.
3. A read on the isolated side cannot both return a successful response and guarantee that it has the latest globally completed value.
4. It must either reject/delay the request or risk returning stale data (or accept independent writes that may conflict).

Partition tolerance is generally not treated as an optional property in a distributed system: networks can fail, and the architecture must define what happens when they do.

## 4. CP — Consistency + Partition Tolerance

A CP-oriented system preserves strong consistency during a partition, even if some requests must be rejected or delayed.

Example: one seat remains on a ticketed event.

    Initial state: Seat 6A = AVAILABLE

    USA side:       Customer A attempts to book
    Europe side:    Customer B attempts to book

    Network between the sides is partitioned.

If both sides independently confirm the seat, the system could double-book it. A consistency-oriented design refuses or delays an operation that it cannot safely coordinate.

Typical behavior:
- The side that can safely establish the authoritative state may continue serving some operations.
- An isolated side may reject or delay reads/writes that cannot be safely completed.
- The system avoids confirming conflicting outcomes.

Good fit when an incorrect or conflicting result is worse than a temporarily unavailable operation:
- Seat or appointment reservations.
- Inventory reservation and allocation.
- Financial transfers or balances where stale state could cause an incorrect decision.
- Uniqueness-sensitive operations.

**Mental model:** A failed or delayed operation is preferable to a successful operation that violates a critical correctness requirement.

CP does not mean the entire application shuts down. It means availability is sacrificed for the requests that cannot safely be served during the partition.

## 5. AP — Availability + Partition Tolerance

An AP-oriented system continues responding during a partition, even if some responses are temporarily stale or conflicting.

Example: a user changes their profile picture in the USA.

    USA replica:    New picture
    Europe replica: Old picture

Europe cannot receive the update yet, but it continues to serve profile reads using the value it currently has. After the network recovers and replication/reconciliation completes, the replicas converge.

Good fit when a temporarily stale result is acceptable:
- Profile pictures and profile display information.
- View/like counts.
- Search indexes that can catch up asynchronously.
- Analytics dashboards and other eventually updated information.
- Content descriptions where a short delay is not harmful.

**Mental model:** Continue serving a useful response, even if the data is not perfectly up to date.

AP does not mean data stays inconsistent forever. An eventually consistent design aims to reconcile changes and converge after communication is restored. The reconciliation strategy matters, especially if both sides accepted conflicting writes.

## 6. CP vs. AP comparison

| Question | CP-oriented behavior | AP-oriented behavior |
|---|---|---|
| What happens during a partition? | Rejects or delays operations that cannot be safely coordinated | Continues serving requests from available nodes |
| Can a read return stale data as a successful response? | Not for an operation promising strong consistency | Yes |
| Can both sides accept independent conflicting writes? | Not if doing so would violate the consistency guarantee | May be allowed, with later conflict resolution |
| Main priority | Correctness of the observed state | Successful responses and continued operation |
| Typical example | Booking the last seat | Showing a possibly outdated profile picture |

These labels describe behavior under a partition, not a permanent label that should be attached to every feature of a database product.

## 7. Why not CA?

CA means Consistency + Availability without a partition-tolerance guarantee.

A system may provide both consistency and availability while its nodes can communicate normally. But if it is distributed and a network partition occurs, it cannot guarantee both:
- Returning the old value preserves availability but breaks strong consistency.
- Refusing/delaying the request preserves strong consistency but breaks availability.

Therefore, CA is not a viable guarantee for a distributed system that must continue handling network partitions.

A single-node database can avoid inter-replica partitions, but it does not provide the same fault tolerance as a distributed system. If that node fails, the service can become unavailable.

## 8. CAP choices can differ by operation

A whole application does not necessarily need one uniform consistency/availability choice. Different operations have different correctness requirements.

Example: a ticket-booking platform.

| Operation | Likely priority | Reason |
|---|---|---|
| Confirm a seat booking | CP-oriented | Avoid confirming the same seat twice |
| Browse event descriptions | AP-oriented | A slightly old description is usually acceptable |
| Read authoritative seat allocation | Consistency-oriented | A stale result could lead to an incorrect booking decision |
| View promotional content | Availability-oriented | A temporary stale response is usually harmless |

Example: a social platform.

| Operation | Likely priority | Reason |
|---|---|---|
| Establish a unique match or relationship | Consistency-oriented | Avoid conflicting relationship state |
| View a profile picture | Availability-oriented | A previous picture can be shown briefly |
| View a like count | Availability-oriented | A small temporary discrepancy is acceptable |

The important design question is:

**Would it be harmful if this particular operation returned stale or conflicting data?**

Choose guarantees based on the operation's business requirements, not simply on the application name.

## 9. Architectural approaches associated with the choices

### Consistency-oriented approaches

- **Single authoritative leader/source for writes:** route relevant writes through one authority so competing nodes do not independently accept conflicting updates. The leader itself must be protected by a safe failover/election mechanism.
- **Coordination or consensus:** nodes coordinate to agree on an order or committed value. During a partition, nodes that cannot reach the required quorum may have to reject or delay operations.
- **Distributed transactions:** protocols such as two-phase commit coordinate a transaction across participating resources. They add coordination and failure-handling complexity; they are not automatically required for ordinary database-plus-cache updates.

### Availability-oriented approaches

- **Asynchronous replication:** a primary acknowledges a write before every replica has necessarily applied it. Replicas can serve reads, but those reads may be stale.
- **Independent writes and reconciliation:** if multiple sides accept writes during a partition, the system needs a way to reconcile conflicting changes after connectivity returns.
- **Change Data Capture (CDC):** captures database changes and propagates them asynchronously to downstream systems such as caches, search indexes, or analytics stores. Consumers may temporarily lag behind the source.

These are design approaches, not absolute technology labels. A product may expose different consistency options depending on operation, topology, and configuration.

## 10. Connection to consistency models

CAP consistency is the strong, linearizable form of consistency. It is not synonymous with every consistency model.

| Consistency guarantee | What it provides | Example |
|---|---|---|
| Strong / linearizable | Reads observe the latest completed write or a newer one | A critical account balance read |
| Causal | Causally related operations are observed in dependency order | A reply is not shown before its parent message |
| Read-your-own-writes | A client sees its own successful writes on subsequent reads | A user updates a profile and immediately sees the new value |
| Eventual | Replicas converge after updates stop and replication progresses | A search index catches up after a document is created |

### Read-your-own-writes example

A user changes their profile picture in the USA. The update has reached the USA replica but not Europe.

The application can route that user's immediate follow-up read to the authoritative replica, so the user sees the new picture. Another user in Europe might still see the old picture.

This offers a useful user experience without requiring every user and every replica to observe every update immediately.

## 11. Recovery after a partition

The behavior after connectivity returns is part of the design.

For a CP-oriented system:
- Nodes reconnect and restore communication.
- Replicas catch up from the authoritative history or committed log.
- Operations that were rejected or timed out may be retried by clients.
- The system resumes normal service once it can safely coordinate.

For an AP-oriented system:
- Replicas exchange updates they missed.
- Conflicting writes, if any, must be reconciled using the system's conflict-resolution policy.
- Replicas converge when updates have been applied and no new conflicting changes remain.

Do not assume that network recovery automatically resolves every conflict correctly. The application/data model needs a reconciliation rule.

## 12. CAP interview approach

When discussing CAP in a system-design interview:

1. **Identify the operation.** Separate critical writes from ordinary reads or display-only data.
2. **State the consistency requirement.** Can this operation tolerate stale data or conflicting writes?
3. **Describe behavior during a partition.** Which requests continue, and which are rejected or delayed?
4. **Explain the data path.** Is there a leader, quorum, independent replicas, or asynchronous replication?
5. **Explain recovery.** How do replicas catch up? How are conflicting writes reconciled?
6. **Apply the choice per operation.** Avoid declaring an entire application simply "CP" or "AP" without explaining the actual guarantees.

Useful interview phrasing:

"During a network partition, I would prioritize strong consistency for operations where stale or conflicting results violate a business invariant, even if that means rejecting or delaying some requests. For display-oriented operations that tolerate stale data, I would prefer continued availability and asynchronous convergence."

## 13. Common misconceptions

| Misconception | Correct understanding |
|---|---|
| CAP means permanently choosing two properties out of three | The central trade-off is between consistency and availability when a partition occurs |
| Partition tolerance is optional | Distributed systems must account for possible network partitions |
| CAP consistency is the same as ACID consistency | CAP refers to strong/linearizable consistency across operations and replicas |
| AP means data remains inconsistent forever | Replicas may converge after communication and reconciliation |
| CP means the whole application goes down | Only operations that cannot safely proceed need to be rejected or delayed |
| Every feature must use the same CAP choice | Different operations can have different correctness requirements |
| A database product is always CP or always AP | Guarantees depend on architecture, configuration, and operation-level settings |
| Availability simply means high uptime | CAP availability is about successful responses from non-failing nodes during a partition |

## 14. PACELC — a useful extension

PACELC extends the CAP discussion:

- **If there is a Partition (P):** a system trades off Availability (A) against Consistency (C).
- **Else (E), when the network is operating normally:** it may still trade off Latency (L) against Consistency (C).

The extension highlights that coordination can cost latency even when there is no partition. This is useful context, but CAP itself remains the main topic here.

## 15. Interview summary

- **C — Consistency:** strong/linearizable behavior; a read observes the latest completed write or a newer one.
- **A — Availability:** every request to a non-failing node receives a successful response, even if the response is stale.
- **P — Partition tolerance:** the system handles communication failures between nodes.
- During a partition, a distributed system must choose between strong consistency and availability for affected operations.
- **CP:** reject or delay operations that cannot be safely coordinated.
- **AP:** continue responding, possibly with stale or conflicting data; reconcile and converge later.
- **CA:** possible while communication is healthy, but not a partition-tolerant guarantee.
- Make the choice per operation based on the cost of stale or conflicting data.

**Interview question to remember:** Would it be harmful if this operation briefly returned stale or conflicting data? That answer drives the consistency-versus-availability decision.
