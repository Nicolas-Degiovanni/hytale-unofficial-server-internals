---
description: Architectural reference for ChunkStore
---

# ChunkStore

**Package:** com.hypixel.hytale.server.core.universe.world.storage
**Type:** Scoped Singleton

## Definition
```java
// Signature
public class ChunkStore implements WorldProvider {
```

## Architecture & Concepts

The ChunkStore is the central authority for the lifecycle of all chunk data within a single World instance. It acts as a high-performance, asynchronous cache and management layer, abstracting the complexities of chunk loading, generation, saving, and unloading from the rest of the server. It is the bridge between the logical game world and the physical storage or procedural generation systems.

Architecturally, ChunkStore is the heart of the world's data management. It does not contain game logic itself, but rather provides the data containers (chunks) upon which game systems operate. It is built upon Hytale's Entity Component System (ECS) framework, where each chunk is treated as an entity. The ChunkStore instance serves as the "external data" for its own dedicated ECS `Store`, which manages all chunk components like WorldChunk, BlockChunk, and ChunkColumn.

Its core design is asynchronous, revolving around the `CompletableFuture` paradigm. This prevents the main server thread from blocking on expensive I/O operations (loading from disk) or CPU-intensive tasks (world generation). All requests for chunk data that is not immediately available in memory return a future, allowing the calling system to continue its work and react when the data is ready.

The internal state of each chunk is managed by a sophisticated state machine, encapsulated within the private `ChunkLoadState` class. This machine tracks whether a chunk is loaded, being loaded from disk, being generated, has failed, or is on a failure backoff timer. Access to this state is controlled by a `StampedLock`, enabling high-throughput, lock-free optimistic reads while ensuring safety during state transitions.

## Lifecycle & Ownership

-   **Creation:** A ChunkStore is instantiated by its parent `World` object during the world's own initialization sequence. It is fundamentally inseparable from its `World`. The `start` method is subsequently called, which initializes its internal ECS `Store` and registers its core management systems.
-   **Scope:** The ChunkStore instance has a lifetime identical to its parent `World`. It persists for the entire duration of the world's session, from server start or world creation until the world is unloaded.
-   **Destruction:** The `shutdown` method is invoked when the `World` is being unloaded. This triggers a clean shutdown of its internal ECS `Store`, clears all cached chunk state from the `chunks` map, and releases references to its loader and saver subsystems.

## Internal State & Concurrency

-   **State:** The ChunkStore is highly stateful and mutable. Its primary state is the `chunks` map, a `Long2ObjectConcurrentHashMap` that maps a chunk's long index to its corresponding `ChunkLoadState`. This map is the single source of truth for the status of every chunk currently managed by the store. It also holds nullable references to the active `IChunkLoader`, `IChunkSaver`, and `IWorldGen` implementations, which are configured during its startup phase.

-   **Thread Safety:** This class is engineered for high-concurrency environments and is thread-safe.
    -   The top-level `chunks` map is concurrent, allowing safe additions and removals from multiple threads.
    -   Individual chunk state transitions are guarded by a `StampedLock` within the `ChunkLoadState` object. This provides fine-grained locking, preventing contention between operations on different chunks and allowing for high-performance optimistic reads.
    -   Public-facing methods like `getChunkReferenceAsync` are safe to call from any thread. They orchestrate a pipeline that moves work from background I/O or generation threads onto the main world thread for final integration into the game state.
    -   **WARNING:** While requesting data is thread-safe, integrating the final result is not. Methods like `add` and `postLoadChunk` contain `world.debugAssertInTickingThread()` assertions, enforcing that the final attachment of a loaded chunk to the world simulation *must* occur on the main world thread to prevent severe race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChunkReferenceAsync(long index, int flags) | CompletableFuture | O(N) | Asynchronously retrieves a chunk. Triggers loading from disk or world generation if not in memory. N represents the cost of I/O or generation. This is the primary method for accessing chunk data. |
| getChunkReference(long index) | Ref | O(1) | Synchronously retrieves a reference to an already loaded chunk. Returns null if the chunk is not in memory. **WARNING:** Does not trigger loading. |
| remove(Ref reference, RemoveReason reason) | void | O(1) | Schedules a chunk to be removed from the store. Must be called from the main world thread. |
| setGenerator(IWorldGen generator) | void | O(1) | Sets or replaces the world generation provider. Can be used to hot-swap generators. |
| waitForLoadingChunks() | void | O(N) | Blocks the calling thread until all currently pending chunk load/generation futures are complete. Used for safe shutdown and world saving. |
| isChunkOnBackoff(long index, ...) | boolean | O(1) | Checks if a chunk has recently failed to load and is in a temporary backoff period to prevent log spam and repeated failures. |

## Integration Patterns

### Standard Usage

The correct pattern for interacting with the ChunkStore is asynchronous. A system should request a chunk and chain subsequent logic to the resulting `CompletableFuture` to be executed when the chunk is ready.

```java
// Standard asynchronous request for a chunk that needs to be ticked
ChunkStore chunkStore = world.getChunkStore();
long chunkIndex = ChunkUtil.indexChunk(10, -5);

CompletableFuture<Ref<ChunkStore>> future = chunkStore.getChunkReferenceAsync(chunkIndex, ChunkFlag.TICKING);

future.thenAcceptAsync(chunkRef -> {
    if (chunkRef != null && chunkRef.isValid()) {
        // This code executes on the main world thread once the chunk is loaded and ready.
        // It is now safe to access the chunk's components.
        WorldChunk wc = chunkStore.getStore().getComponent(chunkRef, WorldChunk.getComponentType());
        if (wc != null) {
            // Perform game logic
        }
    } else {
        // Handle cases where the chunk failed to load or was unloaded before completion.
    }
}, world); // Ensure execution on the world's task scheduler
```

### Anti-Patterns (Do NOT do this)

-   **Blocking on Futures:** Never call `future.join()` or `future.get()` on a future returned by `getChunkReferenceAsync` from the main server thread. This will freeze the entire world tick loop until the chunk I/O or generation is complete, causing catastrophic performance degradation.
-   **Assuming Synchronous Access:** Do not use `getChunkReference` and assume it will always return a non-null value. This method does not trigger loading and is only suitable for accessing chunks that are guaranteed to be in memory.
-   **Cross-Thread Component Modification:** Once a chunk is loaded and its future completes, do not modify its components from any thread other than the main world thread. All game logic and component mutations must be synchronized with the world's tick.

## Data Pipeline

The ChunkStore orchestrates a complex data pipeline to load a chunk from storage or generate it from scratch. The flow is designed to maximize concurrency and minimize impact on the main game loop.

> Flow:
> System Call to `getChunkReferenceAsync` -> **ChunkStore** checks `chunks` map -> If absent, creates `ChunkLoadState` and `CompletableFuture` -> Task submitted to `IChunkLoader` (I/O Thread Pool) OR `IWorldGen` (Generator Thread Pool) -> Raw chunk data is loaded/generated -> `preLoadChunkAsync` processes data off-thread -> `postLoadChunk` is scheduled on the main World thread -> **ChunkStore** integrates the chunk into its ECS `Store` -> Original `CompletableFuture` is completed -> Dependent game systems are notified.

