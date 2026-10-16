# Serverless vs. Serverful

## 1. The problem this solves

Every backend needs computing resources to execute code and handle requests. Someone must provide the compute environment, keep it available, and make sure enough capacity exists as workload changes.

There are two broad approaches:
- **Serverful:** You provision and operate a server environment for your application, with more direct control over runtime, capacity, and scaling.
- **Serverless:** A cloud provider manages much of the underlying execution infrastructure. You deploy code or an application, and the platform provisions execution capacity as needed.

**Serverless does not mean there are no servers.** It means you do not directly manage the underlying servers in the traditional way.

## 2. Core concepts

- **Server:** A computer that runs software and responds to requests.
- **Provisioning:** Making computing resources available, including allocating CPU and memory.
- **Scaling:** Increasing or reducing available computing capacity.
- **Instance:** One running copy of an application.
- **Function as a Service (FaaS):** A serverless model where a function runs in response to an event, such as an HTTP request or message.
- **API gateway:** A component that accepts API requests and routes them to the appropriate backend handler.
- **Cold start:** The additional delay that can occur when a serverless platform initializes an execution environment before handling a request.

Serverful deployments can also use automated scaling. **Autoscaling and serverless are not synonyms.** Serverless is an operational model; it is not a guarantee of automatic reliability or zero cost while idle.

## 3. Request flow

### Serverful
1. A customer sends a request.
2. A load balancer distributes it across running application instances, when multiple instances exist.
3. A Spring Boot instance processes the request.
4. The application reads or writes data in its database.
5. The response returns to the customer.

You or your platform setup must ensure enough capacity is available. Cloud VMs and managed serverful offerings can still provide autoscaling.

### Serverless function
1. A customer sends a request.
2. An API gateway or another trigger routes it to a function.
3. The managed platform runs the function, allocating execution capacity within configured limits.
4. The function reads or writes data in its database or calls other services.
5. The response returns, or the function completes its event-driven work.

The platform manages much of the execution infrastructure, but quotas, concurrency, startup delay, downstream capacity, retries, and other operational concerns still matter.

## 4. Dry run: lunch-rush traffic

Assume, purely for illustration, that one application instance can comfortably process 200 requests per second.

| Period | Incoming traffic | Simple capacity implication |
|---|---:|---|
| Normal | 100 requests/sec | One instance may be enough |
| Lunch rush | 1,000 requests/sec | At least five instances by this simplified calculation |
| After rush | 80 requests/sec | One instance may be enough again |

**Serverful:** capacity must be available for the rush. This could mean adding instances manually or configuring autoscaling. Scale down later if the setup permits it.

**Serverless:** the platform can increase concurrent executions as demand rises, within its quotas and configuration, and reduce active capacity as demand falls.

This does not make serverless infinitely scalable. If all executions need the same PostgreSQL database, the database can become the bottleneck. For example, 1,000 concurrent executions trying to open database connections can overwhelm a database with a much smaller connection limit.

## 5. Tradeoffs

| Consideration | Serverful | Serverless |
|---|---|---|
| Infrastructure control | More direct control | More platform-managed |
| Scaling | Configure or use platform autoscaling | Platform can adjust execution capacity automatically within limits |
| Traffic patterns | Often fits steady, predictable workloads | Often fits bursty or event-driven workloads |
| Idle capacity | Provisioned capacity may cost money while idle | Some models reduce compute charges when idle, but other charges remain |
| Runtime | Broader control, depending on VM/container setup | Constrained by the selected platform's capabilities and limits |
| Operational effort | More infrastructure responsibility | Less server management, but application operations remain |
| Cost predictability | Provisioned capacity can be easier to budget | Costs vary with usage and pricing model |

### Choose serverful when
- Traffic is steady and provisioned capacity can be used efficiently.
- You need a persistent process, long-running worker, or more control over the runtime and operating environment.
- Latency requirements make cold-start risk unacceptable and provisioned serverless capacity is not a good fit.
- You need capabilities that a particular serverless platform does not support.

