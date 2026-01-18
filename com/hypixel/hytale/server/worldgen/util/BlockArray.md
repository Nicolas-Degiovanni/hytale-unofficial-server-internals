---
description: Architectural reference for the BlockArray interface, a core data structure in the world generation pipeline.
---

# BlockArray

**Package:** com.hypixel.hytale.server.worldgen.util
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface BlockArray {
```

## Architecture & Concepts
The BlockArray interface defines a fundamental contract for an immutable collection of block type identifiers. It serves as a standardized, read-only view of a set of blocks, abstracting away the underlying storage mechanism. This abstraction is critical within the world generation system, where various algorithms need to query block sets without being coupled to a specific implementation like a primitive array, a List, or a HashSet.

Its primary role is to provide data to decision-making components. For example, a biome definition might use a BlockArray to specify which block types are considered valid ground for placing foliage, or a structure generator might use it to define the set of materials it is built from. By relying on this interface, algorithms remain decoupled and can operate on any data source that conforms to the BlockArray contract.

## Lifecycle & Ownership
As an interface, BlockArray itself has no lifecycle. The lifecycle and ownership semantics apply to the **concrete classes** that implement it.

- **Creation:** Instances are typically created and populated during the loading phase of world generation assets, such as biome or structure definition files. They are data-transfer objects, often deserialized from configuration.
- **Scope:** The lifetime of a BlockArray instance is tied to its owning object. If a BlockArray defines the stone types for a biome, it will exist as long as that biome definition is loaded in memory. They are generally long-lived but confined to the server's world-generation context.
- **Destruction:** Garbage collected when the owning configuration object (e.g., BiomeDefinition) is unloaded. There is no explicit destruction method.

## Internal State & Concurrency
The BlockArray contract strongly implies immutability. Implementations are expected to be treated as read-only after their initial construction.

- **State:** The state consists of a collection of integer block IDs. While the interface does not enforce immutability, all consumers **must** treat instances as immutable.
- **Thread Safety:** Implementations of this interface **must be thread-safe**. World generation is a highly concurrent process, and a single BlockArray instance (e.g., "all valid ores") may be read by multiple generator threads simultaneously. Safe publication and read-only internal state are required.

**WARNING:** The `getBlocks` method returns a primitive `int[]`. This poses a significant risk as it can leak internal state. Implementers should always return a defensive copy of the internal array to prevent external modification. Consumers must never attempt to modify the returned array.

## API Surface
The public contract is minimal, focusing exclusively on querying the block collection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlocks() | int[] | O(N) | Returns a copy of all block IDs in the collection. |
| size() | int | O(1) | Returns the total number of block IDs in the collection. |
| contains(int blockId) | boolean | O(1) to O(N) | Checks for the existence of a specific block ID. Performance is implementation-dependent. |

## Integration Patterns

### Standard Usage
The interface is used to pass sets of block definitions to algorithms that need to make placement or replacement decisions based on block type.

```java
// A generator receives a BlockArray defining valid ground types
public void placeFeature(World world, BlockArray validGroundTypes) {
    for (int x = 0; x < 16; x++) {
        // ... find y coordinate
        int existingBlockId = world.getBlock(x, y, z);
        if (validGroundTypes.contains(existingBlockId)) {
            // It's safe to place the feature here
            world.setBlock(x, y + 1, z, FEATURE_BLOCK);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Modifying the Returned Array:** The array returned by `getBlocks` must never be modified. Doing so could corrupt the state of a globally used definition, leading to unpredictable world generation bugs that are extremely difficult to trace.
    ```java
    // DO NOT DO THIS
    BlockArray ores = Biome.getOreTypes();
    int[] oreIds = ores.getBlocks();
    oreIds[0] = Block.WATER; // Corrupts the biome definition for all threads
    ```
- **Performance Assumptions:** Do not assume the performance of `contains`. While some implementations may use a HashSet for O(1) lookups, a simple implementation might use a linear scan, resulting in O(N) performance. Code in performance-critical loops should be mindful of this.

## Data Pipeline
BlockArray is not a processing stage in a pipeline; rather, it is the data that flows into a decision-making stage.

> Flow:
> Biome Configuration File -> Deserializer -> **BlockArray Instance** -> Structure Placement Algorithm -> World State Mutation

