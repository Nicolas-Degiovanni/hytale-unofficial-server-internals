---
description: Architectural reference for BufferChunkSaver
---

# BufferChunkSaver

**Package:** com.hypixel.hytale.server.core.universe.world.storage
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class BufferChunkSaver implements IChunkSaver {
```

## Architecture & Concepts
The BufferChunkSaver is an abstract base class that forms a critical bridge in the world persistence pipeline. It decouples the high-level, object-oriented representation of a world chunk (Holder<ChunkStore>) from the low-level, byte-oriented storage mechanism. Its primary architectural role is to enforce a contract for saving and removing raw byte buffers, while providing a concrete implementation for the serialization logic.

This class employs the **Template Method Pattern**. The public `saveHolder` method defines the non-varying steps of the persistence algorithm:
1.  Serialize the Holder<ChunkStore> object into a BsonDocument.
2.  Convert the BsonDocument into a raw ByteBuffer.
3.  Delegate the final, storage-specific write operation to the abstract `saveBuffer` method.

This design allows concrete subclasses, such as a file-based or database-backed saver, to focus solely on the mechanics of writing bytes to a medium, without needing to understand the complexities of chunk serialization. All operations are asynchronous, returning a CompletableFuture to prevent blocking the main server thread during I/O-intensive operations.

### Lifecycle & Ownership
-   **Creation:** This class is abstract and cannot be instantiated directly. Concrete subclasses are instantiated by the world management system (e.g., WorldStorage) during server or world initialization, typically based on server configuration.
-   **Scope:** An instance of a BufferChunkSaver subclass lives for the entire duration that its associated world is loaded in memory. It is a long-lived, foundational service for a given world.
-   **Destruction:** The object is eligible for garbage collection when the world is unloaded and all references from the world storage system are released. Subclasses managing native resources like file handles must implement their own cleanup logic.

## Internal State & Concurrency
-   **State:** The internal state is minimal and immutable after construction. It consists of a single `final` reference to a Store<ChunkStore>, which is used during the serialization process. This class does not cache chunk data; it is a stateless processor.
-   **Thread Safety:** This base class is inherently thread-safe due to its immutable state. However, the responsibility for ensuring thread-safe I/O operations lies entirely with the concrete subclasses that implement `saveBuffer` and `removeBuffer`. The asynchronous API contract using CompletableFuture is designed for safe use in a multi-threaded server environment, offloading I/O work from the main game loop.

## API Surface
The public API is designed for high-level interaction by the world storage engine. The protected abstract methods form the contract for implementers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| saveHolder(x, z, holder) | CompletableFuture<Void> | O(S) | Serializes and persists a chunk. S is the size of the chunk data. Asynchronous. |
| removeHolder(x, z) | CompletableFuture<Void> | O(1) | Triggers the asynchronous removal of a chunk at the given coordinates. |
| saveBuffer(x, z, buffer) | CompletableFuture<Void> | Backend-dependent | **Abstract.** The core method subclasses must implement to write bytes to storage. |
| removeBuffer(x, z) | CompletableFuture<Void> | Backend-dependent | **Abstract.** The core method subclasses must implement to remove data from storage. |

## Integration Patterns

### Standard Usage
Developers should not interact with this class directly. The world persistence engine uses a concrete implementation to save chunks that have been marked as dirty. The interaction is managed entirely by higher-level world services.

```java
// Engine-level code (conceptual)
// A concrete implementation is retrieved from the world's service registry.
IChunkSaver saver = world.getStorageService().getChunkSaver();

// The engine calls saveHolder when a chunk needs to be persisted.
Holder<ChunkStore> chunkToSave = world.getChunk(x, z);
saver.saveHolder(x, z, chunkToSave)
     .thenRun(() -> log.info("Chunk saved successfully."))
     .exceptionally(ex -> {
         log.error("Failed to save chunk", ex);
         return null;
     });
```

### Anti-Patterns (Do NOT do this)
-   **Blocking on the Future:** Never call `.get()` or `.join()` on the returned CompletableFuture from a performance-critical thread, such as the main server tick loop. Doing so will freeze the server until the I/O operation completes. Always use asynchronous chaining with methods like `thenRun` or `thenAcceptAsync`.
-   **Incorrect Implementation:** Subclasses must ensure that their implementations of `saveBuffer` and `removeBuffer` are fully non-blocking and thread-safe. A blocking implementation violates the architectural contract of the entire persistence system.

## Data Pipeline
The BufferChunkSaver is a key processing stage in the data flow from in-memory game state to persistent storage.

> Flow:
> In-Memory Chunk (Holder) -> **BufferChunkSaver.saveHolder** -> BSON Serialization -> ByteBuffer -> **Subclass.saveBuffer** -> Physical Storage (e.g., File System, Database)

