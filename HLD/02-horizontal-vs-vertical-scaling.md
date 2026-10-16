# Horizontal vs. Vertical Scaling

## 1. The problem this solves

As an application becomes popular, one server may no longer have enough CPU, memory, or other resources to handle incoming requests while meeting latency targets. Scaling means increasing or decreasing the computing capacity available to the workload.

There are two fundamental approaches:
- **Vertical scaling (scale up/down):** Increase or reduce the resources assigned to one machine.
- **Horizontal scaling (scale out/in):** Add or remove machines or application instances that share the workload.

Scaling is not automatically the first fix. First identify what is limiting the system; more capacity may not fix an inefficient query, a slow dependency, or a database bottleneck.

## 2. Vertical scaling — make one machine more powerful

Vertical scaling increases resources on one machine, such as CPU cores or RAM.

Example:
- Before: one VM with 2 CPU cores and 4 GB RAM.
- After: one VM with 8 CPU cores and 32 GB RAM.
- The application still runs on one machine; that machine has more resources.

**Choose vertical scaling when** a workload fits on one machine, you want to keep the architecture simple, and a larger machine can meet the capacity requirement at a reasonable cost.

**Tradeoffs and limits:**
- Upgrading can be straightforward and may require few application changes.
- A machine has hardware and platform limits; it cannot be made larger forever.
- A single-machine deployment can be a single point of failure unless there is redundancy elsewhere.
- A larger machine does not fix the wrong bottleneck. More application CPU will not automatically make a slow PostgreSQL query faster.

## 3. Horizontal scaling — add more machines

Horizontal scaling adds or removes application instances so that they can process workload together.

A typical setup is:

```text
                  Users
                    |
                    v
              Load Balancer
               /    |    \
              v     v     v
          Server A Server B Server C
               \    |    /
                    v
                PostgreSQL
```

A **load balancer** distributes incoming requests across available application instances. Each **instance** is one running copy of the application.

For example, assume one instance can handle about 200 requests per second for the current workload. Four instances have a simplified aggregate capacity of:

`4 × 200 = 800 requests/second`

This is an illustration, not a guarantee. Real capacity depends on request mix, headroom, failures, shared dependencies, and the latency target. Do not plan to run every component continuously at its theoretical limit.

**Choose horizontal scaling when** the workload needs more capacity than one practical machine can provide, or when multiple instances are useful for redundancy and availability.

**Tradeoffs:**
- Can grow by adding instances, but the whole system remains limited by shared resources such as the database.
- Requires routing, health checks, deployment coordination, and monitoring across instances.
- Shared state and concurrent operations need careful design.
- More instances do not guarantee availability if the load balancer, database, network, or another critical dependency remains a single point of failure.

## 4. Stateless services are easier to scale horizontally

A **stateless service** does not depend on information stored exclusively in one instance's local memory to process a later request. It can process a suitable request on any healthy instance, given the same configuration and access to required dependencies.

For example, a restaurant-details request can go to Server A and the next request can go to Server B if both servers can retrieve the restaurant from a shared database.

A **stateful service** may depend on data retained in a particular instance's local memory. If Server A stores a user's cart only in its memory and the next request goes to Server B, Server B cannot see that cart.

Common approaches:
- Store durable shared state in an appropriate database.
- Use a shared data store such as Redis when its semantics fit the use case.
- Use session affinity (sticky sessions) to route a user's requests to the same instance when justified.

Session affinity has limitations: load can become uneven, and an instance failure can lose locally held session data. A common production design keeps application instances stateless and places shared state in systems designed to manage it.

**Important:** stateless describes the application instance's dependency on local state, not whether the overall system stores data. A stateless service can read and write a stateful PostgreSQL database.

## 5. Dry run: traffic increases

Assume a Spring Boot API can handle about 200 requests/second per instance for the relevant workload. Traffic rises from 150 to 800 requests/second.

### Option A — vertical scaling
Upgrade one instance so its measured capacity rises to 500 requests/second. It still cannot meet the new 800 requests/second demand on its own. A further upgrade, horizontal scaling, or workload optimization may be required.

### Option B — horizontal scaling
Run four instances with an assumed capacity of 200 requests/second each. The simplified aggregate capacity is 800 requests/second. In production, keep capacity headroom and verify that PostgreSQL, Redis, and external services can handle the load generated by all four instances.

The lesson is not “horizontal always wins.” It is that each option changes capacity differently and has different constraints.

## 6. Autoscaling is a separate concept

**Autoscaling** is the automatic adjustment of computing capacity according to configured metrics or policies. It commonly implements horizontal scaling by adding instances when load increases and removing them when load drops.

Keep these distinctions separate:

| Question | Concept |
|---|---|
| Who manages the execution infrastructure? | Serverful vs. serverless |
| How is compute capacity increased or decreased? | Vertical vs. horizontal scaling |
| Who triggers capacity changes? | Manual scaling vs. autoscaling |

Serverful VMs and managed containers can autoscale. Serverless platforms can automatically adjust execution capacity too, subject to service limits. **Serverless is not a synonym for horizontal scaling.**

Autoscaling may take time to react, so the service needs enough baseline capacity and appropriate scaling signals.

## 7. Watch for downstream bottlenecks

Adding application instances can increase concurrent requests to shared systems. For example, if 10 instances each allow up to 50 PostgreSQL connections, they can collectively request up to 500 connections. That may exceed the database's useful capacity or configured connection limit.

Use system-wide thinking:
- Tune each connection pool in the context of the total number of application instances.
- Check whether PostgreSQL can handle the workload and concurrency.
- Optimize queries before assuming the database needs more capacity.
- Consider shared caches, queues, external APIs, and single points of failure.

Scaling one tier can expose a bottleneck in another.

## 8. Tradeoff summary

| Consideration | Vertical scaling | Horizontal scaling |
|---|---|---|
| Approach | Increase resources on one machine | Add more machines/instances |
| Initial complexity | Often simpler | More distributed-system concerns |
| Capacity limit | Machine/platform ceiling | Grows with instances until a shared bottleneck is reached |
| State management | Local state may be easier to maintain in a single instance | Shared state or careful routing is often needed |
| Failure handling | Single-instance deployment can be vulnerable to a machine failure | Multiple instances can provide redundancy, if other components are designed accordingly |
| Cost | Larger machine costs can rise quickly | More instances add cost but allow flexible capacity increments |

A hybrid approach is common: use a suitably sized instance, scale it vertically when sensible, and run multiple instances when capacity or availability requirements call for it.

## 9. Summary card

- **Idea:** Vertical scaling makes one machine more powerful; horizontal scaling adds machines/instances to share work.
- **Tradeoff:** Vertical is often simpler; horizontal offers flexible growth and potential redundancy with added operational complexity.
- **Key concept:** Stateless application instances are easier to scale horizontally because any healthy instance can process an appropriate request.
- **Watch-out:** More app instances can overload shared dependencies, especially PostgreSQL and external services.
- **Project connection:** Run multiple Spring Boot instances behind a load balancer, externalize shared state, and size database connection pools system-wide.
- **Interview one-liner:** “I’d identify the bottleneck first. Vertical scaling increases capacity within one machine; horizontal scaling distributes work across instances, but requires careful handling of shared state, downstream limits, cost, and availability.”
