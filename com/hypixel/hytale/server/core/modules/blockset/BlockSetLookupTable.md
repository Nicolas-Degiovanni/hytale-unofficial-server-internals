---
description: Architectural reference for BlockSetLookupTable
---

# BlockSetLookupTable

**Package:** com.hypixel.hytale.server.core.modules.blockset
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class BlockSetLookupTable {
```

## Architecture & Concepts
The BlockSetLookupTable is a performance-critical, in-memory data structure designed to provide accelerated lookups for sets of block IDs. It functions as a pre-computed index, transforming a raw map of BlockType definitions into multiple, queryable views based on different block properties like group, hitbox, and category.

Its primary role is to serve as a read-only cache that decouples game logic from the expensive process of iterating over all block assets to find matches. By converting string-based block names from the global BlockTypeAssetMap into more efficient integer IDs, it provides a compact and high-performance mechanism for systems that need to query subsets of blocks.

This class is a foundational component for any system that performs conditional logic based on block characteristics, such as world generation, player interaction, or physics calculations.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its public constructor, which requires a complete Map of String to BlockType definitions. This is a deliberate design choice; the table is built from a specific, provided dataset, not from global state. It is typically instantiated by a higher-level manager during a loading or initialization phase.

- **Scope:** The lifetime of a BlockSetLookupTable is bound to its owner. It is not a global singleton. It is designed to represent a specific collection of blocks (a "blockset") and persists only as long as that collection is relevant.

- **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or disposal methods. It is reclaimed once all references to it are released.

## Internal State & Concurrency
- **State:** **Immutable**. This is the most critical architectural property of the class. During construction, the internal maps are populated from the source data. Upon completion, all internal collections are wrapped in unmodifiable decorators (e.g., Object2ObjectMaps.unmodifiable). Once an instance is created, its state can never be altered.

- **Thread Safety:** **Inherently thread-safe**. Due to its strict immutability, a BlockSetLookupTable instance can be safely shared and read by multiple threads without any external locking or synchronization. This makes it ideal for use in concurrent server environments where different systems may need to query block data simultaneously.

## API Surface
The public API is minimal and exclusively provides read-only access to the pre-computed data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BlockSetLookupTable(Map) | constructor | O(N) | Creates and indexes the table from a source map. Throws IllegalArgumentException if a block name cannot be resolved in the global asset map. |
| addAll(IntSet) | void | O(K) | Populates a provided IntSet with all unique block IDs contained in this table. K is the number of unique blocks. |
| getBlockNameIdMap() | Object2ObjectMap | O(1) | Returns an unmodifiable map of block names to their integer ID sets. |
| getGroupNameIdMap() | Object2ObjectMap | O(1) | Returns an unmodifiable map of group names to their corresponding block ID sets. |
| getHitboxNameIdMap() | Object2ObjectMap | O(1) | Returns an unmodifiable map of hitbox types to their corresponding block ID sets. |
| getCategoryIdMap() | Object2ObjectMap | O(1) | Returns an unmodifiable map of item categories to their corresponding block ID sets. |
| isEmpty() | boolean | O(1) | Returns true if no blocks were indexed by the table. |

## Integration Patterns

### Standard Usage
The intended use is to create an instance once during a loading phase and retain it for fast, repeated queries. It is a data object, not a service.

```java
// During initialization, a manager creates the table from a specific set of blocks.
Map<String, BlockType> myBlockSet = loadBlockSetForZone();
BlockSetLookupTable lookup = new BlockSetLookupTable(myBlockSet);

// Later, in game logic, query the pre-computed data.
IntSet blocksInGrassGroup = lookup.getGroupNameIdMap().get("grass");
if (blocksInGrassGroup.contains(player.getBlockBelow())) {
    // Apply logic
}
```

### Anti-Patterns (Do NOT do this)
- **Per-Query Instantiation:** Do not create a new BlockSetLookupTable for every query. The constructor performs a full iteration and indexing of the source map, which is an expensive operation designed to be run once.

- **Attempted Modification:** Do not attempt to modify the collections returned by the getter methods. They are immutable wrappers and will throw an UnsupportedOperationException, causing a server crash.

- **Reliance on Global State:** While the constructor uses the global BlockType.getAssetMap() for ID resolution, the instance itself is self-contained. Do not assume its contents reflect the current global state if assets are hot-loaded or changed after its creation.

## Data Pipeline
The class acts as a transformation and indexing step in the server's data flow. It converts raw configuration data into a query-optimized format.

> Flow:
> Raw BlockType Asset Map -> **BlockSetLookupTable Constructor** (Transforms & Indexes) -> Immutable In-Memory Maps -> Game Systems (Performs fast lookups)

