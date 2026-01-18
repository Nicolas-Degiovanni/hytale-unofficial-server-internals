---
description: Architectural reference for ChunkBounds
---

# ChunkBounds

**Package:** com.hypixel.hytale.server.worldgen.util.bounds
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ChunkBounds implements IChunkBounds {
```

## Architecture & Concepts
The ChunkBounds class is a fundamental geometric primitive within the server-side world generation engine. It represents a mutable, two-dimensional, axis-aligned bounding box (AABB) defined by integer coordinates in the chunk coordinate system (X, Z).

Its primary role is to define and constrain the operational area for procedural generation algorithms. Systems such as biome placement, structure generation, and terrain decoration instantiate ChunkBounds to specify which chunks they are permitted to read from or write to. It serves as the foundational data structure for spatial partitioning and region management during the world generation pipeline, ensuring that distinct generation passes do not interfere with each other and operate within their designated regions.

This class is not a service or a manager; it is a lightweight data container designed for high-frequency, temporary allocation by generation tasks.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand, typically at the beginning of a specific world generation task or method call. They are not managed by a central registry or dependency injection framework. The common pattern is direct instantiation via the `new` keyword.

- **Scope:** The lifetime of a ChunkBounds object is intentionally short. It is scoped to the execution of a single generation pass, such as the placement of a single dungeon or the decoration of one biome region. Once the task is complete, the object is no longer referenced.

- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as it falls out of the scope in which it was created. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** The internal state is **highly mutable**. The core design of the class is centered around methods like `include` and `expand` which directly modify the internal `minX`, `minZ`, `maxX`, and `maxZ` fields. It is a stateful object that typically starts as a single point or an inverted box and is progressively enlarged to encompass a desired area.

- **Thread Safety:** This class is **not thread-safe**. All methods perform direct, unsynchronized writes to its member fields. Sharing a single ChunkBounds instance across multiple threads for concurrent modification will result in race conditions, leading to a corrupted and unpredictable bounding box.

    **WARNING:** Each world generation task running on a separate thread must create and operate on its own private ChunkBounds instance. Do not share instances between worker threads.

## API Surface
The public API is designed for efficient geometric manipulation of the bounding box.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ChunkBounds() | constructor | O(1) | Creates an empty, inverted box, ready to be expanded. |
| ChunkBounds(x, z) | constructor | O(1) | Creates a 1x1 bounding box at the specified chunk coordinate. |
| include(x, z) | void | O(1) | Expands the bounds, if necessary, to contain the given point. |
| include(IChunkBounds) | void | O(1) | Expands the bounds to fully contain the area of another bounds object. |
| expandNegative(x, z) | void | O(1) | Shifts the minimum corner (minX, minZ) outwards (more negative). |
| expandPositive(x, z) | void | O(1) | Shifts the maximum corner (maxX, maxZ) outwards (more positive). |
| getLowBoundX() | int | O(1) | Returns the minimum X coordinate of the bounding box. |
| getHighBoundX() | int | O(1) | Returns the maximum X coordinate of the bounding box. |

## Integration Patterns

### Standard Usage
The canonical use case is to define an area of interest for a world generation algorithm, build it up, and then pass it to a subsystem that performs work within that area.

```java
// A world generator defining the area for a new feature.
// 1. Start with the core chunk coordinate.
ChunkBounds featureBounds = new ChunkBounds(chunkX, chunkZ);

// 2. Expand the bounds to include a safety radius for the feature.
featureBounds.expandNegative(10, 10);
featureBounds.expandPositive(10, 10);

// 3. Include another known point of interest, like a nearby landmark.
featureBounds.include(landmark.getChunkX(), landmark.getChunkZ());

// 4. Pass the final, calculated bounds to the generation task.
structurePlacementSystem.generateInBounds(featureBounds);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use Across Threads:** Never pass the same ChunkBounds instance to multiple concurrent tasks. This will cause data corruption. Each thread must have its own instance.

- **Misunderstanding `expand` vs `include`:** Using `expandPositive(5, 5)` on a 1x1 box results in a 6x6 box. Using `include` to add a point 5 units away results in a 5x5 box. Confusing these operations is a common source of off-by-one errors and incorrect region calculations.

- **Assuming Immutability:** Do not pass a ChunkBounds object to a method and assume it will remain unchanged. Methods that accept a ChunkBounds are often expected to modify it. If you need to preserve the original, create a copy: `new ChunkBounds(originalBounds)`.

## Data Pipeline
ChunkBounds does not process a stream of data itself. Instead, it acts as a critical input parameter that *defines the scope* for a subsequent data pipeline. It is a gatekeeper for world generation processes.

> Flow:
> Zone Definition Parameters -> **ChunkBounds** (Instantiation & Configuration) -> Biome Generator -> Modified Chunk Data

