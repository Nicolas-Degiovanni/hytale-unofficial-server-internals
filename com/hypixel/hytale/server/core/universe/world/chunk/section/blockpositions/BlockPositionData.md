---
description: Architectural reference for BlockPositionData
---

# BlockPositionData

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section.blockpositions
**Type:** Transient

## Definition
```java
// Signature
public class BlockPositionData implements IBlockPositionData {
```

## Architecture & Concepts
BlockPositionData is a fundamental data-transfer object (DTO) that represents a single block at a specific position in the game world. Its primary architectural role is to abstract away the complexities of the chunk-based storage system.

Game systems like physics, AI, or world-event handlers need to operate in absolute world coordinates. However, block data is stored in a highly optimized, hierarchical structure of Chunks and BlockSections, where a block is identified by a local index. BlockPositionData serves as the translation layer, providing a clean, world-aware interface to this low-level data.

It encapsulates three key pieces of information:
1.  A local block index within its parent section.
2.  A reference to its parent ChunkSection.
3.  The block's type identifier.

By combining these, it can compute absolute world coordinates on demand, decoupling higher-level game logic from the underlying storage implementation. It is designed as an immutable snapshot; once created, its positional data does not change.

## Lifecycle & Ownership
-   **Creation:** Instances are created on-the-fly by systems that query or iterate over the contents of a BlockSection. It is not managed by a dependency injection container or a central registry. A typical creator would be a world query engine or a block iterator that yields BlockPositionData objects as its result.
-   **Scope:** Extremely short-lived. A BlockPositionData object is a transient artifact of a calculation or query. Its lifetime is typically confined to the local scope of the method that created it.
-   **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection as soon as it goes out of scope and no references remain. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The object is effectively **immutable**. All internal fields (blockIndex, section, blockType) are set at construction and are not modified thereafter. While the fields are not explicitly marked as final, the public API provides no methods for mutation.
-   **Thread Safety:** **Conditionally Thread-Safe**. The object itself performs no internal locking or mutation, making its coordinate calculation methods (`getX`, `getY`, `getZ`) safe to call from any thread. However, it holds a reference to a ChunkSectionReference. If the underlying BlockSection is modified by another thread via this reference, data returned from `getChunkSection` could become inconsistent. Callers must ensure that the underlying world data is not being modified concurrently without proper synchronization.

## API Surface
The public API is focused on translating internal, chunk-local data into world-space coordinates.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChunkSection() | BlockSection | O(1) | Retrieves the parent BlockSection this block belongs to. |
| getBlockType() | int | O(1) | Returns the integer ID representing the type of this block. |
| getX() | int | O(1) | Calculates and returns the absolute world X-coordinate. |
| getY() | int | O(1) | Calculates and returns the absolute world Y-coordinate. |
| getZ() | int | O(1) | Calculates and returns the absolute world Z-coordinate. |
| getXCentre() | double | O(1) | Calculates the geometric center of the block on the world X-axis. |
| getYCentre() | double | O(1) | Calculates the geometric center of the block on the world Y-axis. |
| getZCentre() | double | O(1) | Calculates the geometric center of the block on the world Z-axis. |

## Integration Patterns

### Standard Usage
BlockPositionData is typically received from a world query or iterator. The consumer then uses its methods to get world-aware information without needing to understand chunk math.

```java
// A hypothetical world system provides a reference to a chunk section.
ChunkSectionReference sectionRef = world.getSectionAt(chunkX, chunkY, chunkZ);

// A system iterates through the section's blocks to find a specific type.
for (int i = 0; i < 32768; i++) {
    int blockType = sectionRef.getSection().getBlockType(i);
    if (blockType == BlockTypes.DIAMOND_ORE) {
        // Create a transient data object for this block.
        BlockPositionData blockData = new BlockPositionData(i, sectionRef, blockType);

        // Use the object to work with world coordinates.
        WorldEventBus.post(new OreFoundEvent(blockData.getX(), blockData.getY(), blockData.getZ()));
        break;
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Caching:** Do not store BlockPositionData instances in long-lived collections or caches. They hold a reference to a ChunkSectionReference, which can prevent a chunk from being unloaded from memory, leading to severe memory leaks. They are designed to be discarded after use.
-   **Manual Instantiation with Invalid Data:** Do not create a BlockPositionData with a blockIndex that is out of sync with the provided ChunkSectionReference. The class trusts its constructor arguments, and providing inconsistent data will lead to incorrect world coordinate calculations.
-   **Assuming State Stability:** Do not read data from a BlockPositionData instance, perform a long-running operation, and then assume the underlying BlockSection has not changed. The world may have been modified in the interim. Re-query the world for fresh data if consistency is required.

## Data Pipeline
BlockPositionData is not a processing stage in a pipeline; rather, it is the data payload that flows between stages. It represents the transformation of raw, indexed data into a more usable, context-rich format.

> Flow:
> World Query Engine -> Iterates `BlockSection` raw data -> **BlockPositionData Instantiation** -> Consuming System (Physics, AI, etc.)

