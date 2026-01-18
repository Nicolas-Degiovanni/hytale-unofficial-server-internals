---
description: Architectural reference for SpiralIterator
---

# SpiralIterator

**Package:** com.hypixel.hytale.math.iterator
**Type:** Transient Utility

## Definition
```java
// Signature
public class SpiralIterator {
```

## Architecture & Concepts
The SpiralIterator is a low-level, stateful mathematical utility designed to traverse a 2D grid of chunk coordinates in an outward-spiraling pattern. It serves as a foundational primitive for any system requiring proximity-based processing, abstracting the complex logic of calculating spiral steps into a standard iterator interface.

Its primary function is to provide an ordered sequence of chunk coordinates, starting from a central point and expanding outwards. This is critical for systems such as:
*   **World Streaming:** Loading and unloading chunks around a player or camera, ensuring the nearest chunks are processed first.
*   **AI & Pathfinding:** Area-of-effect scanning or searching for targets in increasing circles of distance.
*   **Resource Discovery:** Procedurally scanning an area for resources or points of interest.

The iterator operates on chunk coordinates, which are encoded into a single long value via the ChunkUtil helper class. It does not store the full list of coordinates; rather, it calculates each coordinate on-the-fly with each call to the next method, making it highly memory-efficient for iterating over large radii.

## Lifecycle & Ownership
- **Creation:** An instance is created directly using the `new` keyword. The object is created in an uninitialized state. A subsequent call to one of the `init` methods is mandatory to configure the iterator's starting point, radius, and internal state.
- **Scope:** The SpiralIterator is a short-lived object. Its lifecycle is typically bound to the scope of a single method or task, such as a single pass of a chunk loading algorithm. It is not designed to be a long-lived service or component.
- **Destruction:** The object is managed by the Java garbage collector and is eligible for cleanup once it is no longer referenced. The `reset` method can be used to invalidate the iterator for reuse, but the more common pattern is to create a new instance for each distinct iteration task.

## Internal State & Concurrency
- **State:** The SpiralIterator is highly **mutable**. Its internal fields, including the current index `i`, relative coordinates `x` and `z`, and direction `dx` and `dz`, are updated on every call to `next`. The object is effectively a state machine that tracks its position within the spiral.

- **Thread Safety:** This class is **NOT thread-safe**. It contains no internal locking or synchronization. Concurrent calls to `next` from multiple threads will result in a corrupted internal state, leading to an unpredictable and incorrect iteration sequence.

    **WARNING:** A single SpiralIterator instance must never be shared across threads. Each thread requiring a spiral iteration must create and manage its own private instance.

## API Surface
The public contract is designed around the standard Java iterator pattern, supplemented by initialization and state inspection methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(chunkX, chunkZ, radiusTo) | void | O(1) | Configures the iterator for a new spiral traversal. Throws IllegalArgumentException for invalid radii. |
| init(chunkX, chunkZ, radiusFrom, radiusTo) | void | O(1) | Configures the iterator to traverse a "ring" between two radii. |
| next() | long | O(1) | Returns the next chunk coordinate in the sequence and advances the internal state. Throws IllegalStateException if not initialized. |
| hasNext() | boolean | O(1) | Returns true if the iterator has more coordinates to process within the specified maximum radius. |
| reset() | void | O(1) | Invalidates the iterator's setup state, requiring a new call to `init` before further use. |

## Integration Patterns

### Standard Usage
The canonical use involves creating an instance, initializing it, and then consuming its coordinates in a `while` loop.

```java
// Standard pattern for iterating a 10-chunk radius around (0,0)
SpiralIterator iterator = new SpiralIterator();
iterator.init(0, 0, 10);

while (iterator.hasNext()) {
    long chunkCoord = iterator.next();
    // Process the chunk coordinate, e.g., queue for loading
    worldLoader.queueChunk(chunkCoord);
}
```

### Anti-Patterns (Do NOT do this)
- **Failing to Initialize:** Creating an iterator and calling `next` without first calling `init` will result in an `IllegalStateException`. The default constructor produces an object that is not ready for use.
- **Concurrent Access:** Sharing a single instance across multiple threads for simultaneous iteration. This will corrupt the iterator's state and produce undefined behavior.
- **State Modification:** Relying on the internal state (e.g., `getX`, `getZ`) for critical logic outside of debugging. The public contract is primarily `hasNext` and `next`; internal coordinate state is an implementation detail.

## Data Pipeline
The SpiralIterator acts as a data generator, not a processor. It transforms a configuration into a stream of data.

> Flow:
> Configuration (Center Point, Radius) -> **SpiralIterator.init()** -> **SpiralIterator.next()** -> Stream of `long` Chunk Coordinates -> Consuming System (e.g., World Loader, AI Scanner)

