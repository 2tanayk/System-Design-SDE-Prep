# Threads — OS-Level Fundamentals

## 1. The problem this solves

A backend may receive many requests at nearly the same time. If each request had to wait for every earlier request to finish before it could begin, users would experience unnecessary delays—especially when requests wait for a database or network service.

Threads let a running application make progress on multiple tasks concurrently.

## 2. Core concepts

- **Process:** A running program with its own process-level resources and virtual address space.
- **Thread:** An execution path inside a process. A process can contain one or more threads.
- **Stack:** Per-thread memory used for active method calls and their execution state.
- **Heap:** Memory where Java objects are typically allocated. Threads in the same process can access shared objects in the heap.
- **Scheduler:** The operating system component that decides which eligible thread gets CPU time.
- **Context switch:** The act of pausing one thread and switching execution to another, preserving state so the first can resume later.
- **Thread pool:** A collection of reusable threads that can execute tasks without creating a new thread for every task.

### Process vs. thread

| Process | Thread |
|---|---|
| A running program with process-level resources | An execution path within a process |
| Has its own virtual address space | Shares process resources, including heap memory, with other threads |
| Contains one or more threads | Has its own execution state and stack |
| Separate processes are generally isolated from each other's memory | Threads within one process can access the same objects |

Threads are generally lighter-weight than separate processes, but they still consume memory and scheduling resources.

## 3. Concurrency vs. parallelism

**Concurrency** means multiple tasks make progress during overlapping periods of time. **Parallelism** means multiple tasks execute at the same instant.

With one CPU core, the operating system can schedule a core to run different threads at different times. With multiple available CPU cores, multiple threads may execute simultaneously on different cores.

A context switch has overhead, so having more threads does not automatically make a program faster.

## 4. Dry run: three Spring Boot requests

Assume a conventional servlet-based Spring Boot application uses a request-processing thread pool.

1. Customer A requests restaurant details. Thread 1 runs the controller and service logic, then waits for a synchronous PostgreSQL query.
2. Customer B places an order. Thread 2 can execute its request while Thread 1 is waiting.
3. Customer C checks an order. Thread 3 can process that request as well.
4. When Customer A's database result arrives, Thread 1 resumes, finishes the response, and becomes available for another request.

A **thread pool** reuses a bounded collection of worker threads. If all are busy and more requests arrive, requests may need to wait according to the server's queueing and overload behavior.

This illustrates the conventional synchronous servlet model. Asynchronous APIs and virtual threads can change how waiting work uses threads, but they do not remove downstream capacity limits.

## 5. Why threads help—and where they don't

Threads are useful when multiple requests need to make progress concurrently. In I/O-bound work, a thread may spend time waiting for a database or network response, during which another runnable thread may use the CPU.

Workloads differ:
- **CPU-bound:** Most time is spent computing. More runnable threads than the CPU can effectively serve may add overhead without improving throughput much.
- **I/O-bound:** Much time is spent waiting. Concurrency can help make progress on other tasks, but the database and other dependencies still have finite capacity.

Threads do not make a slow SQL query intrinsically faster. They can also overwhelm a connection pool or downstream service if concurrency is not controlled.

## 6. The major risk: shared mutable state

Threads inside one process can access the same mutable object. If multiple threads update it without suitable coordination, their operations can interleave incorrectly.

### Race condition / lost update

Suppose a shared counter starts at 0, and two threads each increment it. An increment can conceptually involve reading, adding one, and writing.

| Step | Thread A | Thread B | Counter |
|---|---|---|---:|
| 1 | Reads 0 | — | 0 |
| 2 | — | Reads 0 | 0 |
| 3 | Writes 1 | — | 1 |
| 4 | — | Writes 1 | 1 |

The final value is 1, even though two increments were attempted. This is a **race condition**: the outcome depends on timing and interleaving. The lost update occurs because the overall read-modify-write operation was not atomic.

### Spring singleton beans

Spring's default singleton scope gives one bean instance per application context. Multiple request threads may execute methods on that same instance concurrently.

A singleton service is not automatically thread-safe. Avoid storing request-specific mutable state in shared instance fields. Method-local variables are generally isolated to a thread's method execution, though objects referenced from them may still be shared.

### Ways to coordinate safely

- Prefer stateless service objects and explicitly pass request data.
- Use thread-safe data structures or synchronization when shared in-memory state genuinely must be shared.
- Use database transactions, conditional updates, or database locking for shared business data such as inventory.
- Remember that an in-process Java lock coordinates threads only within that process; it does not coordinate separate application instances.

A **critical section** is code that accesses shared state and needs protection from unsafe concurrent execution. Synchronization and locks can provide protection, but contention reduces concurrency and poor lock ordering can cause deadlocks.

A **deadlock** is a situation where threads wait indefinitely for resources held by one another—for example, Thread A holds Lock 1 while waiting for Lock 2, and Thread B holds Lock 2 while waiting for Lock 1.

## 7. Why not create unlimited threads?

Threads consume stack memory and scheduling resources. Excessive runnable threads may cause memory pressure, more context switching, and contention for shared resources.

For example, a request pool with hundreds of threads cannot force PostgreSQL to handle hundreds of useful concurrent queries if the database or connection pool is the bottleneck. More threads may simply wait longer for connections.

Use bounded pools, timeouts, and concurrency limits appropriate to the workload. There is no universally correct thread-pool size; measure behavior under representative load.

## 8. Tradeoffs and choices

| Approach | Choose it when | Tradeoff |
|---|---|---|
| Thread pool | Conventional request/task processing needs reusable worker threads | Pool saturation and queueing need management |
| Asynchronous processing | The framework and dependencies allow a task to wait without occupying a thread in the same way | More complex control flow and error handling; does not make dependencies faster |
| Background worker/queue | Work can complete after the immediate HTTP response | Requires retry, delivery, and completion-state handling |
| Synchronization/locks | Multiple threads must coordinate access to shared mutable memory | Contention and deadlock risks |
| Database-level coordination | Multiple instances must preserve shared-data correctness | Requires correct transactions, constraints, or locking and has database costs |

These options can coexist in one application. They solve different problems rather than being mutually exclusive alternatives.

## 9. Food-delivery backend examples

- **Restaurant browsing:** Several request threads can serve customers concurrently.
- **Order placement:** Threading allows requests to progress independently, but inventory correctness must still be enforced when customers compete for the same stock.
- **Slow payment service:** Waiting calls can occupy request threads in a synchronous model; suitable timeouts and an appropriate asynchronous workflow may help.
- **Shared service field:** Storing the “current customer ID” in a mutable singleton service field can cause one request to overwrite another request's data.
- **Horizontal scaling:** An in-process lock on Server A cannot protect the same inventory row from simultaneous updates made by Server B. Use coordination at the correct system boundary.

## 10. Summary card

- **Idea:** A thread is an execution path within a process; multiple threads allow tasks to make concurrent progress.
- **Key distinction:** Concurrency is overlapping progress; parallelism is simultaneous execution.
- **Project connection:** Conventional Spring Boot servlet applications commonly process synchronous requests using a reusable worker-thread pool.
- **Main risks:** Shared-state race conditions, deadlocks, thread exhaustion, context-switching overhead, and overloaded downstream dependencies.
- **Tradeoff:** More concurrency can improve resource utilization, but it consumes resources and requires safe handling of shared state.
- **Interview one-liner:** “A process contains one or more threads that share process resources. Threads enable concurrent work, but safe and efficient concurrency requires managing shared state, thread capacity, and downstream bottlenecks.”
