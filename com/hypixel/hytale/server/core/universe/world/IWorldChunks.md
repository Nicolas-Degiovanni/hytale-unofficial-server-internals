---
description: Architectural reference for IWorldChunks
---

# IWorldChunks

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Interface

## Definition
```java
// Signature
@Deprecated
public interface IWorldChunks extends IChunkAccessorSync<WorldChunk>, IWorldChunksAsync {
```

**WARNING:** This entire interface is marked as **Deprecated**. It represents a legacy, blocking-access pattern for world data. New development should exclusively use asynchronous APIs like IWorldChunksAsync to prevent stalling the main server thread.

## Architecture & Concepts

IWorldChunks is a contract that serves as a critical, albeit deprecated, bridge between the synchronous demands of the core game simulation and the asynchronous reality of chunk I/O. Its primary architectural purpose is to provide a blocking, synchronous-style API facade over an underlying asynchronous chunk loading and generation pipeline.

By inheriting from both IChunkAccessorSync and IWorldChunksAsync, it exposes two conflicting worldviews:
1.  **Synchronous Access (IChunkAccessorSync):** Core game logic, such as entity physics or block updates, often requires immediate access to chunk data to proceed. The `getChunk` method provides this guarantee by blocking execution until the requested chunk is available.
2.  **Asynchronous Operations (IWorldChunksAsync):** Loading chunks from disk or generating new terrain is a slow, I/O-bound operation. The underlying system performs these tasks on worker threads to avoid freezing the server, communicating results via a CompletableFuture.

The interface's default methods, particularly `waitForFutureWithoutLock`, contain the complex and hazardous logic required to reconcile these two models, effectively making the main thread wait while still allowing other essential tasks to be processed.

### Lifecycle & Ownership

As an interface, IWorldChunks defines a contract and has no lifecycle of its own. The lifecycle described here applies to the concrete classes that implement it.

-   **Creation:** An object implementing IWorldChunks is instantiated by its parent World object during the world's initialization phase. It is a core component of a server-side World.
-   **Scope:** The instance persists for the entire lifetime of the World it manages. It is intrinsically tied to a single dimension or world instance.
-   **Destruction:** The instance is eligible for garbage collection when its parent World is unloaded and all references to it are released.

## Internal State & Concurrency

-   **State:** This interface defines no state. However, any concrete implementation is expected to be highly stateful, managing an in-memory cache of loaded WorldChunk objects to satisfy synchronous requests quickly.

-   **Thread Safety:** The contract is fundamentally **not thread-safe** and is designed with a strict thread-affinity model. All methods are intended to be called from the main world thread. The `isInThread` method exists as a guard to enforce this assumption.

    **WARNING:** The concurrency model is extremely fragile. The `waitForFutureWithoutLock` method implements a custom spin-wait loop that periodically releases and acquires the global `AssetRegistry.ASSET_LOCK`. This is a specialized mechanism to prevent deadlocks where the main thread is waiting for a chunk from a worker thread, which in turn may be waiting for an asset lock held by the main thread. Calling these methods from any thread other than the designated world thread will bypass this safety mechanism and can easily lead to deadlocks or severe performance degradation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChunk(long index) | WorldChunk | O(N) (Blocking I/O) | Retrieves a ticking chunk. Blocks the calling thread until the chunk is loaded from disk or generated. Returns null on failure. |
| getNonTickingChunk(long index) | WorldChunk | O(N) (Blocking I/O) | Retrieves a non-ticking (data-only) chunk. Blocks the calling thread until the chunk is loaded. |
| isInThread() | boolean | O(1) | Returns true if the current thread is the main world thread, false otherwise. |
| consumeTaskQueue() | void | O(N) | **Deprecated.** Processes pending tasks on the world thread. Used internally during blocking waits. |
| waitForFutureWithoutLock(future) | T | O(N) (Blocking) | A specialized blocking wait that avoids deadlocks with the AssetRegistry. **For internal use only.** |

## Integration Patterns

### Standard Usage

The intended, though now deprecated, usage is to retrieve the provider from a World instance and request a chunk directly. This pattern blocks the thread until the data is available.

```java
// WARNING: This is a deprecated, blocking pattern.
// Prefer asynchronous alternatives for new code.

IWorldChunks chunkProvider = serverWorld.getChunkProvider();
long chunkIndex = ChunkPosition.toIndex(x, y, z);

// This call will freeze the current thread until the chunk is loaded.
WorldChunk chunk = chunkProvider.getChunk(chunkIndex);

if (chunk != null) {
    // Perform immediate, synchronous operations on the chunk data.
    chunk.setBlock(localX, localY, localZ, newBlockState);
}
```

### Anti-Patterns (Do NOT do this)

-   **Usage in New Systems:** Do not use this interface for any new feature development. The blocking nature is a primary source of server lag ("tick spikes"). Always use the asynchronous methods provided by IWorldChunksAsync.
-   **Calling from Worker Threads:** Never call `getChunk` or `getNonTickingChunk` from a non-world thread. The internal deadlock prevention logic in `waitForFutureWithoutLock` is only effective on the main thread. On other threads, it degrades to a simple, inefficient `future.join()`, which can easily cause server-wide deadlocks if that thread needs a resource held by the main thread.
-   **Wrapping in External Locks:** Do not call methods on this interface while holding your own locks. The internal lock-juggling can lead to unpredictable lock ordering and introduce deadlocks that are extremely difficult to debug.

## Data Pipeline

The data flow for a synchronous `getChunk` call demonstrates how the interface masks underlying asynchronous work.

> Flow:
> Main World Thread -> `getChunk(index)` -> **IWorldChunks Implementation** -> Check In-Memory Cache -> (Cache Miss) -> Request chunk via `getChunkAsync` -> I/O Worker Thread Pool (Reads from Disk/DB) -> `CompletableFuture<WorldChunk>` is returned -> **IWorldChunks Implementation** enters `waitForFutureWithoutLock` spin-wait -> Main World Thread processes other tasks via `consumeTaskQueue` while waiting -> I/O Worker Thread completes Future -> Main World Thread exits wait and receives `WorldChunk`

