---
description: Architectural reference for WorldBounds
---

# WorldBounds

**Package:** com.hypixel.hytale.server.worldgen.util.bounds
**Type:** Transient

## Definition
```java
// Signature
public class WorldBounds extends ChunkBounds implements IWorldBounds {
```

## Architecture & Concepts
The WorldBounds class is a fundamental geometric primitive representing a three-dimensional, axis-aligned bounding box (AABB) within the game world. It serves as a core data structure for any system that needs to reason about spatial volumes, most notably the server-side world generation pipeline.

By extending ChunkBounds, it inherits the logic for handling two-dimensional (X and Z) boundaries and extends it to a full 3D volume by adding the vertical Y-axis. This composition makes it a specialized tool for defining regions that span from the bedrock to the sky limit, or any slice in between. Its primary roles include:

-   **Defining Generation Zones:** Procedural generation algorithms use WorldBounds to define their area of operation, ensuring that features like biomes, structures, and cave systems are placed within a designated volume.
-   **Spatial Queries:** Used in collision detection, entity lookups, and culling operations to quickly determine if a point or another volume is contained within or intersects with a given region.
-   **Volume Aggregation:** The class is designed to be mutable, allowing systems to start with a small boundary and progressively expand it by including other points or bounds. This is essential for calculating the total volume occupied by a complex, multi-part object.

## Lifecycle & Ownership
-   **Creation:** WorldBounds instances are created on-demand by systems that require spatial representation. They are not managed by a central service or registry. A typical creator would be a world generator task, a physics query, or a structure placement algorithm. The multiple constructors support creation from explicit coordinates, a single point, or by copying another bounds object.
-   **Scope:** The lifetime of a WorldBounds object is typically short and confined to the scope of the operation that created it. For example, an instance might exist only for the duration of a single procedural generation pass to calculate the extents of a new dungeon.
-   **Destruction:** As a simple data object with no external resource handles, it is managed entirely by the Java Garbage Collector. It is eligible for cleanup as soon as it is no longer referenced.

## Internal State & Concurrency
-   **State:** The internal state is **highly mutable**. It consists of six integer fields representing the minimum and maximum coordinates on the X, Y, and Z axes. Methods like `include` and `expandPositive`/`expandNegative` directly modify this internal state. The object's identity is defined by its current boundaries, which are expected to change throughout its lifecycle.

-   **Thread Safety:** This class is **not thread-safe**. All state-mutating methods perform direct, unsynchronized writes to instance fields. Concurrent modification from multiple threads will result in race conditions, leading to corrupted, inconsistent, or incorrect boundary values.

    **WARNING:** Any use of a shared WorldBounds instance across multiple threads requires explicit, external synchronization (e.g., a `synchronized` block or a `ReentrantLock`). Failure to do so will cause non-deterministic and severe bugs in world generation or physics.

## API Surface
The public API is focused on retrieving boundary values and expanding the volume.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLowBoundY() | int | O(1) | Returns the minimum Y coordinate of the volume. |
| getHighBoundY() | int | O(1) | Returns the maximum Y coordinate of the volume. |
| expandNegative(x, y, z) | void | O(1) | Expands the volume towards negative infinity on all axes. |
| expandPositive(x, y, z) | void | O(1) | Expands the volume towards positive infinity on all axes. |
| include(IChunkBounds) | void | O(1) | Expands this volume to fully contain the provided bounds. |

## Integration Patterns

### Standard Usage
The canonical use case is to initialize a boundary and progressively expand it to encompass multiple points or other volumes. This is common for calculating the total AABB of a complex generated structure.

```java
// How a developer should normally use this
// Start with a bounds representing a single block or point.
WorldBounds totalVolume = new WorldBounds(initialX, initialY, initialZ);

// For each feature or sub-component, expand the total volume.
for (Feature feature : allFeatures) {
    IWorldBounds featureBounds = feature.getBounds();
    totalVolume.include(featureBounds);
}

// The totalVolume now represents the AABB of all combined features.
generator.setOperatingRegion(totalVolume);
```

### Anti-Patterns (Do NOT do this)
-   **Shared Mutable State in Parallel Tasks:** Never pass the same WorldBounds instance to multiple parallel tasks without external locking. Each task should operate on its own local instance and results should be merged sequentially in a thread-safe manner.
-   **Assumption of Immutability:** Do not treat a WorldBounds object received from an external system as immutable. If you need to store or use its value at a specific point in time, create a defensive copy.
    ```java
    // BAD: Storing a reference to a potentially mutable object
    this.cachedBounds = otherSystem.getBounds();

    // GOOD: Creating a defensive copy to ensure stability
    this.cachedBounds = new WorldBounds(otherSystem.getBounds());
    ```

## Data Pipeline
WorldBounds is not a data processing stage itself, but rather a critical data *descriptor* that defines the scope of work for other pipeline stages. It carries spatial context through a system like world generation.

> Flow:
> Generator Configuration -> **new WorldBounds()** -> Procedural Algorithm (uses bounds to constrain work) -> World Voxel Data

