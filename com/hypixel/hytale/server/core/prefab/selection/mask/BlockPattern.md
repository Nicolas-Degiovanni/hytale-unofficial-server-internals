---
description: Architectural reference for BlockPattern
---

# BlockPattern

**Package:** com.hypixel.hytale.server.core.prefab.selection.mask
**Type:** Value Object / Data Structure

## Definition
```java
// Signature
public class BlockPattern {
```

## Architecture & Concepts

The BlockPattern class is a fundamental data structure for representing a weighted, randomized selection of blocks. Its primary role is to decouple game logic from hardcoded block choices, allowing content designers to define complex block palettes in a simple string format. This is heavily utilized in procedural generation, prefab construction, and builder tools where variation is required.

Architecturally, BlockPattern serves as a high-performance, lazily-resolved parser and random sampler. It is designed to be instantiated from a serialized string (e.g., "50%stone,50%dirt") and then efficiently queried for random block IDs.

The core architectural feature is its two-stage resolution process:
1.  **Initial Parsing:** The object is created with a weighted map of *unresolved* block strings. This initial step is lightweight and avoids costly lookups into the global block registry.
2.  **Lazy Resolution:** The first time a block ID is requested via methods like nextBlock, the internal `resolve` method is triggered. This method converts the string identifiers into their final integer block IDs, resolving any nested BlockTypeListAssets in the process. The results are cached internally for all subsequent calls.

This lazy resolution strategy is a critical performance optimization. It ensures that the expensive work of string-to-ID mapping and asset traversal only occurs once, just-in-time, and not during the initial bulk loading of game assets.

## Lifecycle & Ownership

-   **Creation:** BlockPattern instances are almost exclusively created via the static factory method `BlockPattern.parse(String)` or transparently by the framework through its associated `CODEC` during asset deserialization. The static constant `BlockPattern.EMPTY` provides a shared, immutable instance for empty patterns.

-   **Scope:** A BlockPattern is a value object. Its lifetime is bound to the component that owns it, such as a BlockTypeListAsset or a prefab configuration. It holds no external resources and can be freely created and discarded.

-   **Destruction:** Instances are managed by the Java garbage collector and are reclaimed when no longer referenced. No explicit cleanup is required.

## Internal State & Concurrency

-   **State:** The object is **effectively immutable** from an external perspective. While it contains mutable, lazily-initialized caches (`resolvedWeightedMap`), these are write-once and are not exposed. The initial state, defined by the parsed string, is final and cannot be altered after creation. The string representation is also cached upon creation for fast serialization.

-   **Thread Safety:** This class is **NOT THREAD-SAFE**. The `resolve` method, which initializes the internal caches, lacks synchronization. If multiple threads call a method like `nextBlock` on the same unresolved BlockPattern instance simultaneously, they may race to execute the resolution logic. This can lead to redundant work and unpredictable state.

    **WARNING:** Any system that accesses a shared BlockPattern instance from multiple threads must provide its own external synchronization. It is safest to either fully resolve patterns on a single thread before sharing them or to ensure each thread works with its own distinct instance.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(String str) | static BlockPattern | O(N) | **Primary Factory.** Parses a serialized string into a new BlockPattern instance. N is the number of block entries. |
| nextBlock(Random random) | int | O(1) amortized | Returns a random block ID based on the defined weights. Triggers resolution on first call. |
| nextBlockTypeKey(Random random) | BlockEntry | O(1) amortized | Returns a random BlockEntry, including rotation and filler data. Triggers resolution on first call. |
| resolve() | void | O(M\*K) | Explicitly triggers the lazy resolution process. M is the number of entries, K is the cost of asset lookups. |
| isEmpty() | boolean | O(1) | Returns true if the pattern contains no block entries. |

## Integration Patterns

### Standard Usage

The intended use is to parse a pattern from a configuration source and then use it to retrieve random blocks within a generation or placement loop.

```java
// Assume 'patternString' is loaded from a config file, e.g., "75%hytale:stone,25%hytale:dirt"
BlockPattern pattern = BlockPattern.parse(patternString);
Random random = new Random();

// In a world generation loop
for (int i = 0; i < 100; i++) {
    int blockIdToPlace = pattern.nextBlock(random);
    // ... world.setBlock(x, y, z, blockIdToPlace);
}
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Resolution:** Do not share an unresolved BlockPattern instance across multiple threads and call `nextBlock` without external locking. This will cause a race condition.

    ```java
    // BAD - Race condition in resolve()
    BlockPattern sharedPattern = BlockPattern.parse("...");
    executor.submit(() -> sharedPattern.nextBlock(rng));
    executor.submit(() -> sharedPattern.nextBlock(rng));
    ```

-   **Repetitive Parsing:** Do not call `BlockPattern.parse` inside a tight loop. Parse it once and reuse the resulting instance.

    ```java
    // BAD - Inefficient and creates garbage
    for (int i = 0; i < 100; i++) {
        BlockPattern tempPattern = BlockPattern.parse("hytale:stone");
        world.setBlock(x, y, z, tempPattern.firstBlock());
    }
    ```

## Data Pipeline

The BlockPattern acts as a transformation pipeline from a human-readable string to a performance-optimized, integer-based random sampler.

> Flow:
> Serialized String (e.g., "50%stone") -> `BlockPattern.parse()` -> Unresolved `IWeightedMap<String>` -> `nextBlock()` call -> **`resolve()`** -> Cached `IWeightedMap<Integer>` -> Random Block ID

