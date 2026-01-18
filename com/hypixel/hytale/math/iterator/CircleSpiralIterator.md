---
description: Architectural reference for CircleSpiralIterator
---

# CircleSpiralIterator

**Package:** com.hypixel.hytale.math.iterator
**Type:** Transient

## Definition
```java
// Signature
public class CircleSpiralIterator {
```

## Architecture & Concepts
The CircleSpiralIterator is a high-performance, specialized algorithm for traversing a 2D grid of chunks in an outward spiral, filtered by a circular or annular (ring-shaped) boundary. Its primary purpose is to enable efficient, proximity-based queries for systems that operate on a radius around a central point, such as Level of Detail (LOD) management, entity activation, or procedural generation.

The core design prioritizes performance by avoiding expensive mathematical operations within its main loop. It achieves this through two key techniques:
1.  **Square Spiral Traversal:** It walks a square grid pattern, which is computationally simple to advance, rather than calculating points on a perfect circle.
2.  **Squared Distance Comparison:** It compares the squared distance of each point from the center against pre-calculated squared radii. This completely avoids the use of costly square root calculations for every point in the iteration.

This class is designed as a stateful, reusable object. It is not a true Java Iterator and does not implement the Iterator interface, but it follows the same conceptual contract with its `hasNext` and `next` methods.

## Lifecycle & Ownership
-   **Creation:** Instantiated directly via its default constructor: `new CircleSpiralIterator()`. It is a lightweight object intended for frequent creation or pooling.
-   **Scope:** The lifecycle of an iterator is tied to a single, complete traversal task. An owner system (e.g., a ChunkManager) creates an instance, calls `init` to configure the search parameters, consumes the results in a loop, and then either discards the instance or calls `reset` to prepare it for a new task.
-   **Destruction:** The object holds no native resources and is managed entirely by the Java garbage collector. The `reset` method allows for efficient object reuse, which is the recommended pattern in performance-critical loops to avoid GC pressure.

## Internal State & Concurrency
-   **State:** The CircleSpiralIterator is highly mutable. It maintains internal state for the spiral's center coordinates (`chunkX`, `chunkZ`), current relative position (`x`, `z`), current traversal direction (`dx`, `dz`), and iteration progress (`i`). It also caches the next valid chunk index in the `nextChunk` field to correctly implement the `hasNext`/`next` contract.

-   **Thread Safety:** **WARNING:** This class is fundamentally **not thread-safe**. Its internal state is mutated on nearly every method call without any synchronization. Sharing a single instance across multiple threads will lead to state corruption and non-deterministic behavior. Each thread requiring a spiral traversal must use its own dedicated instance of CircleSpiralIterator.

## API Surface
The public contract is designed for a simple `initialize -> loop -> reset` pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(chunkX, chunkZ, radiusTo) | void | O(1) | Configures the iterator for a circular area from radius 0 to radiusTo. |
| init(chunkX, chunkZ, radiusFrom, radiusTo) | void | O(1) | Configures the iterator for an annular (ring) area. Throws IllegalArgumentException for invalid radii. |
| reset() | void | O(1) | Resets the internal state, allowing the object to be reused for a new iteration. |
| next() | long | Amortized O(1) | Returns the next chunk index as a packed long. Throws IllegalStateException if not initialized or NoSuchElementException if the iteration is complete. |
| hasNext() | boolean | Amortized O(1) | Returns true if more chunks exist within the defined area. This method drives the internal state forward to find the next valid chunk. |
| getCurrentRadius() | int | O(sqrt) | **Warning:** Incurs a performance cost due to `Math.sqrt`. Returns the radius of the current spiral layer being processed. |
| getCompletedRadius() | int | O(sqrt) | **Warning:** Incurs a performance cost due to `Math.sqrt`. Returns the radius of the last fully completed spiral layer. |

## Integration Patterns

### Standard Usage
The intended use is to create an instance, initialize it for a specific query, and consume it with a standard `while` loop. For performance-critical systems, the object should be reused via the `reset` method.

```java
// How a developer should normally use this
CircleSpiralIterator iterator = new CircleSpiralIterator();
int centerX = 10;
int centerZ = 20;
int searchRadius = 8;

// Initialize for a search around (10, 20) with a radius of 8 chunks
iterator.init(centerX, centerZ, searchRadius);

while (iterator.hasNext()) {
    long chunkIndex = iterator.next();
    // Process the chunk index, e.g., schedule it for loading
    world.loadChunk(chunkIndex);
}

// The iterator can now be reset and reused for another query
iterator.reset();
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Access:** Never share a single instance of CircleSpiralIterator between threads. This will break its internal state. Create a new instance for each thread.
-   **Ignoring Initialization:** Calling `next()` or `hasNext()` before `init()` will result in an `IllegalStateException`. The iterator is unusable until it is configured.
-   **Re-use Without Reset:** Attempting to call `init()` on an already-used iterator without first calling `reset()` will lead to incorrect behavior. The `setup` flag prevents method calls, but a proper reset is required for logical correctness.

## Data Pipeline
The iterator transforms a geometric query into a linear sequence of chunk identifiers.

> Flow:
> Center Coordinates & Radii -> `init()` -> **Internal State (center, radiiSq, position)** -> `hasNext()`/`next()` call -> **`prepareNext()` method** (Advances spiral, performs squared-distance check) -> Filtered `long` Chunk Index

