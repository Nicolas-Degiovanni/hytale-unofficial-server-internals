---
description: Architectural reference for IBenchmarkableWorldGen
---

# IBenchmarkableWorldGen

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen
**Type:** Interface

## Definition
```java
// Signature
public interface IBenchmarkableWorldGen extends IWorldGen {
```

## Architecture & Concepts
The IBenchmarkableWorldGen interface extends the core IWorldGen contract to introduce performance monitoring capabilities. It represents a specialized type of world generator that can be instrumented and analyzed for performance bottlenecks.

This interface embodies the **Decorator** or **Strategy** pattern, allowing the engine to attach performance measurement logic to world generation algorithms without altering their primary generation responsibilities. It is a critical tool for server operators and developers to diagnose and optimize world creation, which is one of the most computationally expensive processes in the server lifecycle. Any world generator that needs to be profiled must implement this interface.

## Lifecycle & Ownership
As an interface, IBenchmarkableWorldGen does not have its own lifecycle. Instead, it defines a contract that implementing classes must adhere to. The lifecycle of an object implementing this interface is dictated by the server's world management system.

- **Creation:** An implementing class is typically instantiated by a WorldFactory or a similar high-level controller when a new world or region requires generation. It is often selected from a registry of available world generators based on world configuration.
- **Scope:** The instance's lifetime is tied to a specific, bounded world generation task. It may be scoped to the generation of a single world, a region, or even a transient set of chunks for preview. It is not a long-lived, session-wide service.
- **Destruction:** The object is eligible for garbage collection once the world generation task it was created for is complete and all references to it, including its benchmark results, are released.

## Internal State & Concurrency
The interface itself is stateless. However, any concrete implementation is expected to be highly stateful, accumulating timing and performance data during its operation.

- **State:** Implementations will maintain mutable state within their associated IWorldGenBenchmark object. This state includes metrics such as timings for different generation stages, invocation counts, and memory allocation statistics.
- **Thread Safety:** World generation is a massively parallel operation. Implementations of this interface **must be thread-safe**. Access to the underlying benchmark data, especially from the getBenchmark method, must be synchronized or use concurrent data structures to prevent race conditions when metrics are being updated by worker threads and read by a monitoring thread simultaneously.

## API Surface
The interface adds a single method to the IWorldGen contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBenchmark() | IWorldGenBenchmark | O(1) | Retrieves the benchmark data container associated with this world generator instance. This method provides a snapshot or live view of performance metrics. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves dynamically checking if a given IWorldGen instance supports benchmarking and then retrieving the results after a generation pass is complete.

```java
// How a developer should normally use this
IWorldGen generator = world.getActiveGenerator();

// Generation logic is executed here...
generator.generateChunk(position);

// After generation, check for benchmark capability
if (generator instanceof IBenchmarkableWorldGen) {
    IBenchmarkableWorldGen benchmarkableGenerator = (IBenchmarkableWorldGen) generator;
    IWorldGenBenchmark results = benchmarkableGenerator.getBenchmark();
    
    // Log or process the performance results
    log.info("WorldGen Performance: " + results.getSummary());
}
```

### Anti-Patterns (Do NOT do this)
- **Unsafe Casting:** Never assume an IWorldGen instance also implements IBenchmarkableWorldGen. Always use an `instanceof` check before casting to avoid a ClassCastException.
- **Frequent Polling:** Avoid calling getBenchmark repeatedly within a tight loop during active generation. While implementations should be thread-safe, frequent calls may introduce contention or performance overhead. It is designed to be called after a significant unit of work is completed.

## Data Pipeline
This interface does not participate in the primary world data pipeline. Instead, it creates a secondary pipeline for telemetry and performance data.

> **Primary Flow (World Data):**
> Generation Algorithm -> **IWorldGen Implementation** -> Chunk Data -> World Storage
>
> **Secondary Flow (Metrics Data):**
> Generation Algorithm -> (Internal Timers) -> **IBenchmarkableWorldGen Implementation** -> IWorldGenBenchmark -> Monitoring System / Logs

