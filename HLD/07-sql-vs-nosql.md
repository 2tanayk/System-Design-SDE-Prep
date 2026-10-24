# SQL vs. NoSQL Databases

## 1. The problem: choose a data model that fits the application

A food-delivery backend needs to store users, restaurants, menus, orders, order items, inventory, payments, and reviews. These pieces of data have different relationships and access patterns.

The database decision is not simply “Which one is faster?” It is: **Which data model, integrity guarantees, query capabilities, scaling approach, and operational tradeoffs fit this workload?**

SQL and NoSQL are broad families, not guarantees that one option is always faster, more scalable, or more consistent.

## 2. SQL / relational databases

**SQL** (Structured Query Language) is used to define, query, and manipulate data in relational database systems. PostgreSQL and MySQL are common relational database management systems (RDBMSs).

Relational databases organize data into tables of rows and columns. They often use normalization to reduce unnecessary duplication and represent relationships explicitly.

Example:

**users**

| id | name |
|---:|---|
| 1 | Rahul |
| 2 | Priya |

**orders**

| id | user_id | total |
|---:|---:|---:|
| 101 | 1 | 350 |
| 102 | 2 | 280 |
| 103 | 1 | 220 |

Here, users.id can be the primary key, and orders.user_id can be a foreign key referencing it. A primary key identifies a row; a foreign key constraint helps ensure a reference points to a valid row.

A JOIN lets the application query related information across tables:

~~~sql
SELECT orders.id, users.name, orders.total
FROM orders
JOIN users ON orders.user_id = users.id;
~~~

### Why relational databases are useful

- **Relationships and integrity:** primary keys, foreign keys, unique constraints, checks, and other database constraints can enforce important rules.
- **Transactional operations:** multiple related database changes can be grouped into a transaction.
- **Flexible queries:** SQL supports joins, filtering, grouping, sorting, and aggregation across related data.
- **Explicit data model:** columns and types make the intended structure visible and can catch invalid writes.

A relational schema is often normalized, but normalization is a design choice rather than a requirement that every relational table must be maximally normalized. SQL dialects also vary; SQL does not mean every database supports exactly the same syntax or features.

## 3. NoSQL databases

**NoSQL** is a broad category of data stores that do not rely exclusively on the traditional relational-table model. Different NoSQL systems have very different data models and capabilities.

| Model | Example | Typical fit |
|---|---|---|
| Document | MongoDB | JSON-like documents with nested or variable attributes |
| Key-value | Redis | Key-based lookup, caching, session data, counters |
| Wide-column | Apache Cassandra | Distributed workloads designed around partitioned query patterns |
| Graph | Neo4j | Data where navigating relationships is central |

These are common fits, not exclusive rules.

### Document example

A product catalog may include laptops, clothes, and groceries with different attributes:

~~~json
{
  "name": "Laptop X",
  "category": "electronics",
  "price": 65000,
  "specifications": {
    "ramGB": 16,
    "storageGB": 512
  }
}
~~~

~~~json
{
  "name": "Classic T-Shirt",
  "category": "clothing",
  "price": 799,
  "specifications": {
    "sizes": ["S", "M", "L"],
    "material": "Cotton"
  }
}
~~~

A document database can store category-specific attributes together with each product. This may simplify reads when the main operation fetches or updates a product as a unit.

### “Schemaless” needs qualification

NoSQL systems often permit flexible data structures, but that does not mean data modeling and validation are unnecessary. A document may still require validation rules, and the application must not allow fields to drift into inconsistent shapes accidentally.

SQL databases can also store flexible data. PostgreSQL supports JSON and JSONB columns, and JSONB can be queried and indexed. Therefore, varying attributes do not automatically require MongoDB.

## 4. Transactions and data correctness

A **transaction** groups database operations into a logical unit. ACID describes four commonly desired properties:

