---
description: Architectural reference for IWorldGenBenchmark
---

# IWorldGenBenchmark

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IWorldGenBenchmark {
```

## Architecture & Concepts
The IWorldGenBenchmark interface defines a strict contract for performance measurement and instrumentation within the server-side world generation pipeline. It serves as an abstraction layer, decoupling the core world generation algorithms from the concrete implementation of performance monitoring.

This design allows the engine to swap out different benchmarking strategies without altering the world generation code itself. For example, a development environment might use a verbose, console-logging implementation, while a production environment could use a lightweight, no-op implementation or one that reports metrics to a dedicated monitoring service.

The primary role of this interface is to provide a standardized lifecycle for capturing timing data: a clear start, a clear stop, and an asynchronous mechanism for report compilation.

### Lifecycle & Ownership
As an interface, IWorldGenBenchmark itself has no lifecycle. The following describes the expected lifecycle of a concrete implementation.

- **Creation:** An implementation of IWorldGenBenchmark is expected to be instantiated by a high-level orchestrator, such as a WorldCreationTask or a ChunkGenerator, at the beginning of a discrete and measurable generation process.
- **Scope:** The scope of an implementation instance is tightly bound to a single, complete world generation operation (e.g., generating a new world or a specific set of chunks). It is a short-lived, single-use object.
- **Destruction:** The object is eligible for garbage collection immediately after the report has been generated and consumed via the CompletableFuture. It holds no persistent state beyond the scope of its operation.

## Internal State & Concurrency
The interface itself is stateless. However, any non-trivial implementation is inherently stateful.

- **State:** Implementations are expected to maintain internal mutable state, such as start and stop timestamps, and potentially a collection of intermediate timing points or event counters. This state is accumulated between the calls to start and stop.
- **Thread Safety:** Implementations are **not** required to be thread-safe. The contract assumes a single-threaded invocation pattern for a given instance: start(), followed by various world-gen operations, followed by stop(), and finally buildReport(). Concurrent access to a single benchmark instance from multiple threads will lead to undefined behavior and corrupted metrics.

## API Surface
The public contract is minimal, focusing exclusively on the benchmarking lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start() | void | O(1) | Marks the beginning of the measurement period. Throws IllegalStateException if already started. |
| stop() | void | O(1) | Marks the end of the measurement period. Throws IllegalStateException if not yet started. |
| buildReport() | CompletableFuture<String> | O(N) | Asynchronously compiles and returns a formatted string report of the captured metrics. |

**Warning:** The CompletableFuture returned by buildReport indicates that report generation may be a non-trivial, potentially IO-bound, or CPU-intensive operation. Consumers must handle this asynchronicity correctly and should not block on the result in performance-critical code paths.

## Integration Patterns

### Standard Usage
The intended pattern is to create, use, and discard a benchmark instance for a single, well-defined scope of work.

```java
// A world generator orchestrator obtains a benchmark implementation
IWorldGenBenchmark benchmark = new VerboseWorldGenBenchmark();

try {
    benchmark.start();
    // ... execute complex, multi-stage world generation logic ...
    world.generateTerrain();
    world.populateFeatures();
} finally {
    benchmark.stop();
    // Asynchronously process the report without blocking the main thread
    benchmark.buildReport().thenAccept(report -> {
        ServerLogger.info("WorldGen complete. Report:\n" + report);
    });
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not reuse the same IWorldGenBenchmark instance for multiple, distinct generation tasks. This will corrupt timing data. Each task requires a new instance.
- **Incorrect Invocation Order:** Calling stop before start or calling start multiple times on the same instance is a contract violation and will result in an IllegalStateException.
- **Blocking on Report:** Do not call .get() or .join() on the CompletableFuture from buildReport within the main server tick or any other performance-sensitive loop. This negates the benefit of the asynchronous design.

## Data Pipeline
This component does not process a continuous stream of data. Instead, it captures state at two distinct points in time to produce a final summary.

> Flow:
> WorldGen Orchestrator -> `start()` (Captures T0) -> [World Generation Process] -> `stop()` (Captures T1) -> `buildReport()` (Computes T1-T0 and other metrics) -> CompletableFuture -> Formatted String Report

