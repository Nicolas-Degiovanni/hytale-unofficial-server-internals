---
description: Architectural reference for BufferChunkLoader
---

# BufferChunkLoader

**Package:** com.hypixel.hytale.server.core.universe.world.storage
**Type:** Abstract Base Class / Strategy

## Definition
```java
// Signature
public abstract class BufferChunkLoader implements IChunkLoader {
```

## Architecture & Concepts
The BufferChunkLoader is an abstract base class that forms a critical component of the server's world persistence layer. It establishes a standardized, asynchronous pipeline for loading chunk data from a raw binary source and hydrating it into a fully realized game object.

Its primary architectural purpose is to enforce the **Template Method Pattern**. The `loadHolder` method defines a fixed, high-level algorithm for chunk loading:
1.  Asynchronously fetch a raw ByteBuffer.
2.  Deserialize the buffer from BSON into a component Holder.
3.  Hydrate the WorldChunk component with its world context and coordinates.

This pattern decouples the complex, generic logic of deserialization and object hydration from the specific, low-level details of data retrieval (e.g., reading from a file, querying a database). Concrete subclasses, such as a `FileChunkLoader` or `DatabaseChunkLoader`, are only required to implement the `loadBuffer` method, providing the raw bytes for a given chunk coordinate.

This class acts as the bridge between the physical storage layer and the in-memory representation of the game world.

### Lifecycle & Ownership
-   **Creation:** This abstract class is never instantiated directly. Concrete subclasses are instantiated by a higher-level world management service, such as a `WorldStorageManager`, during the initialization of a specific game world. The required `Store<ChunkStore>` dependency is injected at this time, linking the loader to the world's central component data store.
-   **Scope:** An instance of a BufferChunkLoader subclass exists for the entire duration that its corresponding world is loaded on the server. It is a long-lived, session-scoped object.
-   **Destruction:** The object is eligible for garbage collection when the world it services is unloaded and all references from the world management service are released. Subclasses that manage native resources like file handles or database connections may require an explicit shutdown hook, though this is not part of the base contract.

## Internal State & Concurrency
-   **State:** The BufferChunkLoader is effectively stateless. Its only internal field, `store`, is a final reference injected during construction. It does not cache chunk data or maintain any mutable state related to load operations. Each call to `loadHolder` is an independent, self-contained operation.

-   **Thread Safety:** This class is inherently thread-safe. The design is centered around the `CompletableFuture` API, making it ideal for a multi-threaded server environment.
    -   The `loadBuffer` method is expected to perform blocking I/O on a worker thread.
    -   The subsequent deserialization and hydration logic, defined within the `thenApplyAsync` block, is explicitly scheduled to run on a separate thread from the default fork-join pool. This prevents computationally expensive deserialization from blocking the I/O thread, ensuring high throughput.

    **WARNING:** Subclass implementations of `loadBuffer` must be thread-safe if they share resources (e.g., a single database connection pool).

## API Surface
The public contract is focused on the asynchronous loading pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadBuffer(int x, int z) | CompletableFuture<ByteBuffer> | I/O Bound | **Abstract.** Fetches the raw binary data for a chunk. Subclasses must implement the specific I/O logic. |
| loadHolder(int x, int z) | CompletableFuture<Holder<ChunkStore>> | I/O + Deserialization | The primary entry point. Orchestrates the full, asynchronous loading and deserialization of a chunk. |
| getStore() | Store<ChunkStore> | O(1) | Returns the component store associated with the world this loader services. |

## Integration Patterns

### Standard Usage
A `WorldStorage` service uses a concrete implementation to load chunks asynchronously, chaining subsequent game logic to the returned future. This ensures the main server thread is never blocked.

```java
// WorldStorageManager.java
IChunkLoader chunkLoader = new FileChunkLoader(world.getChunkStore()); // Injected or created
int chunkX = 10;
int chunkZ = -5;

// Asynchronously request the chunk load
chunkLoader.loadHolder(chunkX, chunkZ).thenAccept(chunkHolder -> {
    if (chunkHolder != null) {
        // This callback executes on a worker thread when the chunk is ready
        world.addChunk(chunkHolder);
    } else {
        // Handle case where chunk data does not exist
        world.generateNewChunk(chunkX, chunkZ);
    }
});
```

### Anti-Patterns (Do NOT do this)
-   **Blocking the Main Thread:** Never call `.join()` or `.get()` on the returned `CompletableFuture` from the main server tick loop. Doing so will freeze the server until the disk I/O and deserialization are complete, causing catastrophic performance degradation.
-   **Incorrect Threading in Subclasses:** Implementing `loadBuffer` in a way that blocks the calling thread (if it's not already a dedicated I/O thread) defeats the purpose of the asynchronous design. All I/O operations must be offloaded from critical server threads.

## Data Pipeline
The BufferChunkLoader orchestrates a clear, one-way flow of data from persistent storage to the live game world.

> Flow:
> Physical Storage (File, DB) → Subclass `loadBuffer` → Raw `ByteBuffer` → **BufferChunkLoader** `loadHolder` → `BsonUtil` → `BsonDocument` → `ChunkStore.REGISTRY` → Hydrated `Holder<ChunkStore>` → Game World System