- **Atomicity:** changes in a transaction commit together or roll back together.
- **Consistency:** a committed transaction preserves the database rules and constraints that apply to it.
- **Isolation:** concurrent transactions behave according to the chosen isolation level.
- **Durability:** committed changes survive failures covered by the database's durability guarantees.

### Example: order placement

An order workflow may need to create an order, add order items, and decrease inventory. PostgreSQL can perform related database changes inside a transaction, with appropriate constraints and concurrency control.

If the inventory update fails before commit, the application can roll back the order changes as well, provided these operations are included in the same transaction and the failure is handled correctly.

A database transaction does not automatically roll back an external payment provider's HTTP operation. External workflows require additional design, such as idempotency, explicit payment states, retries, and compensation where appropriate.

### Does NoSQL mean “no transactions”?

No. MongoDB supports atomic writes to a single document and also supports multi-document ACID transactions. Embedding related data in one document may make single-document atomic operations sufficient for some use cases, but transactions are available when multiple documents must change atomically.

The transaction model and guarantees depend on the database, configuration, and operation. Do not infer them from the SQL/NoSQL label alone.

## 5. Example A: money transfers — PostgreSQL is the natural starting point

Suppose a bank-like application transfers ₹1,000 from Account A to Account B. The system needs to record the transfer, debit one account, credit the other, and preserve a consistent ledger.

A relational model can represent accounts, transfers, and ledger entries as separate related tables with keys and constraints.

~~~text
BEGIN TRANSACTION
1. Verify sufficient funds under appropriate concurrency control.
2. Debit Account A.
3. Credit Account B.
4. Record the ledger entries.
COMMIT
~~~

PostgreSQL is a natural fit because the core workload emphasizes related records, constraints, transactional changes, and queries over account and ledger history.

MongoDB can also support multi-document transactions. However, if accounts, transfers, and ledger entries are separate collections, relationships and validation need careful design. MongoDB does not enforce cross-collection foreign-key constraints in the same way a relational database can. Embedding mutable account balances in every transfer would duplicate changing data and create correctness problems.

**Decision:** PostgreSQL is the simpler default for this relational, transaction-heavy core. MongoDB is possible, but its document model provides less advantage for the central requirements.

Financial correctness also depends on application logic, idempotency, appropriate isolation/concurrency control, auditing, and failure handling—not on choosing PostgreSQL alone.

## 6. Example B: flexible marketplace catalog — MongoDB may fit naturally

Imagine a marketplace selling electronics, clothing, and groceries. Product attributes vary significantly by category, and the common operation retrieves a product along with its specifications.

A document could store product fields and nested category-specific details together. This can make document reads and updates convenient and reduce the need to combine many tables for the common access pattern.

A highly normalized relational design could require multiple tables for category-specific attributes and joins to assemble the product view. If that structure becomes awkward for the main workload, MongoDB deserves consideration.

But PostgreSQL remains a viable alternative:
- Common fields can remain regular relational columns.
- Variable specifications can be stored in JSONB.
- JSONB can be queried and indexed.
- Relational constraints and transactions remain available for related data.

**Decision:** MongoDB is attractive when documents naturally represent the main unit of access and their structures vary. PostgreSQL with JSONB or a hybrid model may be equally sensible. Choose based on actual queries, validation needs, and update behavior rather than assuming one database cannot handle the other model.

## 7. Horizontal scaling: why distributed SQL can be harder

**Horizontal scaling (scale out)** adds machines or nodes to share work. It is different from vertical scaling, which makes one machine more powerful.

Both SQL and NoSQL systems can scale, but their approaches differ by product and architecture. Some NoSQL databases were designed from the start around partitioned data across multiple nodes; relational systems also support replication, partitioning, and distributed implementations.

### Relational scale-out and sharding friction

**Sharding** splits data across separate database nodes, often by a shard key. It differs from table partitioning inside a single database deployment.

When related rows are on the same shard, local joins and transactions may remain straightforward. When a query joins data spread across shards, it may require network data transfer and coordination, increasing latency and complexity.

