---
description: Architectural reference for IChunkLoader
---

# IChunkLoader

**Package:** com.hypixel.hytale.server.core.universe.world.storage
**Type:** Contract Interface

## Definition
```java
// Signature
public interface IChunkLoader extends Closeable {
```

## Architecture & Concepts
The IChunkLoader interface defines the essential contract for asynchronously loading persistent chunk data into an in-memory representation. It serves as a critical abstraction layer between the server's world management system and the underlying physical storage medium, such as region files or a database.

This component is central to the server's world streaming and data persistence architecture. By defining loading as an asynchronous operation returning a CompletableFuture, the design ensures that I/O-bound disk operations do not block the main server thread, preventing server stalls and maintaining performance. The loader is responsible for deserializing raw chunk data from a persistent source into a usable ChunkStore object, which can then be integrated into the live world simulation.

Implementations of this interface are expected to manage low-level details like file handles, data compression, and storage format parsing.

### Lifecycle & Ownership
As an interface, IChunkLoader itself has no lifecycle. The following pertains to its concrete implementations.

-   **Creation:** An IChunkLoader implementation is typically instantiated by a higher-level storage manager (e.g., a WorldStorageService) when a specific world is being loaded into memory. It is configured with the path to the world's data files.
-   **Scope:** The object's lifetime is tightly coupled to the server-side representation of a world. It persists as long as the world is active and loaded on the server.
-   **Destruction:** The interface extends Closeable. The close method **must** be invoked when the world is unloaded. This is a critical step to release underlying system resources, such as file handles, and to ensure any pending writes are flushed to disk. Failure to call close will result in resource leaks.

## Internal State & Concurrency
-   **State:** The interface is stateless. However, any non-trivial implementation will be highly stateful, managing file pointers, region file caches, and metadata indexes. This internal state is inherently mutable.
-   **Thread Safety:** Implementations of IChunkLoader **must be thread-safe**. The asynchronous nature of the API implies that loadHolder may be called concurrently from multiple worker threads (e.g., for different players moving into separate unloaded areas simultaneously). Implementations are responsible for synchronizing access to shared resources like files to prevent data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadHolder(int x, int z) | CompletableFuture | I/O Bound | Asynchronously requests the loading of a chunk at the given chunk coordinates. Returns a future that completes with the ChunkStore holder or fails with an exception. |
| getIndexes() | LongSet | I/O Bound | Retrieves a set of all chunk coordinates that exist in the persistent storage. This can be a blocking I/O operation and should be used with caution. Throws IOException on failure. |
| close() | void | I/O Bound | Releases all resources held by the loader, such as open file handles. After this is called, the object is no longer usable. |

## Integration Patterns

### Standard Usage
The IChunkLoader is typically managed by a world or region manager. It is not intended for direct use by game logic systems. The standard pattern involves requesting a chunk and chaining subsequent logic to the resulting CompletableFuture to be executed upon completion.

```java
// A WorldManager requests a chunk from its loader
IChunkLoader chunkLoader = world.getStorage().getChunkLoader();

// Asynchronously load and process the chunk without blocking the main thread
chunkLoader.loadHolder(10, -5).thenAccept(chunkStoreHolder -> {
    // This code executes on a worker thread once the chunk is loaded
    world.integrateChunk(chunkStoreHolder.get());
}).exceptionally(error -> {
    // Handle I/O errors or cases where the chunk doesn't exist
    log.error("Failed to load chunk", error);
    return null;
});
```

### Anti-Patterns (Do NOT do this)
-   **Blocking on the Future:** Calling get() on the CompletableFuture returned by loadHolder from the main server thread is a critical performance anti-pattern. This will freeze the server tick loop until the disk I/O completes.
-   **Ignoring the Closeable Contract:** Failing to call close() on the loader when a world is unloaded will lead to severe resource leaks, potentially preventing the server from opening new files or even causing world data to become corrupted.
-   **Concurrent Index Access:** Calling getIndexes repeatedly is inefficient. The set of existing chunks should be fetched once during world initialization and cached by the caller.

## Data Pipeline
The IChunkLoader is a foundational component in the data flow from persistent storage to the live game world.

> Flow:
> World Manager Chunk Request -> **IChunkLoader.loadHolder()** -> I/O Worker Thread Pool -> Disk Read & Deserialization -> CompletableFuture Completion -> In-Memory ChunkStore -> World Simulation

