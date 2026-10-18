# Pages, Virtual Memory, and JVM Memory Basics

## 1. The problem: memory is finite

A machine has a finite amount of physical RAM, and multiple processes need memory at the same time. The operating system must allocate memory, track where data lives, and isolate processes so one process cannot arbitrarily access another process's private memory.

**Virtual memory** helps address these needs by giving a process its own virtual address space and mapping its virtual addresses to physical memory with hardware and OS support.

## 2. Pages and frames

- **Page:** A fixed-size block of virtual memory.
- **Frame:** A same-sized block of physical RAM that can hold a page.
- **Virtual address:** An address used by a program.
- **Physical address:** An address referring to a location in RAM.

Page size depends on the processor architecture and system configuration. 4 KB is a common page size, but not the only possibility.

For example, if a process uses 16 KB of virtual memory and the page size is 4 KB, that memory can be divided into four virtual pages:

~~~text
Virtual memory
+-----------+
| Page 0    | 4 KB
+-----------+
| Page 1    | 4 KB
+-----------+
| Page 2    | 4 KB
+-----------+
| Page 3    | 4 KB
+-----------+
Total: 16 KB
~~~

Physical RAM is divided into frames of the corresponding size. A process's virtual pages do not need to occupy adjacent physical frames.

## 3. Virtual addresses and page tables

Programs generally use virtual addresses rather than directly choosing physical RAM addresses. The **Memory Management Unit (MMU)** is hardware that translates virtual addresses to physical addresses using memory-mapping information maintained by the operating system.

The OS tracks virtual-page-to-physical-frame mappings in a **page table**.

Example mapping:

| Virtual page | Physical frame |
|---|---|
| Page 0 | Frame 5 |
| Page 1 | Frame 2 |
| Page 2 | Frame 9 |
| Page 3 | Frame 1 |

Although the virtual pages are sequential, their physical frames are scattered:

~~~text
Page 0  ----------> Frame 5
Page 1  ----------> Frame 2
Page 2  ----------> Frame 9
Page 3  ----------> Frame 1
~~~

When the program accesses a virtual address on Page 2, the MMU consults the translation information and locates the corresponding physical memory location in Frame 9.

**Why this helps:** paging lets the OS allocate physical memory in fixed-size frames without requiring a process's virtual pages to be stored in one contiguous physical region. Virtual address spaces and memory protection also help isolate processes.

## 4. What happens when a needed page is absent?

A **page fault** occurs when a process accesses a virtual-memory page that cannot currently be accessed through a valid resident mapping. One common reason is that the page is not currently in physical RAM.

Conceptual flow:
1. A process accesses a virtual address.
2. The hardware detects that the mapping needs OS handling.
3. The OS determines whether the access is valid and what handling is required.
4. If valid, the OS may load the page into a frame or otherwise resolve the fault, update mappings, and resume execution.
5. If the access is invalid, the OS may report an error to the process.

Not every page fault requires disk I/O. Some faults can be handled without reading a page from disk. When a page must be retrieved from backing storage, the wait can be much slower than accessing resident RAM.

If memory pressure causes the system to repeatedly evict and reload pages that processes actively need, performance can degrade severely. This behavior is called **thrashing**.

A process's **working set** is the set of pages it is actively using over a period of time. If active memory demands greatly exceed available physical RAM, the system may spend disproportionate time managing pages instead of doing useful work.

## 5. Process vs. thread vs. machine

These are three different levels:

| Level | Meaning |
|---|---|
| Machine | Physical hardware and RAM used by the system |
| Process | A running program with its own virtual address space and process-level resources |
| Thread | An execution path within a process, with its own execution state and stack |

A machine normally runs many processes, not just one. For example, a Linux server may run:
- A JVM process containing a Spring Boot application.
- A separate PostgreSQL process, if the database is on the same machine.
- A Redis process, monitoring agents, and other applications or system processes.

PostgreSQL or Redis may instead run on separate machines in a real deployment. Separate processes generally have isolated virtual address spaces and communicate through mechanisms such as network connections or local sockets.

When you run the command "java -jar app.jar", the operating system starts a JVM process, and your Java application executes inside it. The JVM is not the only process running on the machine.

## 6. Stack and heap are not Java-only concepts