Other challenges can include:
- Cross-shard transactions may require distributed commit coordination; two-phase commit is one possible approach, not a universal requirement for every distributed database operation.
- Global uniqueness constraints and centrally ordered IDs can require coordination. UUIDs, partition-aware identifiers, or allocated ID ranges are possible design alternatives.
- Application routing, rebalancing, failover, and uneven data distribution require operational care.

This does **not** mean relational databases cannot scale horizontally. PostgreSQL supports table partitioning and can be used with replication or separate sharding/distributed solutions. These techniques solve different problems and have different limits.

### NoSQL scale-out advantages—and costs

Some NoSQL systems are designed around partition keys and distributed ownership of records. If a query can be routed to the node responsible for its key, it may avoid cross-node coordination.

Documents can also embed data that is commonly accessed together, reducing joins for that access pattern. Some systems support automatic partition movement or rebalancing.

But these advantages require good data modeling:
- A poor partition key can create a **hot shard**—one node receives a disproportionate amount of traffic or data.
- Queries that span many partitions can become expensive.
- Duplicated data makes updates and reconciliation harder.
- Rebalancing and failover still need operational management.
- Distributed transactions and stronger consistency may involve coordination and additional latency.

NoSQL does not guarantee linear scaling merely by adding nodes. The database's design, partition key, query patterns, and workload determine the result.

## 8. CAP theorem: a distributed-systems tradeoff, not SQL vs. NoSQL

The **CAP theorem** concerns a replicated system when a network partition prevents some nodes from communicating.

- **Consistency (C), in CAP:** a read sees the latest completed write, or the operation fails rather than returning a conflicting stale value. This is a strong consistency notion, often described in terms of linearizability.
- **Availability (A), in CAP:** every request to a non-failing node receives a response, though the response is not necessarily the latest value.
- **Partition tolerance (P):** the system continues to operate despite loss or delay of communication between groups of nodes.

The practical takeaway is **not** “pick any two at all times.” In a distributed system where partitions must be tolerated, the difficult tradeoff occurs during a partition: a system cannot guarantee both strong consistency and availability for every affected operation. It may reject or delay some operations to preserve consistency, or continue serving requests with a risk of stale/conflicting data.

~~~text
Network partition occurs
          |
          v
Can the system communicate enough
to safely complete this operation?
          |
       +--+--+
       |     |
      Yes    No
       |     |
 Continue    Choose behavior:
 normally    - Reject/wait to preserve
               strong consistency, or
             - Respond despite partition,
               accepting weaker consistency
~~~

CAP is not a classification where all SQL systems are CP and all NoSQL systems are AP. Actual guarantees depend on the distributed architecture, replication protocol, configuration, and operation. A single-node database does not face the same replicated-network partition tradeoff between its own nodes, though the overall application can have network failures.

Also, **ACID consistency and CAP consistency are not the same concept**:
- ACID consistency refers to preserving defined database rules and invariants across transactions.
- CAP consistency concerns how replicated reads and writes behave, especially during network partitions.

## 9. ACID and BASE are not synonyms for SQL and NoSQL

**BASE** is a term commonly associated with some availability-oriented distributed designs:
- **Basically Available:** the system aims to respond despite certain failures.
- **Soft State:** replicas or derived state may change as updates propagate.
- **Eventual Consistency:** if updates stop and the system recovers, replicas are expected to converge under the system's guarantees.

BASE is a useful design vocabulary, not a mandatory property of all NoSQL databases. NoSQL products vary considerably and some offer strong consistency and ACID transactions.

Similarly, SQL databases can be distributed and configured with different replication and availability behaviors. Do not present ACID vs. BASE as a strict division between SQL and NoSQL.

## 10. Practical workload evaluation

