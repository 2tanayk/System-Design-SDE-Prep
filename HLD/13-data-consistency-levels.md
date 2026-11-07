# Data Consistency and Its Levels — Strong, Eventual, Causal

## 1. Why consistency models matter

In a distributed database, data may be replicated across multiple nodes or regions. A write can reach one replica before the others have received and applied it.

Example: three regions initially store the document name "Project Proposal". A user updates the value in Region A to "Final Project Proposal". Before replication catches up, a read from Region B may still return "Project Proposal".

This temporary difference is commonly caused by replication lag.

**A consistency model describes the guarantees clients receive about the values and ordering of reads and writes across replicas.** In practical terms: after one user updates something, what can other users expect to see, and when?

## 2. Do not confuse distributed consistency with ACID consistency

These are different concepts:

- **ACID consistency:** a transaction takes the database from one valid state to another, preserving constraints and invariants.
- **Distributed consistency:** describes what values and operation ordering clients are allowed to observe across replicas.

This topic focuses on distributed consistency.

## 3. Strong consistency

### Meaning

For this lesson, strong consistency refers to the strict form called **linearizability**.

A successful read that occurs after a write has completed must observe that write or a newer write. Operations must appear to occur in a single global order that respects real-time ordering.

### Example: rename a document

Initial value on all replicas:

    Region A: Project Proposal
    Region B: Project Proposal
    Region C: Project Proposal

A user renames the document in Region A:

    Project Proposal -> Final Project Proposal

The database coordinates the write according to its consistency protocol and acknowledges success. A subsequent strongly consistent read in Region B cannot return the old name. It must return the new name or a still newer value.

Depending on the database, this may involve waiting for replicas, routing the read to an authoritative node, or coordinating with other nodes.

### Concurrent writes

Two users concurrently rename the same document:

    User A: Project Proposal -> Final Proposal
    User B: Project Proposal -> Approved Proposal

The system must establish one globally consistent ordering for the operations. The final value depends on the chosen order or conflict/transaction rules, but clients cannot observe contradictory histories that violate linearizability.

### Trade-offs

- Coordination can add latency, especially across geographical regions.
- During some network failures, a node may need to reject or delay operations it cannot safely coordinate.
- Strong consistency is useful when stale reads could violate important correctness requirements.

### Typical use cases

- Financial balances when a read must reflect completed writes.
- Inventory reservations where overselling must be prevented.
- Permission revocation when subsequent checks must not use an old permission state.
- Critical configuration that must be read from an authoritative committed state.

**Mental model:** a globally agreed, real-time-consistent view takes priority over serving a potentially stale result.

## 4. Eventual consistency

### Meaning

Replicas may temporarily disagree, but if no new updates occur and the system continues to make progress, they eventually converge to the same value.

Eventual consistency does not promise:
- every read returns the latest value;
- replicas update immediately;
- convergence occurs within a fixed time bound.

### Example: document view count

Initially:

    Region A: 100
    Region B: 100
    Region C: 100

A view is recorded in Region A:

    Region A: 101
    Region B: 100
    Region C: 100

A read in Region B may temporarily return 100. Later, replication catches up:

    Region A: 101
    Region B: 101
    Region C: 101

The replicas have converged.

### Concurrent updates and conflict resolution

Suppose two replicas independently update a replicated counter. Simply overwriting one value with another could lose an update. An eventually consistent system still needs a suitable reconciliation or conflict-resolution strategy.

For a distributed counter, a design that merges increments can be more appropriate than independently overwriting a plain integer.

### Trade-offs

- Nearby replicas may serve reads without coordinating globally, often reducing latency.
- The system may continue serving requests during replication delays.
- Applications must tolerate stale reads and temporarily different values.
- Concurrent updates need a conflict-resolution strategy.

### Typical use cases

- View and like counts where a temporary discrepancy is acceptable.
- Search indexes where newly created documents may take time to appear.
- Analytics dashboards with asynchronously updated metrics.
- Profile images or other content that can take time to propagate.

**Mental model:** replicas do not need to agree immediately; they converge after updates stop and replication makes progress.

## 5. Causal consistency

### Meaning

Causal consistency preserves the ordering of causally related operations.

If operation B causally depends on operation A, every client that observes B must also observe A before B.

It does not require all unrelated concurrent operations to be observed in one global order.

### Example: document and comment

