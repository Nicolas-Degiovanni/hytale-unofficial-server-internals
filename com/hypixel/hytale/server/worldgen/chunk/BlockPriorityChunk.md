---
description: Architectural reference for BlockPriorityChunk
---

# BlockPriorityChunk

**Package:** com.hypixel.hytale.server.worldgen.chunk
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class BlockPriorityChunk {
```

## Architecture & Concepts

The BlockPriorityChunk is a specialized, in-memory data structure that serves as a conflict resolution map during the procedural generation of a single world chunk. It does not store final block types like stone or dirt. Instead, it holds a *priority value* for each block position within the chunk's volume.

World generation is a multi-pass process where different algorithms (e.g., terrain shaping, cave carving, structure placement) compete to place blocks. This class arbitrates that competition. For example, a high-priority PREFAB placement from a structure generator will be allowed to overwrite a low-priority FILLING operation from the base terrain generator.

The system is implemented using a flat byte array and bitwise operations for efficiency. For each block coordinate, an 8-bit value is stored:
*   **Priority (5 bits):** The lower 5 bits represent the priority level, from NONE (0) to PREFAB (9) and higher. This is retrieved using the MASK constant.
*   **Flags (3 bits):** The upper 3 bits are reserved for flags, such as FLAG_SUBMERGE, which modifies generator behavior (e.g., allowing a block to be placed within a water volume).

This component is critical for layering complex world features correctly and preventing generation artifacts, such as dungeons being overwritten by generic cave systems.

### Lifecycle & Ownership

*   **Creation:** A new BlockPriorityChunk is instantiated by a world generation coordinator at the beginning of a generation task for a single chunk. It is intended as a short-lived, single-use workspace.
*   **Scope:** The object's lifetime is strictly bound to the generation process of one chunk. It is created, populated by various generator passes, and then discarded once the final block data has been written to a persistent Chunk object.
*   **Destruction:** As a simple Java object with no native resources, it is automatically reclaimed by the garbage collector once the generation task completes and all references to it are dropped. There is no explicit destruction method.

## Internal State & Concurrency

*   **State:** This class is fundamentally a mutable state container. Its core is the `blocks` byte array, which is continuously modified by various world generation algorithms. The `reset` method must be called after instantiation to ensure a clean initial state of priority NONE (0) for all blocks.
*   **Thread Safety:** **This class is not thread-safe.** All read and write operations directly access the underlying array without any locks or synchronization. It is designed to be confined to a single worker thread responsible for generating one chunk. Sharing an instance across multiple threads will result in race conditions and severe world generation corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| reset() | BlockPriorityChunk | O(N) | Fills the entire internal array with zero bytes. Essential for initialization. |
| get(x, y, z) | byte | O(1) | Retrieves the 5-bit priority value for a block, ignoring any flags. |
| getRaw(x, y, z) | byte | O(1) | Retrieves the full 8-bit value, including both priority and flags. |
| set(x, y, z, type) | void | O(1) | Sets the full 8-bit priority and flag value for a block. |

## Integration Patterns

### Standard Usage

A world generator task creates an instance, resets it, and then passes it sequentially through different generation stages. Each stage reads the existing priority to make decisions and writes a new, higher priority if its conditions are met.

```java
// Pseudocode within a chunk generation task
BlockPriorityChunk priorityMap = new BlockPriorityChunk().reset();

// Pass 1: Base terrain shaping
terrainGenerator.apply(priorityMap, chunkCoords, seed); // Writes FILLING, LAYER priorities

// Pass 2: Cave carving
caveGenerator.apply(priorityMap, chunkCoords, seed); // Writes CAVE, overwriting lower priorities

// Pass 3: Structure placement
structureGenerator.apply(priorityMap, chunkCoords, seed); // Writes PREFAB, the highest priority

// Finalization: A builder reads the map to construct the real chunk
Chunk finalChunk = chunkBuilder.buildFromPriorityMap(priorityMap);
```

### Anti-Patterns (Do NOT do this)

*   **Long-Term Retention:** Do not hold references to a BlockPriorityChunk after its corresponding chunk has been generated. These objects are large (320 KB) and retaining them will cause a memory leak.
*   **Concurrent Modification:** Never pass a BlockPriorityChunk instance to multiple threads that could write to it simultaneously. All generation passes for a single chunk must be executed serially on the same thread.
*   **Forgetting to Reset:** Failing to call `reset` on a new instance will result in undefined behavior, as the initial state of the byte array is not guaranteed. Re-using an instance for a new chunk without calling `reset` will cause features from the previous chunk to "bleed" into the new one.

## Data Pipeline

The BlockPriorityChunk acts as a temporary, intermediate representation of a chunk during its construction.

> Flow:
> World Generation Task Begins -> `new BlockPriorityChunk()` -> **Pass 1: Terrain Generator** -> **Pass 2: Cave Generator** -> **Pass 3: Structure Generator** -> Finalizer reads **BlockPriorityChunk** -> Final block data written to main Chunk object -> **BlockPriorityChunk** is discarded.