| Requirement | What to investigate |
|---|---|
| Many relationships and complex relational queries | SQL joins, constraints, schema, and query capabilities |
| Multi-record transactions | Transaction scope, isolation, constraints, and database guarantees |
| Highly variable document-shaped records | Document model, validation rules, document size, and access pattern |
| High ingestion rate or very large distributed dataset | Write pattern, partition key, hot partitions, replication, and operational limits |
| Many queries across related entities | Join/query flexibility and whether data must be duplicated |
| Low-latency key-based lookups | Key-value stores or suitable indexed access paths |
| Growth beyond one node | Read replicas, partitioning, sharding, routing, and distributed coordination costs |
| High availability during network partitions | Required consistency, failure behavior, and recovery guarantees |

Read-heavy vs. write-heavy alone does not determine the database. Both SQL and NoSQL products can serve read- or write-heavy workloads, depending on design and configuration.

## 11. Hybrid architecture and polyglot persistence

Modern systems often combine storage technologies rather than forcing every workload into one database.

Example food-delivery stack:

- **PostgreSQL:** users, orders, inventory, payment state, and other relational transactional data.
- **Redis:** cache entries, sessions, counters, or other key-oriented workloads where appropriate.
- **Object storage:** restaurant photos and other large media files.
- **Potential document database:** a variable catalog only if its document model meaningfully improves the actual access patterns over PostgreSQL/JSONB.

PostgreSQL's JSONB support allows structured relational data and flexible JSON data to coexist in one database. That can avoid introducing another datastore when flexibility is the only missing feature.

**Polyglot persistence** means using more than one storage technology because different parts of the system have different requirements. It also adds operational costs: more backups, monitoring, failure modes, access controls, and possible data synchronization.

### The practical velocity rule

Start with the technology the team can operate well and that meets current requirements. Do not add a database solely because a future scale problem is hypothetical. Introduce a new data store when its benefits justify its additional complexity.

## 12. Decision checklist

Before choosing SQL or NoSQL, ask:

1. **Data model:** Are relationships central, or is the data naturally document/key/graph shaped?
2. **Access pattern:** Which queries dominate—joins and varied queries, or known key/document access patterns?
3. **Correctness:** What constraints, transactions, and consistency guarantees are required?
4. **Scale:** What actual read/write volume, data size, and distribution are expected?
5. **Failure behavior:** What should happen when replicas cannot communicate?
6. **Operations:** Can the team deploy, monitor, back up, restore, and troubleshoot the chosen database?
7. **Simpler alternative:** Could PostgreSQL JSONB, indexing, caching, replicas, or partitioning meet the requirement without another database?

## 13. Summary card

- **Idea:** SQL models related data in tables and provides relational queries and constraints; NoSQL describes multiple alternative data models such as documents, key-value, wide-column, and graph.
- **Transaction nuance:** SQL is not the only option with ACID transactions, and NoSQL does not automatically mean eventual consistency.
- **Scaling nuance:** Distributed SQL can be sharded; distributed NoSQL can encounter hot partitions, cross-partition queries, and coordination costs.
- **CAP nuance:** The consistency/availability tradeoff applies during network partitions in distributed systems; it is not a SQL-vs-NoSQL triangle.
- **Project connection:** PostgreSQL is a natural starting point for food-delivery orders and inventory; MongoDB may suit a variable document catalog; Redis may suit caching.
- **Interview one-liner:** “I choose based on the data model, query patterns, integrity and transaction guarantees, scale and failure requirements, and operational complexity—not because SQL is always consistent or NoSQL always scales better.”

## Further reading

- [PostgreSQL constraints and foreign keys](https://www.postgresql.org/docs/current/ddl-constraints.html)
- [PostgreSQL table partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [PostgreSQL JSON and JSONB](https://www.postgresql.org/docs/current/datatype-json.html)
- [MongoDB transactions](https://www.mongodb.com/docs/manual/core/transactions/)
- [MongoDB shard-key considerations](https://www.mongodb.com/docs/manual/core/sharding-troubleshooting-shard-keys/)
- [CAP theorem — AWS architecture whitepaper](https://docs.aws.amazon.com/whitepapers/latest/availability-and-beyond-improving-resilience/cap-theorem.html)
