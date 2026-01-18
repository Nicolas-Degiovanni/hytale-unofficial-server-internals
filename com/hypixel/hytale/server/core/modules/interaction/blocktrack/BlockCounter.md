---
description: Architectural reference for BlockCounter
---

# BlockCounter

**Package:** com.hypixel.hytale.server.core.modules.interaction.blocktrack
**Type:** Transient Data Component

## Definition
```java
// Signature
public class BlockCounter implements Resource<ChunkStore> {
```

## Architecture & Concepts
The BlockCounter is a stateful data component responsible for tracking block placement and removal statistics within a single game chunk. It functions as a specialized resource, attaching directly to a ChunkStore instance. This tight coupling ensures that block count data is managed as an integral part of a chunk's persistent state.

Its primary role within the engine is to serve the InteractionModule by providing a mechanism to record and query how many blocks of a specific type have been placed in a chunk. This data can be used for various gameplay systems, such as quests, achievements, or server-side analytics.

The presence of a static CODEC field is a critical architectural feature. It signifies that BlockCounter instances are designed to be serialized and deserialized along with the chunk data they belong to, making the statistics fully persistent across server restarts.

## Lifecycle & Ownership
- **Creation:** A BlockCounter is not instantiated directly. It is created and managed by the server's resource system. An instance is typically created in one of two ways:
    1.  Deserialized from disk when a ChunkStore is loaded.
    2.  Instantiated on-demand by the resource system the first time a block is tracked within a new or previously unmodified chunk.
- **Scope:** The lifecycle of a BlockCounter is strictly bound to the lifecycle of its parent ChunkStore. It exists in memory only as long as the corresponding chunk is loaded.
- **Destruction:** The instance is eligible for garbage collection when its parent ChunkStore is unloaded from memory. Its serialized data is permanently deleted if the chunk file is removed from the world save.

## Internal State & Concurrency
- **State:** The BlockCounter is a mutable, stateful component. Its core state is maintained in the `blockPlacementCounts` map, an `Object2IntOpenHashMap` which stores a mapping of block names to their placement counts. The `clone` method performs a deep copy of this map, ensuring state isolation.

- **Thread Safety:** **WARNING:** This class is **not thread-safe**. The underlying `Object2IntOpenHashMap` is not synchronized. All method calls that read or modify the internal map must be externally synchronized. In practice, this means all interactions with a BlockCounter instance must occur on the main server thread that owns the corresponding world and its chunks. Unsynchronized access from other threads will lead to data corruption or `ConcurrentModificationException`.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceType() | static ResourceType | O(1) | Retrieves the unique type identifier used to register and look up this resource. |
| trackBlock(String blockName) | void | O(1) | Atomically increments the placement count for the given block name. |
| untrackBlock(String blockName) | void | O(1) | Atomically decrements the placement count for the given block name. |
| getBlockPlacementCount(String blockName) | int | O(1) | Returns the current placement count for the given block name. Returns 0 if never tracked. |
| clone() | Resource | O(N) | Creates a deep copy of this BlockCounter, where N is the number of unique block types tracked. |

## Integration Patterns

### Standard Usage
A BlockCounter should never be obtained directly. It must be retrieved as a resource from a valid ChunkStore instance. The InteractionModule is the canonical consumer of this class.

```java
// Correctly retrieve the BlockCounter for a specific chunk's storage
ChunkStore targetChunkStore = world.getChunkAt(chunkPos).getStorage();

// The resource system will create a new BlockCounter if one does not exist
BlockCounter counter = targetChunkStore.getOrCreateResource(BlockCounter.getResourceType());

// Modify the counter
counter.trackBlock("hytale:dirt");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BlockCounter()`. An instance created this way will be detached from any ChunkStore, will not be persisted, and will be ignored by the game's interaction systems.

- **Cross-Thread Modification:** Do not access a BlockCounter from an asynchronous task or a different thread without explicit synchronization on the parent ChunkStore or World object. This is a direct path to server instability.

## Data Pipeline
The BlockCounter acts as a data sink in the block interaction pipeline. It does not forward data but rather accumulates it for persistence and later querying.

> **Interaction Flow:**
> Player Places Block -> Network Packet -> Server `InteractionModule` -> `ChunkStore` lookup -> **`BlockCounter.trackBlock()`**

> **Persistence Flow:**
> World Save Trigger -> `ChunkStore.save()` -> **`BlockCounter`** is serialized via `CODEC` -> Written to Chunk File on Disk