1. Alice creates document 501.
2. Bob opens document 501.
3. Bob adds a comment to document 501.

The comment depends on the document existing. A replica must not expose Bob's comment to a client before that client can also observe the document creation.

Even if the comment reaches a replica before the document, the replica must delay exposing the comment until its causal dependency is available.

### Unrelated concurrent operations

Suppose Alice creates a document while Charlie independently updates his profile picture. There is no causal relationship between these operations.

One replica may observe:

    Create document -> Update profile picture

Another may observe:

    Update profile picture -> Create document

Both orders can be acceptable under causal consistency.

However, if someone comments on Alice's document, the document creation must be observed before that comment.

### Another example: replies

A discussion contains:

    Message 1: "We should upgrade the database."
    Message 2: "Agreed, let's do that."

If Message 2 is a reply to Message 1, a client that sees the reply must also have seen the original message first. Unrelated messages do not necessarily need a single global order.

### Trade-offs and use cases

Causal consistency can preserve useful user-visible ordering without imposing a single global order on every unrelated operation. Its implementation may still require metadata, dependency tracking, coordination, or waiting for missing dependencies.

Typical use cases include:
- collaborative applications;
- threaded discussions and replies;
- distributed activity feeds;
- workflows where a user's own actions and their consequences should appear in order.

**Mental model:** related operations preserve logical order; unrelated concurrent operations need not share one global order.

## 6. Compare the three models

| Property | Strong / linearizable | Eventual | Causal |
|---|---|---|---|
| Must a read after a completed write observe it or a newer write? | Yes, for linearizable reads | Not guaranteed | Not necessarily |
| Can clients temporarily see stale values? | No, for strongly consistent reads | Yes | Yes, provided causal dependencies are preserved |
| Preserves causal ordering? | Yes, through its global ordering | Not necessarily | Yes |
| Requires one global order for unrelated concurrent operations? | Yes, a linearizable order | No | No |
| Coordination overhead | Generally higher | Can be lower | Depends on implementation |
| Example | Critical balance or inventory state | Analytics or view counts | Replies and collaborative workflows |

Causal consistency is not simply a halfway point between strong and eventual consistency. It provides an explicit ordering guarantee for causally related operations that eventual consistency alone does not provide.

It also does not promise that every unrelated read is up to date.

## 7. How consistency relates to availability

Availability means the system can continue responding to requests.

Consider a network partition: Region A and Region B are running, but they cannot communicate.

The system faces a difficult choice:

### Prioritize strong consistency

The two regions cannot independently accept potentially conflicting writes while guaranteeing one globally consistent history. Some operations may need to be rejected or delayed until safe coordination is possible.

### Prioritize availability

Both regions may continue accepting writes independently. They can temporarily contain conflicting or different values, which need reconciliation after communication is restored.

This is a preview of the CAP theorem, which is a later topic in the roadmap.

Important clarification:
- Strong consistency and availability are not mutually exclusive during normal operation.
- The difficult trade-off arises during a network partition when coordination is unavailable.
- Eventual consistency and availability are different properties; one does not automatically imply the other.

## 8. Connection to database replication

Synchronous and asynchronous replication describe how changes are propagated and when the primary acknowledges writes. Consistency models describe what clients are allowed to observe.

| Replication mechanism | What it describes |
|---|---|
| Synchronous replication | The primary waits for configured replica acknowledgements before completing a write. |
| Asynchronous replication | The primary acknowledges before replicas necessarily receive or apply the change. |
| Consistency model | The guarantees clients receive when reading and writing data. |

These are related but not interchangeable. Asynchronous replication can permit stale reads from replicas, but the application's read routing and coordination strategy also matter.

**Remember:** replication describes how data moves; consistency describes what clients are allowed to observe.

## 9. Interview summary

- **Strong consistency:** a read after a completed write observes that write or a newer one; linearizability also requires a single operation order that respects real time.
- **Eventual consistency:** replicas can temporarily disagree but converge if updates stop and replication progresses.
- **Causal consistency:** operations that depend on one another must be observed in the correct order; unrelated concurrent operations need not have a global order.
- Distributed consistency is different from ACID consistency.
- Replication mechanisms and consistency models are related but describe different things.
- During a network partition, maintaining strong consistency can require rejecting or delaying operations that cannot be safely coordinated.

**Interview tip:** Choose consistency guarantees based on the correctness requirements of each operation, rather than choosing the strongest model everywhere by default.
