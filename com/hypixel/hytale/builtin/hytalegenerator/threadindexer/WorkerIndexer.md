---
description: Architectural reference for WorkerIndexer
---

# WorkerIndexer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.threadindexer
**Type:** Utility

## Definition
```java
// Signature
public class WorkerIndexer {
```

## Architecture & Concepts
The WorkerIndexer is a foundational utility for managing and identifying a fixed-size pool of logical workers. It is a core component in systems that require parallel processing, such as world generation or chunk processing, where tasks are distributed across multiple threads.

Its primary purpose is to decouple task logic from the underlying threading model. Instead of relying on volatile identifiers like Java Thread IDs, it provides a stable, zero-based index for each potential worker slot. This allows for the creation of per-worker data structures that are not tied to the lifecycle of a specific thread, a pattern superior to using ThreadLocal when tasks can be executed by different threads in a work-stealing pool.

The system is composed of three key concepts:
*   **WorkerIndexer:** The main orchestrator, which defines the total number of worker slots and acts as a factory for worker IDs and Sessions.
*   **WorkerIndexer.Id:** An immutable, lightweight handle representing a single worker slot. It is essentially a strongly-typed integer wrapper, preventing accidental use of arbitrary integers for indexing.
*   **WorkerIndexer.Data:** A generic, lazily-initialized data store indexed by a WorkerIndexer.Id. This allows each logical worker to have its own dedicated data instance (e.g., a buffer, a random number generator, or a cache) without contention.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor, `new WorkerIndexer(workerCount)`. It is typically created once at the beginning of a large-scale parallel operation, often using the number of available CPU cores as the `workerCount`.
- **Scope:** The WorkerIndexer object is designed to live for the duration of the parallel process it orchestrates. As it is immutable and lightweight, it can be passed by reference to any system that needs to coordinate worker tasks.
- **Destruction:** The object and its associated IDs are managed by the Java garbage collector. There are no native resources or explicit cleanup methods required.

## Internal State & Concurrency
- **State:** The WorkerIndexer class itself is **immutable**. Its internal state, the `workerCount` and the list of `Id` objects, is finalized during construction and cannot be changed. This makes the main WorkerIndexer object inherently thread-safe.

- **Thread Safety:**
    - **WorkerIndexer:** **Thread-safe**. Can be safely shared and accessed by any number of threads.
    - **WorkerIndexer.Session:** **NOT thread-safe**. A Session instance maintains a mutable internal index and is designed for single-threaded, iterator-style access. Sharing a Session across threads will result in race conditions and incorrect ID assignment. Each thread or sequential process must create its own Session.
    - **WorkerIndexer.Data:** **NOT thread-safe for initialization**. The `get` method contains a check-then-act race condition. If two threads call `get` for the same uninitialized ID simultaneously, the initialization `Supplier` may be invoked more than once, and data may be overwritten. Access must be externally synchronized, or, more commonly, the design should ensure that only one thread is responsible for initializing and using a specific ID's data slot.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorkerIndexer(int) | constructor | O(N) | Creates the indexer and pre-allocates N worker IDs. Throws IllegalArgumentException if workerCount is not positive. |
| createSession() | WorkerIndexer.Session | O(1) | Returns a new, stateful session for iterating through worker IDs. **Warning:** Not thread-safe. |
| getWorkerCount() | int | O(1) | Returns the total number of worker slots configured. |
| getWorkedIds() | List<WorkerIndexer.Id> | O(1) | Returns an unmodifiable list of all pre-generated worker IDs. |

## Integration Patterns

### Standard Usage
The typical pattern involves creating a central WorkerIndexer, creating a per-worker data store, and then passing a specific worker ID to each dispatched task.

```java
// 1. At system startup, create an indexer based on available cores.
int coreCount = Runtime.getRuntime().availableProcessors();
WorkerIndexer indexer = new WorkerIndexer(coreCount);

// 2. Create a per-worker data store, for example, for noise generation.
WorkerIndexer.Data<PerlinNoiseGenerator> noiseGenerators = new WorkerIndexer.Data<>(
    indexer.getWorkerCount(),
    () -> new PerlinNoiseGenerator(/* seed */)
);

// 3. When dispatching a task to a worker thread or pool, assign it an ID.
// This example simulates dispatching work for worker 3.
WorkerIndexer.Id workerId = indexer.getWorkedIds().get(3);

// 4. Inside the worker task, use the ID to retrieve its dedicated resources.
PerlinNoiseGenerator myGenerator = noiseGenerators.get(workerId);
// ... perform work using myGenerator ...
```

### Anti-Patterns (Do NOT do this)
- **Sharing a Session:** Never pass the same Session object to multiple threads. Each thread requiring iterator-style access must call `createSession()` to get its own instance.
- **Concurrent `Data` Initialization:** Do not allow multiple threads to call `Data.get(id)` for the same `id` if it might not be initialized. This is a race condition. The common pattern is to have one thread "own" and operate on one ID's data slot for a given operation.
- **Mismatched Sizing:** Do not create a `WorkerIndexer.Data` instance with a size that does not match the `workerCount` of the WorkerIndexer that will be used to access it. This will result in runtime exceptions.

## Data Pipeline
WorkerIndexer does not process a stream of data itself; rather, it provides the identifiers used to partition data and resources for parallel processing.

> Flow:
> Parallel System Initializer -> **new WorkerIndexer(count)** -> Task Dispatcher assigns `WorkerIndexer.Id` to each task -> Task uses `Id` to access a slot in a `WorkerIndexer.Data` store -> Task processes its workload using its dedicated resources.

