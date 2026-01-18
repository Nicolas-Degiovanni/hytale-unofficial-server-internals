---
description: Architectural reference for WorldConfigSaveSystem
---

# WorldConfigSaveSystem

**Package:** com.hypixel.hytale.server.core.universe.system
**Type:** System Component

## Definition
```java
// Signature
public class WorldConfigSaveSystem extends DelayedSystem<EntityStore> {
```

## Architecture & Concepts
The WorldConfigSaveSystem is a specialized, time-based system responsible for orchestrating the persistence of a server world's state to disk. As a subclass of DelayedSystem, it does not execute on every server tick. Instead, it operates on a fixed timer, defaulting to every 10 seconds, to prevent the performance overhead of disk I/O from impacting the main simulation loop.

Its primary architectural role is to act as a high-level coordinator for saving critical world data. It triggers persistence operations on three core components:
1.  **ChunkStore:** Saves all modified chunk data.
2.  **EntityStore:** Saves all entity data.
3.  **WorldConfig:** Saves the world's metadata and configuration, but only if it has been marked as changed.

The system leverages Java's CompletableFuture API to perform these I/O operations asynchronously. This design allows multiple data stores to be saved in parallel, potentially reducing the total time required for a full world save.

**WARNING:** The implementation of `delayedTick` uses `join()` to block until all asynchronous save operations are complete. While the I/O itself runs on other threads, this blocking call will stall the system scheduler's thread until the save is finished. This can lead to periodic server-side lag spikes if disk I/O is slow.

## Lifecycle & Ownership
-   **Creation:** An instance of WorldConfigSaveSystem is created and registered with the server's system scheduler during the initialization of a `World` instance. It is an integral part of the server-side world management infrastructure.
-   **Scope:** The system's lifecycle is tightly bound to the `World` it manages. It persists as long as the world is loaded and running on the server.
-   **Destruction:** The system is deregistered and garbage collected when the corresponding `World` is unloaded, either during a graceful server shutdown or when a world is explicitly taken offline.

## Internal State & Concurrency
-   **State:** This class is stateless. It does not maintain any internal data or cache between invocations. Its behavior is determined entirely by the state of the `World` object provided at runtime.
-   **Thread Safety:** The `delayedTick` method is invoked by the system scheduler, which typically operates in a single-threaded manner per world. The core logic within `saveWorldConfigAndResources` is designed for concurrency. It dispatches save operations to run on a background I/O thread pool via CompletableFuture. The static nature of `saveWorldConfigAndResources` makes it safe to call from any thread, provided the passed `World` object and its underlying stores are themselves thread-safe for read and save operations.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| delayedTick(dt, systemIndex, store) | void | I/O Bound | Overridden entry point for the system scheduler. Blocks until a full world save is complete. |
| saveWorldConfigAndResources(world) | CompletableFuture<Void> | I/O Bound | Static utility that initiates an asynchronous save of the world's chunks, entities, and configuration. |

## Integration Patterns

### Standard Usage
This system is not intended for direct invocation by game logic or plugin developers. It is automatically managed by the server's core engine. The engine's system scheduler is responsible for calling `delayedTick` at the configured interval.

Forcing a world save should be done via higher-level server commands or APIs, which may in turn use the static `saveWorldConfigAndResources` method.

```java
// Example of a high-level API forcing a save
// This code would exist within the server core, not typical user code.

World targetWorld = Universe.get().getWorld("myworld");
if (targetWorld != null) {
    // Initiate the save but do not block the main thread
    WorldConfigSaveSystem.saveWorldConfigAndResources(targetWorld)
        .thenRun(() -> System.out.println("World save complete."))
        .exceptionally(ex -> {
            System.err.println("World save failed: " + ex.getMessage());
            return null;
        });
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Do not call `delayedTick` directly. This bypasses the timing and context management of the system scheduler and can lead to unpredictable save behavior.
-   **Blocking the Main Thread:** Do not call `.join()` or `.get()` on the Future returned by `saveWorldConfigAndResources` from a performance-critical thread, such as the main server tick loop. This will freeze the server until the I/O operations complete.

## Data Pipeline
This system represents a data-out pipeline, moving in-memory game state to persistent storage.

> Flow:
> In-Memory `World` State (Chunks, Entities, Config) -> `delayedTick` Trigger -> **WorldConfigSaveSystem** -> Parallel Asynchronous Save Operations (CompletableFuture) -> Filesystem (Disk)

