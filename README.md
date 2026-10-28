# System-Design-SDE-Prep

System design notes for SDE preparation, organized in the order of the study roadmap. The track covers **High-Level Design (HLD)** first, then **Low-Level Design (LLD)**. Notes focus on intuition, dry runs, practical examples, tradeoffs, and interview-ready summaries.

## Contents

### Part 1 — High-Level Design (HLD)

#### Week 1 — Fundamentals + Databases
- [Serverless vs. Serverful](HLD/01-serverless-vs-serverful.md)
- [Horizontal vs. Vertical Scaling](HLD/02-horizontal-vs-vertical-scaling.md)
- [Application & Database Optimizations Before Scaling](HLD/03-application-database-optimizations.md) *(supplementary topic)*
- [Threads — OS-Level Fundamentals](HLD/04-threads.md)
- [Pages, Virtual Memory & JVM Memory Basics](HLD/05-pages-and-memory-paging.md)
- [How the Internet Works — DNS, TCP, TLS, HTTP](HLD/06-how-the-internet-works.md)
- [SQL vs. NoSQL Databases](HLD/07-sql-vs-nosql.md)
- [In-Memory vs. Disk-Based Databases](HLD/08-in-memory-vs-disk-based-databases.md)
- [Database Replication](HLD/09-data-replication-and-migration.md)
- [Data Migration & Schema Version Control](HLD/10-database-migrations-and-schema-version-control.md)
- [Data Partitioning — range and hash; indexes vs. other strategies](HLD/11-data-partitioning-and-choosing-indexes.md)
- Sharding — key selection and hot-shard problem

#### Week 2 — Consistency, Availability + Cache
- Data Consistency Levels — strong, eventual, causal
- Transaction Isolation Levels
- CAP Theorem
- Caching — Redis/Memcached and use cases
- Cache Write Policies — write-back, write-through, write-around
- Cache Replacement Policies — LFU, LRU, Segmented LRU
- Content Delivery Networks (CDNs)

#### Week 3 — Networking + Load Balancers
- TCP vs. UDP
- HTTP/1.1, HTTP/2, HTTP/3 and HTTPS
- WebSockets
- WebRTC and Video Streaming
- Load-Balancing Algorithms — round robin, least connections, IP hash
- Consistent Hashing
- Proxies vs. Reverse Proxies
- Rate Limiting — token bucket and sliding window

#### Week 4 — Message Queues + Monoliths vs. Microservices
- Asynchronous Processing — Kafka, RabbitMQ
- Publisher–Subscriber Model
- Monoliths vs. Microservices
- Single Points of Failure
- Avoiding Cascading Failures — timeouts, circuit breakers, bulkheads
- Containerization — Docker images vs. containers
- Strangler Fig Migration Pattern

#### Week 5 — Monitoring, Security + Tradeoffs
- Structured Logging and Monitoring Metrics
- Anomaly Detection
- JWT — access tokens vs. refresh tokens; JWT vs. sessions
- SSO and OAuth
- Encryption — at rest vs. in transit; symmetric vs. asymmetric
- Push vs. Pull Architecture
- Consistency vs. Availability
- SQL vs. NoSQL — revisit
- Memory vs. Latency
- Throughput vs. Latency
- Accuracy vs. Latency

### Part 2 — Low-Level Design (LLD)

#### Week 6 — OOP Fundamentals + Design Patterns
- OOP Principles — encapsulation, abstraction, inheritance, polymorphism
- SOLID Principles
- Creational Patterns — Singleton, Factory
- Structural Patterns — Proxy, Bridge
- Behavioral Patterns — Strategy, Command, Observer

#### Week 7 — Concurrency + UML + API Design
- Thread Safety and Spring Singleton Beans
- Pessimistic vs. Optimistic Locking
- Producer–Consumer Pattern
- Race Conditions and Synchronization — synchronized, volatile
- UML Class Diagrams
- UML Sequence Diagrams
- API Design, DTOs vs. Entities, Versioning
- Clean Code — DRY, SRP, avoiding God classes

#### Weeks 8–9 — Practice Problems
1. Tic-tac-toe or Chess
2. Splitwise App
3. Parking Lot
4. Elevator System
5. Notification System
6. Food Delivery App
7. Movie Ticket Booking System
8. URL Shortener
9. Logging Framework
10. Rate Limiter

---

## How to study each topic

For each topic, understand the problem it solves, build the core idea from first principles, dry-run a concrete example, compare alternatives and tradeoffs, connect it to real systems, and finish with an interview-ready summary. Notes are added after the topic has been discussed and understood.
