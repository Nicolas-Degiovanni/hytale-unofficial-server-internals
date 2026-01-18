---
description: Architectural reference for IntMap
---

# IntMap

**Package:** com.hypixel.hytale.server.worldgen.climate.util
**Type:** Transient

## Definition
```java
// Signature
public class IntMap {
```

## Architecture & Concepts
The IntMap class is a fundamental, low-level data structure designed for high-performance world generation tasks. It serves as a specialized 2D grid for integer values, abstracting a flat, one-dimensional array into a two-dimensional coordinate system.

Its primary role within the engine is to act as a temporary data store or "scratchpad" during various stages of procedural generation. For example, it is used to hold heightmap data, biome IDs, temperature values, or any other grid-based numerical information required to build a world chunk. By encapsulating the index calculation logic (y * width + x), it provides a cleaner, more intuitive API for algorithms that think in terms of 2D space, while maintaining the memory locality and performance benefits of a contiguous array.

This is a foundational utility, not a high-level system. It is expected to be created and discarded frequently by worker threads in the world generation service.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor, `new IntMap(width, height)`, by a specific world generation algorithm (e.g., a HeightMapGenerator or BiomeSampler).
- **Scope:** The object's lifetime is strictly limited to the scope of the generation task that created it. It is a short-lived object, typically existing only for the duration of a single method execution or a single phase of chunk processing.
- **Destruction:** The IntMap instance becomes eligible for garbage collection as soon as the reference to it goes out of scope. It does not manage any native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The IntMap is a highly **mutable** container. Its core state is the internal `values` integer array. The `width` and `height` fields are final and immutable after construction. The `clear` method resets the state of the `values` array by filling it with -1.
- **Thread Safety:** This class is **not thread-safe**. It provides no internal locking or synchronization. Concurrent read and write operations from multiple threads will result in race conditions and undefined behavior. It is designed to be confined to a single thread for the duration of its lifecycle. Any multi-threaded access must be managed externally with explicit synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| index(int x, int y) | int | O(1) | Computes the 1D array index for the given 2D coordinates. |
| validate(int index) | boolean | O(1) | Checks if a 1D index is within the valid bounds of the internal array. |
| clear() | void | O(N) | Fills the entire map with -1. N is width * height. |
| at(int x, int y) | int | O(1) | Retrieves the value at the specified 2D coordinates. Throws ArrayIndexOutOfBoundsException on invalid coordinates. |
| at(int index) | int | O(1) | Retrieves the value at the specified 1D index. Throws ArrayIndexOutOfBoundsException on invalid index. |
| set(int x, int y, int value) | void | O(1) | Sets the value at the specified 2D coordinates. Throws ArrayIndexOutOfBoundsException on invalid coordinates. |
| set(int index, int value) | void | O(1) | Sets the value at the specified 1D index. Throws ArrayIndexOutOfBoundsException on invalid index. |

## Integration Patterns

### Standard Usage
IntMap is used as a temporary data grid within a single, procedural generation step. The typical pattern is to create it, populate it with data, read from it for further calculations, and then discard it.

```java
// A generator creates an IntMap to store height data for a region.
IntMap heightMap = new IntMap(16, 16);

// The map is populated, often in a nested loop.
for (int y = 0; y < heightMap.height; y++) {
    for (int x = 0; x < heightMap.width; x++) {
        int calculatedHeight = calculateHeightAt(x, y);
        heightMap.set(x, y, calculatedHeight);
    }
}

// Another system consumes the data.
int centerHeight = heightMap.at(8, 8);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never share a single IntMap instance between multiple threads without external locking. The lack of internal synchronization makes it inherently unsafe for concurrent writes.
- **Long-Term Storage:** Do not use IntMap for persistent or long-term storage of world data. It is a transient, in-memory structure designed for intermediate calculations. Serialized chunk data uses a more robust and optimized format.
- **Ignoring Bounds:** Relying on exception handling for out-of-bounds access is an anti-pattern. Algorithms operating on an IntMap should be designed to respect its `width` and `height` properties to prevent frequent runtime exceptions.

## Data Pipeline
IntMap does not represent a pipeline itself, but rather a data container that exists at a specific stage *within* a larger pipeline. It holds the output of one generation stage, which becomes the input for the next.

> Flow:
> Noise Function -> **IntMap** (stores raw noise values) -> Biome Placement Algorithm -> **IntMap** (stores biome IDs) -> Structure Placement Algorithm -> Final Chunk Data

