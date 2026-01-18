---
description: Architectural reference for BlockMask
---

# BlockMask

**Package:** com.hypixel.hytale.server.core.prefab.selection.mask
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class BlockMask {
```

## Architecture & Concepts
The BlockMask is a fundamental data structure within the server's world manipulation and prefab systems. It represents a declarative set of rules used to determine whether a specific block at a given world coordinate should be included or excluded from an operation.

At its core, a BlockMask is a collection of one or more BlockFilter objects. The evaluation logic is an implicit **OR** operation: a block is considered a match if it satisfies *any* of the contained filters. The entire result can then be logically negated by an *inversion* flag, allowing for complex "include all except" scenarios.

This component is critical for features such as:
-   **Prefab Placement:** Defining which existing blocks should be replaced or ignored when a prefab is placed.
-   **World Generation:** Masking areas to prevent or allow specific feature generation.
-   **In-Game Tooling:** Powering user-facing tools that operate on specific block types (e.g., "replace all stone with dirt").

The class provides a robust serialization and deserialization mechanism via its static CODEC field, enabling masks to be defined in external configuration files and parsed at runtime. The string format supports comma-separated filters, providing a concise and human-readable representation.

## Lifecycle & Ownership
-   **Creation:** BlockMask instances are almost exclusively created through static factory methods. The primary entry point is **BlockMask.parse(String)**, which deserializes a mask from its string representation. The **BlockMask.combine(BlockMask...)** method is used to programmatically construct more complex masks from existing ones. Direct instantiation via the constructor is discouraged.
-   **Scope:** Instances of BlockMask are typically short-lived and transactional. They are created on-demand to execute a specific, bounded operation (e.g., the placement of a single prefab). They are passed as parameters to methods that iterate over world data and are eligible for garbage collection once the operation completes. They are not designed to be held as long-term state in services.
-   **Destruction:** Managed entirely by the Java garbage collector. No native resources are held, and no explicit cleanup is required.

## Internal State & Concurrency
-   **State:** The primary state consists of a final array of BlockFilter objects and a mutable boolean **inverted** flag. The **filters** array is immutable after construction, ensuring the core filtering logic cannot change. However, the **inverted** flag can be modified post-construction via the setInverted method.

-   **Thread Safety:** **This class is NOT thread-safe.** The presence of the public, unsynchronized **setInverted** method makes instances mutable. If one thread is calling **isExcluded** while another thread calls **setInverted** on the same instance, the behavior is undefined and may lead to inconsistent results.

    **WARNING:** A single BlockMask instance must not be shared and modified across multiple threads without external synchronization. For concurrent operations, either create a new BlockMask per thread or ensure all mutations occur before the instance is passed to worker threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(String masks) | static BlockMask | O(N) | **Primary Factory.** Deserializes a mask from its string representation. |
| combine(BlockMask... masks) | static BlockMask | O(N) | Creates a new, optimized mask by merging multiple existing masks. |
| isExcluded(...) | boolean | O(F) | The core evaluation method. Checks if a block is excluded by the mask. F is the number of filters. |
| setInverted(boolean inverted) | void | O(1) | **MUTATES STATE.** Toggles the inversion logic for the entire mask. This is not thread-safe. |
| withOptions(...) | BlockMask | O(F) | Returns a *new* BlockMask instance with modified filter options. Follows an immutable pattern. |
| toString() | String | O(F) | Serializes the mask into its canonical string representation. |

## Integration Patterns

### Standard Usage
The intended use is to parse a mask from a configuration source and use it to test blocks within a world operation. The instance is typically created, used for a batch of checks, and then discarded.

```java
// Example: Using a mask to filter blocks for replacement
BlockMask mask = BlockMask.parse("hytale:stone,hytale:dirt");
ChunkAccessor world = ...;
Vector3i min = ...;
Vector3i max = ...;

for (int x = min.getX(); x <= max.getX(); x++) {
    for (int y = min.getY(); y <= max.getY(); y++) {
        for (int z = min.getZ(); z <= max.getZ(); z++) {
            int blockId = world.getBlock(x, y, z);
            // The isExcluded check returns false for stone or dirt, so we invert the logic here.
            if (!mask.isExcluded(world, x, y, z, min, max, blockId)) {
                // This block is a match, perform an action.
                world.setBlock(x, y, z, REPLACEMENT_BLOCK_ID);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation During Operation:** Modifying a BlockMask via **setInverted** while it is actively being used for filtering in another thread or even in a complex single-threaded loop. This leads to unpredictable behavior.

-   **Direct Instantiation:** Avoid using **new BlockMask()**. The static **parse** and **combine** methods perform important optimizations, such as grouping and merging similar filters via the internal **groupFilters** utility. Bypassing these factories can result in less performant mask objects.

-   **Leaked Array Modification:** The **getFilters** method returns the internal array. Modifying this array externally would violate the class's invariants and cause catastrophic failures. It should be treated as read-only.

## Data Pipeline
The BlockMask serves as a deserialized, executable representation of a filtering rule. Its primary data flow is from a static configuration into the core game logic.

> Flow:
> String in Config File (e.g., "hytale:grass_block;hytale:dirt") -> **BlockMask.parse()** -> **BlockMask Instance** -> World Manipulation Service -> **isExcluded(blockId)** -> Boolean Decision -> Block is modified or skipped

