---
description: Architectural reference for WorldGenTimingsCollector
---

# WorldGenTimingsCollector

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen
**Type:** Transient

## Definition
```java
// Signature
public class WorldGenTimingsCollector {
```

## Architecture & Concepts
The WorldGenTimingsCollector is a high-performance, thread-safe instrumentation component designed to measure the performance of discrete stages within the server-side world generation pipeline. It acts as a specialized data sink for timing metrics, aggregating nanosecond-precision data from numerous concurrent chunk generation tasks.

Its primary architectural role is to decouple the world generation logic from the server's central monitoring and metrics system. It achieves this by providing a simple reporting API to worker threads while exposing its aggregated data through a static MetricsRegistry. This registry allows external systems to periodically scrape performance data without interfering with the high-throughput generation process.

A key design feature is the built-in "warm-up" period. The collector intentionally discards timing data for the first 100 chunks generated. This prevents initial JIT compilation overhead and cache misses from polluting the steady-state performance metrics, ensuring that the reported data accurately reflects the server's sustained operational throughput.

The collector is also designed to provide insight into the health of the underlying execution engine by directly querying the world generation ThreadPoolExecutor for queue depth and active worker counts.

## Lifecycle & Ownership
- **Creation:** An instance is created and owned by the primary service that manages the world generation thread pool, such as a WorldGenerator or UniverseManager. The ThreadPoolExecutor that executes chunk generation tasks is a mandatory dependency injected during construction.
- **Scope:** The collector's lifecycle is strictly bound to the world generation system it monitors. It persists for the entire duration that its associated ThreadPoolExecutor is active. A separate instance would exist for each distinct world generation pipeline.
- **Destruction:** The object is eligible for garbage collection when the parent world generation service is shut down and releases its reference to both the collector and the thread pool. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** The internal state is highly mutable, consisting of atomic counters for total chunks processed, cumulative nanoseconds per generation stage, and total reports per stage. This state represents a running average of world generation performance.
- **Thread Safety:** This class is unconditionally thread-safe and designed for high-contention environments. All internal counters and accumulators use lock-free atomic primitives (AtomicLong, AtomicLongArray). This design choice is critical to minimize performance overhead, as report methods are called frequently from many different worker threads simultaneously. No explicit locking is used.

## API Surface
The public API is divided into two categories: data reporting and data retrieval. Reporting methods are called by worker threads, while retrieval methods are typically invoked by the metrics system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| reportChunk(long nanos) | double | O(1) | Reports the total time for a single chunk generation. |
| reportZoneBiomeResult(long nanos) | double | O(1) | Reports time spent on the Zone Biome stage. |
| reportPrepare(long nanos) | double | O(1) | Reports time spent on the Prepare stage. |
| reportBlocksGeneration(long nanos) | double | O(1) | Reports time spent on the Blocks Generation stage. |
| reportCaveGeneration(long nanos) | double | O(1) | Reports time spent on the Cave Generation stage. |
| reportPrefabGeneration(long nanos) | double | O(1) | Reports time spent on the Prefab Generation stage. |
| getQueueLength() | int | O(1) | Returns the current number of tasks in the executor's work queue. |
| getGeneratingCount() | int | O(1) | Returns the number of threads actively executing tasks. |

**WARNING:** The `report` methods will return Double.NEGATIVE_INFINITY during the initial warm-up phase (the first 100 chunks). Do not rely on their return value for immediate feedback.

## Integration Patterns

### Standard Usage
The collector should be instantiated by a manager class and passed by reference to the worker tasks it creates. The worker then reports its progress as it completes each stage.

```java
// In a WorldGenerator or similar service
ThreadPoolExecutor worldGenExecutor = createMyExecutor();
WorldGenTimingsCollector timings = new WorldGenTimingsCollector(worldGenExecutor);

// In a ChunkGenerationTask running on the executor
public void run() {
    long startTime = System.nanoTime();
    // ... perform biome calculations ...
    long biomeTime = System.nanoTime();
    timings.reportZoneBiomeResult(biomeTime - startTime);

    // ... perform block generation ...
    long blockTime = System.nanoTime();
    timings.reportBlocksGeneration(blockTime - biomeTime);

    // ... etc ...
    timings.reportChunk(System.nanoTime() - startTime);
}
```

### Anti-Patterns (Do NOT do this)
- **Global Singleton:** Do not treat this class as a global singleton. Each world generation pipeline (e.g., Overworld, Nether) with its own thread pool requires its own dedicated WorldGenTimingsCollector instance for metrics to be meaningful.
- **Mismatched Executor:** Never share a single collector instance across multiple, unrelated ThreadPoolExecutors. The queue and active count metrics would become nonsensical and misleading.
- **External Instantiation:** Worker tasks must not create their own instances of the collector. They must receive a shared instance from their parent factory or service to ensure all data is aggregated correctly.

## Data Pipeline
The WorldGenTimingsCollector is a collection point, not a processing node. It aggregates data from many sources and makes it available for a single consumer (the metrics registry).

> Flow:
> Chunk Generation Task -> `report...()` call -> **WorldGenTimingsCollector** (Atomic State Update) -> MetricsRegistry (Periodic Scrape) -> Server Monitoring Dashboard