### Choose serverless when
- Traffic is irregular, bursty, or intermittent.
- You have short, independent event-driven tasks, scheduled work, or handlers that fit service limits.
- You want to reduce time spent provisioning and managing execution infrastructure.
- The platform's limits, pricing, and downstream dependencies fit the workload.

### Consider managed containers as a middle ground
A managed container platform can run a Spring Boot service without requiring you to manage each underlying server directly. Your application can remain running, and the platform can manage deployment and scaling. This can be a useful baseline for conventional REST APIs and persistent workers.

## 6. When serverless may be a poor fit

Avoid or reconsider serverless **when a critical requirement cannot be met by the specific platform**. It is not a blanket rule that serverless should never be used for any of these workloads.

### 6.1 Consistently low latency
Some platforms can scale execution environments down when idle. A later request may experience a cold start. This can be noticeable for Java applications with framework initialization. Consider continuously running instances or serverless provisioned/warmed capacity when the latency target requires it.

### 6.2 Long-running or continuously active work
Function platforms may impose maximum execution time, memory, or other resource limits. Tasks that run for hours or workers that continuously consume Kafka messages may require checkpointing and splitting, or may fit better in a persistent worker/container. Limits vary by service; some serverless job platforms support longer work.

### 6.3 Sustained, predictable, high utilization
At consistently high usage, per-invocation or execution-duration charges can exceed the cost of efficiently utilized servers or containers. This depends on the provider, CPU/memory, networking, logging, discounts, and operational costs. Compare total cost rather than assuming one model is always cheaper.

### 6.4 Runtime or process requirements
A function platform may not support a required runtime, native dependency, OS-level configuration, persistent process, or particular networking control. A VM or container offers more control, while managed containers may still remove much infrastructure work.

### 6.5 Downstream systems cannot handle burst concurrency
Serverless compute may scale more quickly than the database, queue, or third-party API can handle. Control concurrency and admission rate, use suitable connection pooling/proxies, and design retries carefully. Serverful applications can overload dependencies too; bounded capacity is not an automatic fix.

### 6.6 Portability or specialized compliance requirements
Provider-specific integrations may increase migration cost. Some organizations require network, host, deployment, or operational controls unavailable from a given service. Assess the actual platform against requirements—serverless is not inherently noncompliant or insecure.

## 7. Food-delivery project examples

- **Main Spring Boot order API:** A managed container or VM may be a natural starting point because it is a conventional, continuously running service.
- **Small payment-webhook handler:** Serverless may fit an HTTP-triggered task, provided duplicate deliveries, idempotency, retries, and database load are handled correctly.
- **Scheduled report or cleanup:** A scheduled serverless invocation may work well for a short task.
- **Continuous Kafka consumer:** A persistent worker/container is often a better fit than a function platform designed for short invocations.

A single system can mix these approaches. The choice is made per component, not necessarily for the whole application.

## 8. Decision checklist

Before choosing serverless, ask:

1. **Execution:** Does the task fit the runtime and resource limits?
2. **Latency:** Can it meet the response-time target, including any startup delay?
3. **Cost:** Is it cost-effective at expected usage, including networking, logging, and other dependencies?
4. **Dependencies:** Can databases, queues, and external APIs handle the scaling behavior?
5. **Control:** Does the platform support the runtime, networking, and operational requirements?

If serverless fails a critical requirement, don't force it into the design.

## 9. Summary card

- **Idea:** Serverful means more direct control of the application environment; serverless means the provider manages more of the execution infrastructure.
- **Tradeoff:** Control and provisioned capacity versus less server management and usage-flexible compute.
- **Remember:** Serverless still uses servers, and serverful systems can autoscale too.
- **Watch-outs:** Cold starts, execution limits, cost at sustained utilization, platform constraints, and downstream bottlenecks.
- **Project link:** Consider a continuously running Spring Boot order API versus a short payment-webhook handler or scheduled task.
- **Interview one-liner:** “I'd choose based on workload shape, latency, runtime needs, total cost, and downstream capacity—not because serverless is always cheaper or serverful cannot autoscale.”
