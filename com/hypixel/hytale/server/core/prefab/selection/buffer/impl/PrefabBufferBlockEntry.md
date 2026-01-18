---
description: Architectural reference for PrefabBufferBlockEntry
---

# PrefabBufferBlockEntry

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer.impl
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class PrefabBufferBlockEntry {
```

## Architecture & Concepts
The PrefabBufferBlockEntry is a fundamental, short-lived data container that represents a single block's complete state before it is materialized in the game world. It is not a service or manager, but rather a Plain Old Java Object (POJO) that acts as a data transfer object within the server's world generation pipeline.

This class is a core component of the Prefab system. Prefabs are pre-designed structures—such as dungeons, trees, or ruins—that are placed into the world procedurally. A PrefabBufferBlockEntry serves as the in-memory, intermediate representation of one block within such a structure. It holds all the necessary information—block type, orientation, fluid data, and extended tile entity state—required to translate a conceptual block from a prefab definition into a concrete block in the world's ChunkStore.

Its primary role is to decouple the prefab definition from the final world state, allowing for intermediate processing and modification before the block is committed to storage.

### Lifecycle & Ownership
- **Creation:** Instances are created in large numbers by prefab loading and processing systems during world generation. A `PrefabProcessor` or a similar generator will parse a prefab definition and instantiate a collection of these objects to represent the structure.
- **Scope:** Extremely short-lived and tightly scoped. An instance exists only for the duration of a single prefab placement operation within a specific world chunk.
- **Destruction:** Once the data from a PrefabBufferBlockEntry has been written to the target ChunkStore, the object is dereferenced. It holds no persistent references and is designed to be aggressively reclaimed by the garbage collector. Holding onto these objects beyond the generation pass is a memory leak.

## Internal State & Concurrency
- **State:** Highly mutable. With the exception of the final field *y*, all public fields can be modified after instantiation. This mutability is intentional, allowing world generation logic to apply transformations, such as rotating a structure or substituting block types based on biome, before final placement.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and manipulated exclusively by the single worker thread responsible for generating a specific region of the world. Any concurrent access or modification from multiple threads without external synchronization will result in race conditions, data corruption, and non-deterministic world generation.

## API Surface
The public contract of this class is direct field access. It contains no logic or methods beyond its constructors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (constructor) | PrefabBufferBlockEntry | O(1) | Creates a new block entry. Overloads provide defaults for simpler block types. |
| y | final int | O(1) | The immutable vertical coordinate (Y-axis) of the block within its column. |
| blockId | int | O(1) | The numerical identifier for the block type. Can be modified post-construction. |
| blockTypeKey | String | O(1) | The human-readable string key for the block type, e.g., "hytale:stone". |
| chance | float | O(1) | The probability (0.0 to 1.0) that this block will actually be placed. |
| state | Holder<ChunkStore> | O(1) | A nullable reference to extended block state, used for tile entities like chests or signs. |
| fluidId | int | O(1) | The numerical identifier for any fluid contained within this block space. |

## Integration Patterns

### Standard Usage
The standard pattern involves a generator creating a collection of these objects from a prefab source, potentially modifying them, and then iterating through them to apply the data to the world.

```java
// Conceptual example of a prefab placement system
Prefab sourcePrefab = prefabRegistry.get("hytale:ancient_well");
List<PrefabBufferBlockEntry> blockBuffer = sourcePrefab.getBlockBufferForPlacement();

for (PrefabBufferBlockEntry entry : blockBuffer) {
    // Example: Modify block type based on biome context
    if (world.getBiomeAt(x, z) == Biomes.SNOWY_FOREST) {
        if (entry.blockTypeKey.equals("hytale:grass")) {
            entry.blockTypeKey = "hytale:snow";
        }
    }
    
    // Commit the final block data to the world chunk
    chunk.setBlockFromBufferEntry(x, entry.y, z, entry);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not cache or store instances of PrefabBufferBlockEntry in any long-lived collection. They are transient data carriers for a single operation and retaining them will cause a significant memory leak.
- **Cross-Thread Sharing:** Do not pass an instance of this class to another thread for modification without explicit and robust synchronization. The object is designed for single-threaded ownership within a generation task.
- **Misuse for Reading:** This class is exclusively for the world *generation* and *writing* pipeline. When reading block data from the world, you will receive different data structures. Attempting to use this class to represent an existing block is incorrect.

## Data Pipeline
A PrefabBufferBlockEntry is not a processing stage itself, but rather the data payload that flows between stages in the world generation pipeline.

> Flow:
> Prefab Definition (File) -> Prefab Parser -> **Collection of PrefabBufferBlockEntry** -> World Generator / Biome Processor -> ChunkStore (Final World Data)