**Stack** and **heap** are broadly used programming and memory-management concepts, not concepts exclusive to Java. Different languages and runtimes can organize memory differently.

Within a typical JVM process, there are several memory areas:

~~~text
JVM process
|
+-- Java heap
|    +-- Java objects and arrays (typically)
|
+-- Thread stacks
|    +-- Each thread's method-call and execution state
|
+-- Metaspace
|    +-- Class metadata managed by the JVM
|
+-- Other memory
     +-- JIT code cache, native allocations, runtime data
~~~

This is a conceptual view, not a complete map of every JVM memory region.

### Java stack

Each thread has its own JVM stack. The stack stores information related to active method calls, including execution state and local variables.

Example:

~~~java
void processOrder() {
    int quantity = 2;
    Restaurant restaurant = new Restaurant();
}
~~~

Conceptually, quantity and the local reference variable restaurant are associated with this method's execution state on the current thread's stack.

### Java heap

The Restaurant object is typically allocated on the Java heap. The heap is shared by threads within the same JVM process, so different threads can access the same object if they have references to it.

The reference and the object it refers to are distinct things. This is a useful mental model, although JVM optimizations may change the actual allocation details.

### Other JVM memory

- **Metaspace:** Holds class metadata managed by the JVM.
- **Thread stacks:** Each thread has its own stack.
- **Other JVM/native memory:** May include compiled-code caches, native allocations, and runtime structures.

The JVM uses memory beyond the Java heap. Therefore, total process memory consumption is not the same thing as Java heap usage.

## 7. Garbage collection vs. paging

These mechanisms solve different problems:

- **Garbage collection (GC):** The JVM reclaims Java heap memory occupied by objects that are no longer reachable.
- **Paging:** The operating system and hardware manage virtual-to-physical memory mappings and residency in RAM.

A Java application may experience excessive garbage collection without significant paging. A process can also experience operating-system memory pressure for reasons beyond its Java heap.

## 8. Spring Boot example: excessive memory use

Imagine a restaurant-search endpoint loads hundreds of thousands of database records into memory even though it returns only a small subset.

1. The application materializes many Java objects.
2. Heap usage and allocation activity rise.
3. Temporary objects may increase garbage-collection work.
4. The JVM process's total memory consumption may contribute to system-wide RAM pressure.
5. Depending on the environment, the OS may reclaim or page memory; severe pressure can increase latency or cause failures.

Practical responses include:
- Select only fields the API needs.
- Paginate large results or stream them when appropriate.
- Avoid loading entire tables into memory.
- Observe heap usage, garbage-collection behavior, process memory, and OS-level memory pressure.
- Configure appropriate application/container memory limits and available capacity.

Do not confuse this with a guarantee that high heap usage always causes paging. The actual behavior depends on JVM configuration, OS behavior, physical RAM, other processes, and workload.

## 9. Key takeaways and tradeoffs

- Virtual memory gives each process a virtual address space, with mappings to physical memory managed by the OS and hardware.
- Paging divides virtual memory into pages and physical RAM into frames; a process's pages can occupy non-contiguous frames.
- A page fault requires handling when an access cannot proceed with the current resident mapping; some faults require retrieving data from backing storage.
- Paging provides flexibility but does not make physical memory unlimited. Severe memory pressure and thrashing can hurt performance.
- A JVM is one process among others. Inside it, Java heap objects, per-thread stacks, and other runtime memory all contribute to memory use.
- Garbage collection and paging are different mechanisms.

## 10. Summary card

- **Idea:** Virtual memory is divided into pages, while physical RAM is managed in frames; page tables map virtual pages to frames.
- **Hardware/OS:** The MMU translates virtual addresses; the OS manages mappings and handles page-fault events.
- **Process model:** Your Spring Boot app runs inside a JVM process; a machine usually has many processes.
- **Java memory:** The heap is shared among threads in a JVM; each thread has its own stack. Stack and heap are not Java-only concepts.
- **Performance:** Repeated paging under memory pressure can cause major slowdowns; GC and paging are different mechanisms.
- **Interview one-liner:** “Paging lets the OS map a process's virtual pages to physical frames without requiring contiguous physical memory. The JVM itself is a process with heap, per-thread stacks, and other runtime memory; excessive memory demand can trigger GC overhead or OS-level pressure.”
