---
description: Architectural reference for ChunkWorldgenBenchmark
---

# ChunkWorldgenBenchmark

**Package:** com.hypixel.hytale.server.worldgen.benchmark
**Type:** Transient

## Definition
```java
// Signature
public class ChunkWorldgenBenchmark implements IWorldGenBenchmark {
```

## Architecture & Concepts
The ChunkWorldgenBenchmark class is a specialized, in-memory data aggregator used for performance and diagnostic analysis of the server's world generation process. Its primary function is to act as a thread-safe sink for statistical data, specifically counting the occurrences of generated prefabs and cave nodes during a benchmark session.

This component is designed for high-throughput, concurrent environments. The world generation system executes across multiple worker threads, and this class provides a safe and efficient mechanism for these threads to report their progress without introducing lock contention. It achieves this by leveraging concurrent data structures for its internal state.

Architecturally, it embodies the *Collector* pattern. It does not participate in the logic of world generation itself; it passively observes and records events, decoupling the benchmarking concern from the core generation algorithms.

### Lifecycle & Ownership
-   **Creation:** An instance of ChunkWorldgenBenchmark is typically created on-demand by a higher-level system, such as a server command handler or a dedicated benchmark management service, to initiate a profiling session. It is not a global singleton.
-   **Scope:** The object's lifetime is strictly bound to a single benchmark run. The session begins with a call to the start method and concludes after the stop method is invoked and a report is generated.
-   **Destruction:** The instance is eligible for garbage collection once the benchmark session is complete and all external references are released. The stop method explicitly clears the internal collections to release memory, preventing data from one session from leaking into another.

## Internal State & Concurrency
-   **State:** This class is highly mutable. Its core state consists of two maps, prefabCounts and caveNodeCounts, which are populated and grown throughout the benchmark's lifecycle. An enabled flag also tracks the active state of the benchmark.

-   **Thread Safety:** The class is designed for partial thread safety, which requires careful handling by the caller.
    -   **Data Collection:** The registerPrefab and registerCaveNode methods are fully thread-safe. They utilize ConcurrentHashMap and AtomicInteger to guarantee that increments from multiple world generation threads are performed atomically and without data loss.
    -   **Lifecycle Management:** The start and stop methods are **not** thread-safe. They are intended to be called from a single, controlling thread that manages the benchmark session. Calling these methods concurrently with each other or with registration methods will result in undefined behavior.
    -   **Visibility:** The enabled flag is not declared as volatile. This implies that the system relies on the Java Memory Model's guarantees for thread start/stop synchronization or that a happens-before relationship is established by the framework that manages the worker threads. Callers should not assume immediate visibility of this flag's state change across all threads without proper memory barriers.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start() | void | O(1) | Enables the benchmark session. This is a state-mutating operation. |
| stop() | void | O(N) | Disables the session and clears all collected data. N is the number of unique keys. |
| buildReport() | CompletableFuture<String> | O(N log N) | Asynchronously generates a sorted, formatted report. The complexity is dominated by sorting keys. |
| isEnabled() | boolean | O(1) | Returns the current state of the benchmark session. |
| registerPrefab(name) | void | O(1) amortized | Atomically increments the count for a given prefab. Thread-safe. |
| registerCaveNode(name) | void | O(1) amortized | Atomically increments the count for a given cave node. Thread-safe. |

## Integration Patterns

### Standard Usage
The intended use follows a strict lifecycle managed by a single controller thread. Worker threads performing world generation can then safely call the registration methods.

```java
// Example: A benchmark controller initiates and finalizes the process
ChunkWorldgenBenchmark benchmark = new ChunkWorldgenBenchmark();

// 1. Start the benchmark from the main thread
benchmark.start();

// 2. Pass the instance to world generation workers, which may run in parallel
//    (This part is conceptual and happens inside the engine)
worldGenerator.runOnWorkers(() -> {
    // Each worker checks if the benchmark is active
    if (benchmark.isEnabled()) {
        // ...generation logic...
        benchmark.registerPrefab("hytale:oak_tree_large");
        benchmark.registerCaveNode("hytale:ore_vein_diamond");
    }
});

// 3. After all workers are finished, stop the benchmark from the main thread
benchmark.stop();

// 4. Generate and process the report
benchmark.buildReport().thenAccept(System.out::println);
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring the Lifecycle:** Do not call registration methods without a corresponding start/stop sequence. While the methods will still function, this violates the intended state management and can lead to data from different logical sessions being mixed.
-   **Concurrent State Management:** Do not call start or stop from multiple threads. These methods are not reentrant or thread-safe and must be controlled from a single source.
-   **Assuming Internal Checks:** The registerPrefab and registerCaveNode methods do **not** internally check the isEnabled flag. They will increment their counters regardless of the benchmark's state. The calling code is responsible for checking isEnabled to prevent unnecessary work.

## Data Pipeline
This class acts as a terminal data sink during the collection phase and a data source during the reporting phase.

> Flow:
> World Generation Worker Thread -> `registerPrefab()` -> **ChunkWorldgenBenchmark (Internal ConcurrentHashMap)** -> `buildReport()` -> Formatted String Report

