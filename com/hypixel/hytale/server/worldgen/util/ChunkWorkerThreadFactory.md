---
description: Architectural reference for ChunkWorkerThreadFactory
---

# ChunkWorkerThreadFactory

**Package:** com.hypixel.hytale.server.worldgen.util
**Type:** Factory

## Definition
```java
// Signature
public class ChunkWorkerThreadFactory implements ThreadFactory {
```

## Architecture & Concepts
The **ChunkWorkerThreadFactory** is a specialized factory responsible for producing worker threads for the server's world generation subsystem. Its primary architectural function is to enforce a critical resource management pattern for computationally expensive world generation tasks.

Instead of creating generic **Thread** objects, this factory produces instances of a private inner class, **ChunkWorker**. Each **ChunkWorker** is intrinsically linked to the lifecycle of a **ChunkGeneratorResource**. Before executing any submitted task, the **ChunkWorker** acquires and initializes a thread-local resource from the **ChunkGenerator**. Upon task completion or failure, it guarantees the release of this resource within a `finally` block.

This pattern is essential for performance and stability. It ensures that expensive-to-create objects used during chunk generation (e.g., noise samplers, data buffers) are reused within a single thread, avoiding the overhead of constant allocation and initialization. It also prevents resource leaks, which could otherwise degrade server performance over time.

Furthermore, the factory implements a robust thread naming scheme, embedding both a factory ID and a thread ID into the name. This is invaluable for debugging, profiling, and identifying performance bottlenecks within specific world generation thread pools. All created threads are configured as daemon threads, preventing them from blocking JVM shutdown.

## Lifecycle & Ownership
- **Creation:** An instance of **ChunkWorkerThreadFactory** is created by the service responsible for bootstrapping the world generation thread pool, typically a high-level **WorldGenerator** or server initialization manager. It is constructed with a reference to the active **ChunkGenerator** and a naming format string.

- **Scope:** The factory's lifetime is bound to the **ExecutorService** it is configured for. It persists as long as the thread pool exists and may need to create new worker threads.

- **Destruction:** The object is eligible for garbage collection once the owning **ExecutorService** is shut down and no longer holds a reference to it. It does not manage any unmanaged resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** This class is stateful. It maintains an immutable reference to a **ChunkGenerator** and a `threadNameFormat`. It also manages two mutable counters:
    - A static **AtomicInteger** (`FACTORY_COUNTER`) to ensure every factory instance has a unique ID across the entire server.
    - An instance-level **AtomicInteger** (`threadCounter`) to ensure each thread produced by this factory has a unique ID within its pool.

- **Thread Safety:** The class is thread-safe. State modification is limited to incrementing atomic counters, which are designed for high-performance concurrent access. The core `newThread` method is frequently called by the **ExecutorService** from multiple threads when scaling up the worker pool, and its design safely handles this scenario.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newThread(Runnable r) | Thread | O(1) | Creates and returns a specialized **ChunkWorker** thread. This is the sole entry point defined by the **ThreadFactory** interface. |

## Integration Patterns

### Standard Usage
This factory is not intended for direct use. It should be supplied to a **java.util.concurrent.ExecutorService** during its construction. The executor will then invoke the `newThread` method internally as needed to populate its thread pool.

```java
// Correctly configure a thread pool for world generation
ChunkGenerator mainGenerator = world.getChunkGenerator();
ThreadFactory workerFactory = new ChunkWorkerThreadFactory(mainGenerator, "WorldGen-Pool-%d-Worker-%d");

// The factory is passed to the executor, which manages the thread lifecycle
ExecutorService worldGenExecutor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors(),
    workerFactory
);

// Submit tasks to the executor, not the factory
worldGenExecutor.submit(new ChunkGenerationTask(position));
```

### Anti-Patterns (Do NOT do this)
- **Direct Thread Management:** Never call `start()` on the **Thread** object returned by `newThread`. The **ExecutorService** is the sole owner and manager of the thread's lifecycle. Manually starting the thread bypasses the pool's scheduling and management logic.

- **Reusing a Single Factory:** Avoid sharing a single factory instance across multiple, logically distinct thread pools. Doing so will merge the thread naming schemes, making debugging and performance analysis significantly more difficult. Each **ExecutorService** should have its own dedicated **ChunkWorkerThreadFactory** instance.

## Data Pipeline
This component does not directly process data. Instead, it establishes the infrastructure for data processing. Its role is to create workers that correctly manage the resources needed for the main world generation pipeline.

> Flow:
> **ExecutorService** needs a new worker -> Invokes `newThread` on **ChunkWorkerThreadFactory** -> Factory creates a **ChunkWorker** -> **ChunkWorker** acquires a thread-local **ChunkGeneratorResource** -> **ChunkWorker** executes a `Runnable` (e.g., a chunk generation task) -> **ChunkWorker** releases the **ChunkGeneratorResource**.

