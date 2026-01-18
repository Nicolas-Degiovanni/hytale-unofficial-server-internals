---
description: Architectural reference for IChunkAccessorSync
---

# IChunkAccessorSync

**Package:** com.hypixel.hytale.server.core.universe.world.accessor
**Type:** Contract Interface

## Definition
```java
// Signature
@Deprecated
public interface IChunkAccessorSync<WorldChunk extends BlockAccessor> {
```

## Architecture & Concepts

The IChunkAccessorSync interface defines a synchronous, blocking contract for accessing and manipulating world data at the block and chunk level. It serves as a high-level Facade over the server's underlying chunk management and storage systems. Its primary architectural role is to provide a unified API for game logic to query or modify the world without needing direct knowledge of the chunk caching, loading, or serialization mechanisms.

The **Sync** suffix is the most critical aspect of this interface's design. It signifies that any method, particularly those retrieving a chunk like getChunk, may block the calling thread until the required data is loaded from disk or generated. This design simplifies certain game logic by guaranteeing that data is available upon method return, but it carries a severe performance cost if used improperly, potentially stalling the main server thread.

**WARNING:** This interface is marked as **Deprecated**. It represents an older architectural pattern that favors simplicity over performance. Modern systems should use asynchronous or non-blocking alternatives to prevent stalling the server's main tick loop. Its use in new code is strictly forbidden.

## Lifecycle & Ownership

-   **Creation:** As an interface, IChunkAccessorSync is not instantiated directly. Concrete classes that implement this contract, such as a World or WorldView object, are created and managed by the server's Universe system when a world is loaded. Game systems receive a reference to an implementation, never creating one themselves.
-   **Scope:** The lifetime of an object implementing this interface is intrinsically tied to the lifetime of the world it represents. It remains valid as long as the world is loaded and active on the server.
-   **Destruction:** The implementing object is marked for garbage collection when the corresponding world is unloaded or the server shuts down. References to it become invalid at this point.

## Internal State & Concurrency

-   **State:** The interface itself is stateless. However, any class that implements it is, by definition, a view over a highly stateful and mutable system: the world chunk cache. Operations performed through this interface directly modify the state of the world.
-   **Thread Safety:** **Not thread-safe.** This interface is designed to be called exclusively from the main server thread, which is responsible for the world tick. Calling its methods from any other thread is extremely hazardous. Doing so will almost certainly lead to race conditions, data corruption, or deadlocks with the chunk loading and saving systems. All world modifications must be synchronized with the main game loop.

## API Surface

The API provides two categories of methods: chunk retrieval and block manipulation. The block manipulation methods are default implementations that first retrieve the relevant chunk and then delegate the operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChunk(long chunkIndex) | WorldChunk | O(Disk I/O) | Retrieves a chunk by its packed index. **This is a blocking call** and may stall the thread if the chunk is not in memory. |
| getChunkIfLoaded(long chunkIndex) | WorldChunk | O(1) | Retrieves a chunk only if it is already loaded and ticking. Returns null otherwise. Non-blocking. |
| getBlock(int x, int y, int z) | int | O(Disk I/O) | Convenience method to get a block ID at a world position. Internally calls getChunk. |
| setBlock(int x, int y, int z, ...) | void | O(Disk I/O) | Convenience method to set a block at a world position. Internally calls getChunk. |
| breakBlock(int x, int y, int z, ...) | boolean | O(Disk I/O) | Convenience method to break a block at a world position. Internally calls getChunk. |

## Integration Patterns

### Standard Usage

The intended use is for synchronous game logic running on the main server thread, where an immediate, blocking result is required.

```java
// Example: A script running within the main server tick
// 'world' is an object that implements IChunkAccessorSync

void placeBedrockAtSpawn(IChunkAccessorSync world) {
    // This call may block if the spawn chunk isn't loaded
    world.setBlock(0, 64, 0, "hytale:bedrock");
}
```

### Anti-Patterns (Do NOT do this)

-   **Usage in New Code:** The most significant anti-pattern is using this deprecated interface at all. Prefer asynchronous APIs to avoid blocking the main thread.
-   **Multi-threaded Access:** Never call methods on this interface from an asynchronous task, worker thread, or network thread. This will bypass the world's thread-safety mechanisms and lead to catastrophic state corruption.
-   **Hot Loop Operations:** Avoid calling methods like getBlock or setBlock inside tight loops that span multiple chunks. Each call may incur the overhead of a hash map lookup for the chunk. If performing many operations in one chunk, retrieve it once with getChunk and operate on the chunk object directly.

## Data Pipeline

The data flow for a typical write operation demonstrates the synchronous, blocking nature of the interface. The calling thread is halted until the entire sequence is complete, including any potential disk access.

> Flow:
> Game Logic Call -> `IChunkAccessorSync.setBlock(x, y, z, ...)` -> `ChunkUtil.indexChunkFromBlock(x, z)` -> `getChunk(chunkIndex)` -> **[THREAD BLOCKS]** ChunkCache checks for chunk -> If miss, ChunkLoader reads from disk -> **[THREAD RESUMES]** -> `WorldChunk.setBlock(...)` -> Chunk data is mutated in memory.

