---
description: Architectural reference for IChunkSaver
---

# IChunkSaver

**Package:** com.hypixel.hytale.server.core.universe.world.storage
**Type:** Interface / Service Contract

## Definition
```java
// Signature
public interface IChunkSaver extends Closeable {
```

## Architecture & Concepts
The IChunkSaver interface defines the essential contract for the asynchronous persistence of world chunk data. It serves as a critical abstraction layer, decoupling the server's in-memory world representation from the underlying physical storage mechanism, such as region files or a database.

This component sits at the boundary between the game logic and the I/O subsystem. Its primary architectural purpose is to prevent the main server thread from blocking on slow disk operations. Every persistence method returns a CompletableFuture, signaling that the operation will be executed by a separate I/O worker thread pool. This design is fundamental to maintaining high server performance and avoiding tick-rate degradation (lag) caused by world saving.

Implementations of this interface are responsible for serialization, data compression, and managing the physical layout of chunk data on disk.

### Lifecycle & Ownership
As an interface, IChunkSaver itself does not have a lifecycle. The following describes the lifecycle of a concrete implementation.

-   **Creation:** An implementation is instantiated by the WorldStorage service when a world is first loaded into memory. The specific implementation is chosen based on world configuration.
-   **Scope:** The object's lifetime is tightly coupled to the server-side representation of a world. It persists as long as the world is loaded and active.
-   **Destruction:** The close method, inherited from the Closeable interface, is invoked when the world is unloaded or during a graceful server shutdown. This is a critical step to ensure all buffered writes are flushed to the storage medium, preventing data loss.

## Internal State & Concurrency
-   **State:** The interface is stateless. However, any concrete implementation is expected to be highly stateful, managing internal write buffers, file handles, pending operation queues, and potentially a cache of recently accessed chunk locations on disk.
-   **Thread Safety:** Implementations of this contract **must be thread-safe**. Methods like saveHolder will be called concurrently from multiple world-management threads. It is the responsibility of the implementation to serialize access to the underlying storage medium, typically through internal locking, a dedicated single-threaded executor, or lock-free queueing mechanisms. Callers can safely submit save operations from any thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| saveHolder(int, int, Holder) | CompletableFuture<Void> | I/O Bound | Asynchronously saves or overwrites the chunk data at the given chunk coordinates (x, z). The operation is not complete until the future resolves. |
| removeHolder(int, int) | CompletableFuture<Void> | I/O Bound | Asynchronously deletes the chunk data at the given chunk coordinates. This is used for world pruning or regeneration operations. |
| getIndexes() | LongSet | I/O Bound | Synchronously reads the storage index to discover all currently existing chunk coordinates. Throws IOException on failure. **Warning:** This can be a blocking operation and should not be called on the main server thread. |
| flush() | void | I/O Bound | Synchronously forces all buffered or pending save operations to be written to the physical storage. Throws IOException on failure. |

## Integration Patterns

### Standard Usage
The IChunkSaver is managed by a higher-level service, typically WorldStorage. Game logic should not interact with it directly but rather trigger saves through the World object, which delegates the I/O operation.

```java
// Example of an internal engine system using the saver
// This code would exist within a WorldStorage or similar class.

IChunkSaver chunkSaver = this.getActiveSaver();
Holder<ChunkStore> chunkToSave = world.getChunkHolder(chunkX, chunkZ);

// Offload the save operation without blocking the current thread
CompletableFuture<Void> saveFuture = chunkSaver.saveHolder(chunkX, chunkZ, chunkToSave);

// Handle potential I/O errors asynchronously
saveFuture.exceptionally(error -> {
    log.error("Failed to save chunk at {}, {}: {}", chunkX, chunkZ, error);
    return null; // Return null to signify the error was handled
});
```

### Anti-Patterns (Do NOT do this)
-   **Blocking The Main Thread:** Never call get or join on the CompletableFuture returned by saveHolder from the main server tick loop. This will freeze the server until the I/O operation completes, causing catastrophic performance degradation.
-   **Ignoring The Close Method:** Failure to call close on an IChunkSaver instance during world unload or server shutdown will result in data loss. Any writes buffered in memory will be discarded.
-   **Frequent Flushing:** Calling flush excessively can degrade storage performance by defeating the benefits of I/O buffering and scheduling. It should only be used during critical save points, such as before a server shutdown.

## Data Pipeline
The IChunkSaver is the final stage in the data pipeline for persisting world state from memory to a durable medium.

> Flow:
> In-Memory ChunkStore (Modified by game logic) -> World Ticking System flags chunk as "dirty" -> WorldStorage Service -> **IChunkSaver.saveHolder()** -> I/O Worker Thread serializes and compresses data -> Physical Storage (e.g., Region File)

