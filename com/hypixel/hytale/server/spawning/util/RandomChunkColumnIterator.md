---
description: Architectural reference for RandomChunkColumnIterator
---

# RandomChunkColumnIterator

**Package:** com.hypixel.hytale.server.spawning.util
**Type:** Transient

## Definition
```java
// Signature
public class RandomChunkColumnIterator {
```

## Architecture & Concepts

The RandomChunkColumnIterator is a stateful utility designed for procedural generation and entity spawning systems. Its primary function is to provide a sequence of random, non-repeating coordinates within a single chunk column. A chunk column represents a 32x32 vertical slice of the world, containing 1024 possible horizontal positions.

This class is not a generic iterator. It is purpose-built to solve a common problem in procedural generation: how to place a set of features (e.g., trees, rocks, spawn points) randomly within an area without overlap and in a repeatable manner.

The core of its operation relies on two key components:
1.  A **ChunkColumnMask**, which acts as a bitmask to track the 1024 available positions. As the iterator yields a position, the corresponding bit in the mask is cleared to prevent it from being selected again.
2.  A seeded **Random** number generator. The ability to provide a deterministic seed, typically derived from the world chunk's coordinates, is the most critical architectural feature. This ensures that the sequence of generated positions is identical every time that specific chunk is generated, guaranteeing world consistency across different game sessions and servers.

When all available positions have been exhausted, the iterator automatically resets itself, allowing for continuous, cyclical iteration if required.

## Lifecycle & Ownership

-   **Creation:** An instance is created on-demand by a higher-level system responsible for a specific, scoped task, such as a BiomeFeatureSpawner or a mob spawning manager. It is typically instantiated at the beginning of a method that needs to place objects within a chunk. The choice of constructor is critical:
    -   The default constructor is used for non-deterministic operations.
    -   Constructors accepting a WorldChunk are used for deterministic world generation.
    -   Constructors accepting a ChunkColumnMask are used to constrain spawning to a specific sub-region of the chunk.

-   **Scope:** The object's lifetime is intentionally short and is strictly bound to the scope of the single spawning or generation operation for which it was created. It should be treated as a method-local resource.

-   **Destruction:** The object is eligible for garbage collection as soon as the method it was created in completes and no more references to it exist. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency

-   **State:** This class is highly mutable and stateful. Its internal state includes:
    -   The `availablePositions` bitmask, which is modified on every call to `nextPosition`.
    -   The internal state of the `java.util.Random` instance.
    -   The `currentIndex` of the last position returned.

-   **Thread Safety:** **This class is not thread-safe.** Its internal state is accessed and modified without any synchronization mechanisms. Sharing an instance of RandomChunkColumnIterator across multiple threads will result in race conditions, an invalid internal state, and non-deterministic behavior.

    **WARNING:** All interactions with a RandomChunkColumnIterator instance must be confined to the thread that created it, typically the main server thread or a dedicated world generation worker thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nextPosition() | int | O(1) | Returns the index of the next random, unvisited position. Resets and continues if all positions are exhausted. |
| nextPositionAvoidBorders() | int | O(1) | A specialized variant of nextPosition that attempts to select a position away from the edges of contiguous available areas. |
| markPositionVisited() | void | O(1) | Manually marks the current position as visited, removing it from the pool of available positions. |
| reset() | void | O(1) | Resets the iterator to its initial state, making all original positions available again and resetting the random seed. |
| saveIteratorPosition() | void | O(1) | Bookmarks the current number of remaining positions. |
| isAtSavedIteratorPosition() | boolean | O(1) | Checks if the number of remaining positions matches the last saved bookmark. |

## Integration Patterns

### Standard Usage

The most common use case is to create a deterministic iterator for placing features during world generation. The iterator is created, used in a loop to place a certain number of features, and then discarded.

```java
// Standard pattern for deterministic feature placement in a chunk.
void placeFeaturesInChunk(WorldChunk chunk, ChunkColumnMask spawnableArea) {
    // The iterator is seeded with the chunk's coordinates for determinism.
    RandomChunkColumnIterator iterator = new RandomChunkColumnIterator(spawnableArea, chunk);

    int featuresToPlace = 10;
    for (int i = 0; i < featuresToPlace && iterator.positionsLeft() > 0; i++) {
        int positionIndex = iterator.nextPosition();
        int x = ChunkUtil.xFromColumn(positionIndex);
        int z = ChunkUtil.zFromColumn(positionIndex);

        // ... logic to place a feature at world coordinates (x, z)
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Caching or Re-use:** Do not cache and re-use an iterator instance for different spawning operations (e.g., for placing trees and then for placing ores). The internal state will be incorrect. **Always create a new instance for each distinct task.**

-   **Cross-Thread Access:** Never pass an instance of this iterator to another thread or access it concurrently. This will corrupt its state.

-   **Using Default Constructor for World Generation:** Using `new RandomChunkColumnIterator()` for procedural world generation is a critical error. It uses a random, time-based seed, which will cause the world to generate differently every time the server starts. Always use the constructor that accepts a WorldChunk to ensure determinism.

## Data Pipeline

RandomChunkColumnIterator acts as a generator, not a processor. It originates coordinate data based on an initial configuration.

> Flow:
> (Input) WorldChunk + ChunkColumnMask -> **RandomChunkColumnIterator** (Instantiation with deterministic seed) -> Call to `nextPosition()` -> (Output) `int` position index -> `ChunkUtil` coordinate conversion -> World Generation System (places block/entity)

