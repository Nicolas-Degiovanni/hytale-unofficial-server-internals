---
description: Architectural reference for ChunkThreadPoolExecutor
---

# ChunkThreadPoolExecutor

**Package:** com.hypixel.hytale.server.worldgen.util
**Type:** Specialized Utility

## Definition
```java
// Signature
public final class ChunkThreadPoolExecutor extends ThreadPoolExecutor {
```

## Architecture & Concepts
The ChunkThreadPoolExecutor is a specialized, lifecycle-aware extension of the standard Java ThreadPoolExecutor. It is the primary engine for executing asynchronous world generation tasks on the Hytale server.

Its core architectural purpose is not to reinvent thread pooling but to augment the standard Java implementation with Hytale-specific observability and lifecycle management. It achieves this through two key additions:

1.  **Generation Counter:** A static, atomic counter assigns a unique, sequential ID to every executor instance (e.g., ChunkGenerator-0, ChunkGenerator-1). This is critical for logging and diagnostics, allowing developers to disambiguate logs from different world generation sessions or parallel generation processes within the same server instance.

2.  **Shutdown Hook:** The constructor requires a Runnable `shutdownHook`. This provides a clean, decoupled mechanism for the system that creates the executor to be notified upon its complete termination. This is a fundamental pattern for graceful shutdown, ensuring that dependent resources are released only after all world generation tasks are finished.

By extending ThreadPoolExecutor, this class inherits a robust and highly configurable task execution framework while layering on top the necessary hooks for integration into the server's broader service management lifecycle.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a high-level world generation management service when a world is loaded or a new, large-scale generation process is initiated. It is never created directly by game logic. The creating service is responsible for providing all configuration, including thread counts, queue implementation, and the critical shutdown hook.
-   **Scope:** The lifecycle of a ChunkThreadPoolExecutor is tightly bound to a specific world generation session. It is a session-scoped resource, not a global or static pool.
-   **Destruction:** The owning service initiates shutdown by calling the standard `shutdown()` or `shutdownNow()` methods. The executor will stop accepting new tasks and complete its work. The final step in its lifecycle is the invocation of the protected `terminated()` method, which in turn executes the provided `shutdownHook`, signaling its complete death to the owning system.

## Internal State & Concurrency
-   **State:** This class is highly stateful, managing a pool of worker threads, a work queue, and various internal statistics, all inherited from its parent. The state it adds—the `generation` ID and the `shutdownHook` reference—is immutable after construction.
-   **Thread Safety:** This class is thread-safe. All public methods for submitting tasks and initiating shutdown are concurrency-safe, as guaranteed by the Java Concurrency Framework. The static `GENERATION_COUNTER` uses an AtomicInteger to ensure safe instantiation from multiple threads, though in practice this is rare.

## API Surface
The primary API contract is defined by the constructor for configuration and the inherited methods from ThreadPoolExecutor for operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ChunkThreadPoolExecutor(...) | constructor | O(1) | Creates and configures a new executor. Requires all standard pool parameters plus a non-null shutdown hook. |
| execute(Runnable) | void | O(1) | Inherited. Submits a task for execution. Throws RejectedExecutionException if the pool is shutting down or the queue is full. |
| shutdown() | void | O(1) | Inherited. Initiates a graceful shutdown. No new tasks are accepted, but queued tasks will be executed. |
| terminated() | protected void | O(1) | Overridden hook. Executes the provided shutdown hook after all threads have terminated. **WARNING:** This is a framework hook, not for external invocation. |

## Integration Patterns

### Standard Usage
The executor is created and managed by a service that oversees a world's lifecycle. The service provides a lambda or method reference as the shutdown hook to perform cleanup.

```java
// Within a WorldManager or similar service
Runnable cleanupTask = () -> {
    // This code runs AFTER the executor is fully terminated
    log.info("World generation for world " + worldId + " is complete.");
    this.releaseWorldGenResources(worldId);
};

// The ThreadFactory is crucial for naming threads for debugging
ThreadFactory factory = new NamedThreadFactory("hytale-worldgen-%d");

BlockingQueue<Runnable> workQueue = new LinkedBlockingQueue<>();

// Creation is complex and handled by a factory or builder
ChunkThreadPoolExecutor worldGenExecutor = new ChunkThreadPoolExecutor(
    4, 8, 60L, TimeUnit.SECONDS, workQueue, factory, cleanupTask
);

// During gameplay, tasks are submitted
worldGenExecutor.execute(new GenerateChunkTask(position));

// When the world is unloaded, initiate shutdown
worldGenExecutor.shutdown();
```

### Anti-Patterns (Do NOT do this)
-   **Global Singleton:** Do not attempt to use a single, static instance of this executor for all worlds. Its lifecycle is designed to be tied to a specific session, and its shutdown hook is session-specific.
-   **Blocking Shutdown Hooks:** The `shutdownHook` Runnable must be non-blocking and should not throw exceptions. Placing long-running or fallible operations in the hook can stall the server shutdown sequence.
-   **Ignoring Shutdown:** Failing to call `shutdown()` on this executor when its associated world is unloaded constitutes a major resource leak. The worker threads will never terminate.

## Data Pipeline
This component acts as a task processor, not a data transformer. Its input is a unit of work (a Runnable), and its output is the side effect of that work's execution.

> Flow:
> World Service (requests chunk) -> GenerateChunkTask (created) -> **ChunkThreadPoolExecutor**.execute() -> Worker Thread (executes task) -> Chunk Data (written to world cache/disk)

