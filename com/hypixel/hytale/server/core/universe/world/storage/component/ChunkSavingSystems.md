---
description: Architectural reference for ChunkSavingSystems
---

# ChunkSavingSystems

**Package:** com.hypixel.hytale.server.core.universe.world.storage.component
**Type:** Utility

## Definition
```java
// Signature
public class ChunkSavingSystems {
```

## Architecture & Concepts

The ChunkSavingSystems class is a static utility that orchestrates the entire server-side world persistence pipeline. It is not a single, instantiable object but rather a collection of static methods and nested Entity Component System (ECS) classes that work together to manage the asynchronous saving of world chunks to disk.

This system acts as the bridge between the in-memory game state (represented by ECS components in a ChunkStore) and the physical storage layer (managed by an IChunkSaver implementation). Its primary design goals are to prevent data loss and to perform I/O operations with minimal impact on main thread performance.

The architecture is composed of three key parts:
1.  **The `Data` Resource:** A thread-safe, per-world data structure that holds the state of the saving pipeline. This includes a queue of chunks pending a save operation, a set to prevent duplicate queue entries, and a list of active `CompletableFuture` objects representing in-flight I/O operations. As an ECS Resource, it is a singleton within the scope of a specific ChunkStore.
2.  **The `Ticking` System:** An ECS TickingSystem that runs on every server tick. It is responsible for the continuous, background saving process. It periodically queries for "dirty" chunks (those marked as `needsSaving`), adds them to the `Data` queue, and then dequeues a throttled number of chunks to initiate their save operations on a background thread pool.
3.  **The `WorldRemoved` System:** An ECS StoreSystem that hooks into the world shutdown lifecycle. Its sole purpose is to trigger a final, blocking save of all remaining dirty chunks, ensuring no data is lost when the server or a specific world is shut down.

This separation of concerns allows for a robust and performant design. Routine saves are throttled and asynchronous, while shutdown saves are comprehensive and synchronous to guarantee data integrity.

### Lifecycle & Ownership
-   **Creation:** The ChunkSavingSystems class itself is never instantiated. Its nested system classes, `Ticking` and `WorldRemoved`, are instantiated by the ECS framework when a world's ChunkStore is initialized. The `Data` resource is also created and registered with the ChunkStore at this time.
-   **Scope:** The systems and the `Data` resource are scoped to a single World instance and persist for the entire lifetime of that world.
-   **Destruction:** The systems and the `Data` resource are destroyed when the world is unloaded and its associated ChunkStore is torn down. The `onSystemRemovedFromStore` callback in the `WorldRemoved` system is the final execution point, triggering the synchronous save-all operation before the components are deallocated.

## Internal State & Concurrency
-   **State:** The core state is managed within the nested `Data` class, which is highly **mutable**. It contains the queue of chunks to be saved, futures for ongoing I/O, and atomic counters for progress reporting. This state is central to the entire saving process.
-   **Thread Safety:** This class is designed for high-concurrency environments and is **thread-safe**.
    -   The `Data` resource uses concurrent collections (`ConcurrentLinkedDeque`, `ConcurrentHashMap.newKeySet()`) to allow the main game thread to safely queue chunks while background I/O threads may be processing other operations.
    -   Chunk serialization and the I/O write operation (via `IChunkSaver.saveHolder`) are dispatched to the `ForkJoinPool.commonPool()` via `CompletableFuture.supplyAsync`. This offloads the most expensive work from the main server thread.
    -   **CRITICAL:** State modification on ECS components (e.g., setting the `isSaving` flag to false or updating the `ON_DISK` flag) is marshaled back to the world's main thread using `whenCompleteAsync(..., world)`. This prevents hazardous race conditions where a background I/O thread might modify component data while the main game loop is reading it.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| saveChunksInWorld(Store) | CompletableFuture<Void> | O(N * I/O) | Triggers a forced, synchronous save of all dirty chunks in the world. Blocks until all I/O is complete. Intended for world shutdown only. |
| tryQueue(int, ArchetypeChunk, Store) | void | O(1) | Internal helper. Queues a single chunk for saving if it is marked as dirty and not already being saved. Fires a cancellable ChunkSaveEvent. |
| saveChunk(Ref, Data, boolean, Store) | void | O(1) + I/O | Internal helper. Initiates the asynchronous save for a single chunk reference. Dispatches the I/O task to a background thread. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. It is automatically registered and managed by the engine's ECS framework. The standard "usage" is indirect: by modifying a component attached to a chunk entity, the engine will automatically mark the corresponding WorldChunk component as dirty. The `Ticking` system will detect this change and handle the entire persistence process transparently.

```java
// An external system modifies a chunk, which implicitly triggers the saving system.
// This is the ONLY intended interaction pattern.

// 1. Get a reference to a chunk entity.
Ref<ChunkStore> chunkRef = world.getChunkRef(x, z);

// 2. Modify a component on that chunk.
// This action internally sets the chunk's 'needsSaving' flag to true.
store.getComponent(chunkRef, SomeOtherComponent.class).setValue(123);

// 3. On a subsequent tick, ChunkSavingSystems.Ticking will automatically
//    detect the change and queue the chunk for saving. No further action is needed.
```

### Anti-Patterns (Do NOT do this)
-   **Calling `saveChunksInWorld`:** Do not invoke `saveChunksInWorld` during normal gameplay. It is a blocking operation that will iterate, serialize, and save every dirty chunk, causing a severe server stall. It is designed exclusively for the shutdown sequence managed by the `WorldRemoved` system.
-   **Direct Instantiation:** Never create instances of `ChunkSavingSystems.Ticking` or `ChunkSavingSystems.WorldRemoved`. They are ECS systems and must be managed by the ECS `Store`.
-   **State Manipulation:** Do not access or modify the `ChunkSavingSystems.Data` resource from external systems. The integrity of the save queue and its associated futures is critical for data consistency.

## Data Pipeline
The flow of data from an in-memory change to a persisted disk file is a multi-stage, asynchronous pipeline.

> Flow:
> 1.  **Game Logic:** A system modifies a component on a chunk entity.
> 2.  **Dirty Flag:** The chunk's `WorldChunk` component is marked as `needsSaving = true`.
> 3.  **Detection (Main Thread):** The `ChunkSavingSystems.Ticking` system's `tick` method runs and its ECS `Query` identifies the dirty chunk.
> 4.  **Queueing (Main Thread):** The chunk's entity reference (`Ref<ChunkStore>`) is pushed into the concurrent queue within the `Data` resource.
> 5.  **Dequeuing & Dispatch (Main Thread):** The `Ticking` system pulls a reference from the queue (throttling the rate) and initiates an asynchronous save. The chunk's serializable components are copied.
> 6.  **I/O (Worker Thread):** A `CompletableFuture` is created and executed on a background thread pool. This task serializes the component data and calls `IChunkSaver.saveHolder` to write the data to disk.
> 7.  **Callback (World Thread):** Upon I/O completion (success or failure), a `whenCompleteAsync` callback is scheduled to run on the world's specific thread.
> 8.  **State Update (World Thread):** The callback updates the `WorldChunk` component, setting `isSaving = false` and updating flags like `ON_DISK`. This safely completes the cycle without cross-thread data modification.

