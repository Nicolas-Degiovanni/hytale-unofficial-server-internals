---
description: Architectural reference for BlockFilter
---

# BlockFilter

**Package:** com.hypixel.hytale.server.core.prefab.selection.mask
**Type:** Transient

## Definition
```java
// Signature
public class BlockFilter {
```

## Architecture & Concepts

The BlockFilter is a predicate object responsible for evaluating complex spatial and type-based rules against blocks in the game world. It serves as a core component for any system that needs to selectively modify or query regions of the world, such as builder tools, procedural generation, and command systems.

Its primary architectural function is to decouple the definition of a selection rule from its execution. Rules are defined using a concise, human-readable string format (e.g., `!>stone|dirt` which means "not above stone or dirt"). This string is parsed into an immutable BlockFilter instance. The instance can then be passed to world manipulation systems and executed efficiently against potentially millions of blocks.

A key design feature is the lazy, on-demand resolution of block names. The filter is constructed with string identifiers for blocks. Only when a check is first performed does the filter query the server's asset registries to convert these names into highly optimized integer IDs. This defers the cost of asset lookups until they are absolutely necessary.

The system is built around the internal `FilterType` enum, which defines the geometric nature of the test:
*   **TargetBlock:** Does the block at the given coordinate match the list?
*   **AdjacentBlock:** Does any block orthogonally adjacent to the coordinate match?
*   **NeighborBlock:** Does any of the 26 surrounding blocks match?
*   **Directional:** Does the block to the North, East, South, West, Above, or Below match?
*   **Selection:** Is the coordinate within a predefined bounding box?

## Lifecycle & Ownership

-   **Creation:** BlockFilter instances are almost exclusively created by deserializing a string representation. This is typically done via the static `BlockFilter.parse(String)` method or implicitly through the provided `BlockFilter.CODEC` when loading assets (e.g., a builder tool's configuration from JSON). Direct instantiation using `new BlockFilter(...)` is rare.
-   **Scope:** The lifetime of a BlockFilter is tied to its owner. If it is part of an asset definition, it lives as long as that asset is loaded in memory. If created for a single command execution, it is short-lived and eligible for garbage collection once the operation completes.
-   **Destruction:** Managed entirely by the Java garbage collector. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency

-   **State:** The object is stateful and contains a mutable cache. Initial properties like `blockFilterType`, `blocks` (as strings), and `inverted` are set at construction and are effectively immutable. The `resolvedBlocks` and `resolvedFluids` fields, which are integer sets for fast lookups, are lazily populated on the first call to a matching method (e.g., `isExcluded`).
-   **Thread Safety:** **This class is not thread-safe.** The lazy initialization of `resolvedBlocks` within the `resolve` method is susceptible to race conditions. If multiple threads access a single, unresolved BlockFilter instance concurrently, the resolution logic may be executed multiple times, and updates to the internal state are not guaranteed to be safely published across threads.

    **WARNING:** Any BlockFilter instance that may be accessed by multiple threads must be externally synchronized. A common pattern is to explicitly call `resolve()` from a single thread during an initialization phase before sharing the instance.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(String str) | static BlockFilter | O(N) | Parses a filter string into a BlockFilter instance. Throws if format is invalid. |
| resolve() | void | O(M*L) | **Critical.** Populates internal integer ID caches from block strings. M is number of block strings, L is asset lookup cost. Idempotent. |
| isExcluded(...) | boolean | O(1) | The primary predicate method. Returns true if the block at the given coordinates should be excluded based on the filter rules. |
| toString() | String | O(1) | Returns the original string representation of the filter. This value is cached at construction. |

## Integration Patterns

### Standard Usage

The intended use is to parse a filter from a configuration source once, resolve it, and then reuse the instance for all subsequent checks within an operation.

```java
// 1. Define the filter rule as a string (e.g., from a config file)
String filterRule = ">stone|dirt";

// 2. Parse the rule into an object once, outside of any loops
BlockFilter filter = BlockFilter.parse(filterRule);

// 3. (Optional but recommended) Pre-resolve the filter during initialization
filter.resolve();

// 4. Use the filter instance repeatedly inside a world-iteration loop
for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        // ... get blockId at position ...
        if (!filter.isExcluded(chunkAccessor, x, y, z, min, max, blockId)) {
            // This block is included, perform action
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Parsing in a Loop:** Never call `BlockFilter.parse()` inside a tight loop. This creates excessive object churn and repeatedly parses the same string. Parse once, reuse the object.
-   **Concurrent Unresolved Access:** Using a shared BlockFilter instance across multiple threads without first calling `resolve()` or providing external locking can lead to race conditions and unpredictable behavior.
-   **String Manipulation:** Do not attempt to build filter logic by manipulating the filter strings. The `BlockFilter` object provides a structured way to represent these rules; use the object model.

## Data Pipeline

The BlockFilter acts as a transformation component, converting a high-level string definition into a low-level, high-performance predicate function.

> Flow:
> 1.  **Configuration String** (e.g., `"!~grass_block"`) from an asset file or command argument.
> 2.  `BlockFilter.CODEC` or `BlockFilter.parse()` deserializes the string into a `BlockFilter` object in memory.
> 3.  The `resolve()` method is invoked, which queries global registries like `BlockType.getAssetMap()` and `Item.getAssetMap()`.
> 4.  String identifiers ("grass_block") are converted into optimized integer sets (`resolvedBlocks`).
> 5.  During a world operation, the `isExcluded(ChunkAccessor, ...)` method is called with world data.
> 6.  The method performs cheap integer comparisons and spatial checks against the `ChunkAccessor`.
> 7.  A final **Boolean Result** is returned, indicating whether to include or exclude the block.

