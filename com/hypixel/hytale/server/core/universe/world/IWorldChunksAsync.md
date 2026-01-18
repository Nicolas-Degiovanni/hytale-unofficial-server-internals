---
description: Architectural reference for IWorldChunksAsync, the deprecated contract for asynchronous world chunk access.
---

# IWorldChunksAsync

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Interface / Contract

## Definition
```java
// Signature
@Deprecated
public interface IWorldChunksAsync {
    // Methods defined here
}
```

## Architecture & Concepts

The IWorldChunksAsync interface defines a contract for retrieving world chunk data without blocking the calling thread. It represents a critical, albeit deprecated, component in the server's world data management system, designed to offload expensive chunk loading and generation operations from the main server loop.

The core architectural primitive is the Java CompletableFuture. By returning a future, implementations of this interface promise to deliver a WorldChunk object at a later time. This decouples the *request* for a chunk from its *fulfillment*, enabling high-performance, non-blocking I/O and world generation. This pattern is essential for maintaining server responsiveness under heavy load.

A key concept exposed by this API is the distinction between a standard chunk and a *non-ticking* chunk.
*   **Ticking Chunks:** Fully processed chunks where entity simulation, block updates, and other game logic are active. These are computationally expensive.
*   **Non-Ticking Chunks:** Chunks loaded into memory primarily for read-only access, such as serialization for network transport or for procedural generation queries that need to inspect neighboring data. They do not process entity logic, offering a significant performance optimization.

**WARNING:** This interface is marked as **Deprecated**. Its usage is strongly discouraged in new systems. A more modern, robust asynchronous pattern has superseded this contract. Existing code should be migrated away from this interface to the new standard.

### Lifecycle & Ownership

As an interface, IWorldChunksAsync defines a contract and has no lifecycle itself. The lifecycle described here pertains to the objects that *implement* this interface, typically a central World or WorldManager class.

*   **Creation:** The implementing object is instantiated once per world, during the server's world initialization sequence. It is a foundational service for the world it manages.
*   **Scope:** The service object persists for the entire lifetime of a loaded world. Its internal caches and thread pools are active as long as the world is active.
*   **Destruction:** The object and its resources are released and made eligible for garbage collection only when its associated world is fully unloaded from the server.

## Internal State & Concurrency

*   **State:** Any class implementing this interface is expected to be highly stateful. It must manage an internal cache of loaded chunks, a queue of pending chunk requests, and a reference to the thread pool responsible for chunk I/O and generation.
*   **Thread Safety:** This contract is designed to be fully thread-safe. Implementations must guarantee that all methods can be called from any thread (e.g., main server thread, network I/O threads, AI behavior threads) without external synchronization. This is typically achieved using concurrent data structures and atomic operations for managing the chunk cache and request queue.

## API Surface

The public contract focuses exclusively on initiating asynchronous chunk retrieval operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChunkAsync(long index) | CompletableFuture | O(1) | Asynchronously requests a full, ticking chunk by its packed coordinate index. The call returns immediately. |
| getNonTickingChunkAsync(long index) | CompletableFuture | O(1) | Asynchronously requests a non-ticking chunk by its packed coordinate index. Optimized for read-only access. |
| getChunkAsync(int x, int z) | CompletableFuture | O(1) | Convenience overload. Converts x, z coordinates into a packed index and calls the primary method. |
| getNonTickingChunkAsync(int x, int z) | CompletableFuture | O(1) | Convenience overload for requesting a non-ticking chunk with x, z coordinates. |

## Integration Patterns

### Standard Usage

The correct pattern is to request a chunk and attach a callback to the returned CompletableFuture. This ensures the main thread is never blocked waiting for the chunk to load.

```java
// How a developer should normally use this
IWorldChunksAsync chunkProvider = world.getChunkProvider();
CompletableFuture<WorldChunk> futureChunk = chunkProvider.getChunkAsync(10, -5);

// Chain subsequent logic to execute upon completion
futureChunk.thenAccept(chunk -> {
    // This code executes on the completion thread once the chunk is loaded
    System.out.println("Chunk is loaded at: " + chunk.getPosition());
    // ... process the chunk data
});
```

### Anti-Patterns (Do NOT do this)

*   **Blocking on a Future:** Calling `join()` or `get()` on the returned CompletableFuture from a performance-critical thread (like the main server tick loop) is a severe anti-pattern. It completely negates the benefit of the asynchronous API and will cause server stalls.
*   **Using a Deprecated API:** The most critical anti-pattern is using this interface in any new feature development. All new code must use the modern, replacement API for world data access.
*   **Ignoring Completion Failures:** Failing to handle exceptions on the CompletableFuture chain (using `exceptionally`) can lead to silent failures where chunk data is never processed.

## Data Pipeline

The flow for a chunk request involves multiple threads and systems, coordinated by the CompletableFuture.

> Flow:
> Game Logic Thread -> **IWorldChunksAsync.getChunkAsync()** -> [Request added to internal queue] -> Chunk Worker Thread Pool -> (Disk I/O or World Generation) -> [CompletableFuture is completed] -> Callback Execution on Completion Thread

