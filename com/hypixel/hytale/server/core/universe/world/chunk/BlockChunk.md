---
description: Architectural reference for BlockChunk
---

# BlockChunk

**Package:** com.hypixel.hytale.server.core.universe.world.chunk
**Type:** Component

## Definition
```java
// Signature
public class BlockChunk implements Component<ChunkStore> {
```

## Architecture & Concepts

The BlockChunk is the fundamental server-side data structure representing a 32x320x32 volume of blocks within the game world. It serves as the canonical source of truth for all block data, lighting, environmental properties, and vertical column information for a specific chunk coordinate.

Architecturally, BlockChunk is designed as a component within Hytale's Entity Component System (ECS). It is attached to a `ChunkStore` entity, which acts as a container for all data related to a single vertical column of the world. A BlockChunk is not a monolithic data block; it is a composite object that aggregates several specialized data structures:

*   **BlockSections:** The 320-block vertical space is subdivided into ten `BlockSection` objects, each managing a 32x32x32 cube of blocks. This spatial partitioning is critical for performance, allowing for efficient rendering, networking, and simulation updates on a more granular level than the entire chunk.
*   **Palettes:** It utilizes `ShortBytePalette` for heightmap data and `IntBytePalette` for tintmap data. These are optimized, compressed data structures for storing 2D column-based information.
*   **EnvironmentChunk:** This nested object manages environmental data (e.g., biome-specific effects, atmospheric conditions) on a per-block basis within the chunk's volume.
*   **Packet Caching:** To optimize network performance and reduce CPU load from repeated serialization, BlockChunk maintains `SoftReference` caches for its corresponding network packets (e.g., `SetChunkHeightmap`). These packets are generated asynchronously and cached, allowing the server to reuse them for multiple players viewing the same chunk. The use of `SoftReference` ensures these caches do not cause memory leaks and can be reclaimed by the garbage collector under memory pressure.

The class also provides a versioned serialization mechanism via its static `CODEC` field, ensuring backward compatibility for chunks saved with older server versions.

## Lifecycle & Ownership

-   **Creation:** A BlockChunk is never instantiated directly by game logic systems. Its lifecycle is strictly managed by the world's chunk management system. Instances are created in two primary scenarios:
    1.  **World Generation:** A new BlockChunk is created programmatically when a new, unexplored area of the world is generated.
    2.  **Deserialization:** An existing BlockChunk is instantiated and hydrated with data from disk via its `CODEC` when a previously saved chunk is loaded into memory.
-   **Scope:** The object's lifetime is coupled directly to the in-memory lifetime of the world chunk it represents. It persists as long as the server considers the chunk to be "loaded" and active.
-   **Destruction:** A BlockChunk is marked for garbage collection when its parent chunk is unloaded from memory by the server, typically because no players are nearby. There is no explicit destruction method; ownership is released, and the Java Garbage Collector reclaims its memory.

## Internal State & Concurrency

-   **State:** BlockChunk is a highly **mutable** and stateful component. It contains the core block data, heightmaps, lighting, and various boolean flags like `needsSaving` and `needsPhysics`. These flags act as dirty bits, signaling to other engine systems that the chunk requires processing by the persistence or physics engines.
-   **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access from the main server game loop. All mutations, such as calls to `setBlock` or `setHeight`, must be performed on the world's primary tick thread. Concurrent modification from other threads (e.g., network threads, asynchronous tasks) will lead to data corruption, race conditions, and server instability. The only thread-safe operations are the asynchronous packet generation methods (`getCachedHeightmapPacket`), which are explicitly designed to perform serialization work on a background thread pool.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setBlock(x, y, z, blockId, ...) | boolean | O(1) | Modifies the block at the given world-relative coordinates. Marks the chunk for saving and invalidates the relevant BlockSection's network cache. Throws IllegalArgumentException if Y is out of bounds. |
| getBlock(x, y, z) | int | O(1) | Retrieves the block ID at the given coordinates. Returns 0 (air) for out-of-bounds queries. |
| updateHeightmap() | void | O(N*M) | Recalculates the entire heightmap for the chunk by iterating through each (x, z) column. This is a computationally intensive operation. |
| forEachTicking(...) | int | O(T) | Iterates over all blocks within the chunk that are registered for random block ticks. T is the number of ticking blocks, not the total number of blocks. |
| consumeNeedsSaving() | boolean | O(1) | Atomically retrieves and resets the `needsSaving` flag. This is used by the persistence system to identify which chunks to write to disk. |
| getCachedHeightmapPacket() | CompletableFuture | O(1) | Returns a future for the cached network packet representing the chunk's heightmap. May trigger asynchronous serialization if the cache is empty. |

## Integration Patterns

### Standard Usage

Interaction with BlockChunk should occur through high-level world APIs. If direct access is necessary, it must be retrieved as a component from a `ChunkStore` entity, which is typically managed by a `World` or `Universe` service.

```java
// A system running on the main server thread
Holder<ChunkStore> chunkHolder = world.getChunkAt(chunkX, chunkZ);
if (chunkHolder != null) {
    BlockChunk blockChunk = chunkHolder.getComponent(BlockChunk.getComponentType());
    // Modify a block, which implicitly marks the chunk for saving
    blockChunk.setBlock(10, 80, 25, newBlockId, 0, 0);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new BlockChunk()`. The chunk lifecycle is managed exclusively by the server's world loading and generation systems. Manual creation will result in a disconnected, untracked chunk that will not be saved or simulated correctly.
-   **Cross-Thread Modification:** Do not access and modify a BlockChunk instance from any thread other than the main server tick thread. This is the most common source of critical server errors. Asynchronous operations should delegate state changes back to the main thread for execution.
-   **State Mismanagement:** Forgetting to call `markNeedsSaving()` after a custom data modification (one that doesn't use a built-in method like `setBlock`) will result in that change being lost upon server restart.

## Data Pipeline

The BlockChunk is a central hub for several critical data flows within the server.

**Persistence Pipeline (Saving a Chunk):**
> Game Logic -> `BlockChunk.setBlock()` -> `markNeedsSaving()` is set to true -> World Persistence System (on tick) -> Detects `consumeNeedsSaving()` is true -> `BlockChunk.CODEC.serialize()` -> Raw Bytes -> Disk

**Networking Pipeline (Sending Chunk to Client):**
> Player Enters View Distance -> `LoadBlockChunkPacketSystem` is triggered -> `BlockChunk.getCached...Packet()` -> Asynchronous Packet Serialization -> `CachedPacket` -> Network Queue -> Player Client

**Block Simulation Pipeline (Ticking):**
> Main Server Tick -> World Ticking System -> `BlockChunk.forEachTicking()` -> Iterates `BlockSection` ticking sets -> `BlockTickStrategy` is executed for each block -> Game State is mutated

