---
description: Architectural reference for JitterPointField
---

# JitterPointField

**Package:** com.hypixel.hytale.builtin.hytalegenerator.fields.points
**Type:** Transient

## Definition
```java
// Signature
public class JitterPointField extends PointField {
```

## Architecture & Concepts

The JitterPointField is a procedural generation primitive that populates a spatial volume with a set of pseudo-random points. It implements the PointField contract, establishing it as a source of positional data for higher-level world generation systems, such as biome object placement or ore vein distribution.

Its core architectural pattern is the **Jittered Grid**, also known as a Cellular Point Field. Instead of generating truly random points, which can lead to undesirable clustering and gaps, this class first divides space into a uniform grid of cells. It then generates exactly one point within each cell, with its position randomly displaced or "jittered" from the cell's center. This technique guarantees a minimum distance between points, resulting in a more uniform and aesthetically pleasing distribution, often referred to as a Poisson-like distribution.

The underlying pseudo-random displacement is provided by an internal FastNoiseLite instance. The JitterPointField acts as an abstraction layer over this noise library, translating a high-level request for points within a bounding box into a series of low-level noise queries organized by a grid structure. The scaling parameters control the density of this grid, allowing developers to define how sparse or dense the point distribution is along each axis.

## Lifecycle & Ownership

-   **Creation:** An instance is created directly via its constructor, `new JitterPointField(seed, jitter)`. It is typically instantiated by a world generation orchestrator or a specific feature generator that requires a set of uniformly distributed points.
-   **Scope:** The object's lifetime is ephemeral and tied to the specific generation task for which it was created. It holds the state necessary for a single, coherent point-generation operation (e.g., placing all trees in a single chunk). It is not intended to be a long-lived service.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the orchestrator that created it releases its reference. There are no manual cleanup or disposal methods.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. It encapsulates the generation `seed`, `jitter` magnitude, a configured `FastNoiseLite` instance, and a set of scaling vectors. The `setScale` method directly mutates the internal scaling state. This state is fundamental to its operation and is not shared across instances.

-   **Thread Safety:** **This class is not thread-safe.** The internal state, particularly the scaling vectors, can be modified via the `setScale` method. Concurrent calls to `setScale` and any of the `points...` methods will result in a race condition and produce non-deterministic, incorrect output.

    **WARNING:** Each instance of JitterPointField must be confined to a single thread. Do not share instances between worker threads in a parallelized world generation system. Instead, create a new instance for each thread or task.

## API Surface

The public contract is designed for configuration followed by a single type of execution: streaming points within a given volume.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| JitterPointField(seed, jitter) | constructor | O(1) | Creates a new field generator with a specific seed and jitter magnitude. |
| setScale(scaleX, scaleY, scaleZ, scaleW) | PointField | O(1) | Configures the density of the underlying grid. **WARNING:** This method mutates internal state and is not thread-safe. |
| points3d(min, max, pointsOut) | void | O(N) | Generates all 3D points within the bounding box. N is the number of grid cells intersecting the volume. |
| points3i(min, max, pointsOut) | void | O(N) | Convenience wrapper for `points3d` that rounds results to integer vectors. |

## Integration Patterns

### Standard Usage

The intended pattern is to instantiate, configure the scale, and then invoke a point generation method to stream results into a collection or process them immediately.

```java
// How a developer should normally use this
int seed = 12345;
double jitter = 0.8; // 80% jitter
JitterPointField treePlacementField = new JitterPointField(seed, jitter);

// Set the grid density. Here, one cell every 16 units on X/Z, 32 on Y.
treePlacementField.setScale(16.0, 32.0, 16.0, 1.0);

Vector3i chunkMin = new Vector3i(0, 0, 0);
Vector3i chunkMax = new Vector3i(16, 256, 16);
List<Vector3i> treePositions = new ArrayList<>();

// Stream all valid tree positions within the chunk bounds into the list.
treePlacementField.points3i(chunkMin, chunkMax, treePositions::add);
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Modification:** Accessing a single JitterPointField instance from multiple threads is unsafe and will lead to data corruption. Do not share instances across generation workers.
-   **Mid-Stream Reconfiguration:** Calling `setScale` after beginning a point generation process or between calls to `points...` for the same logical operation can produce inconsistent boundaries and artifacts. An instance should be configured once per logical task.
-   **Ignoring Bounding Box:** Do not assume that every point generated by the noise function is valid. The internal logic correctly filters points that are jittered outside of the requested bounding box. Relying on unfiltered output from a modified version of this class would place objects outside their intended region.

## Data Pipeline

The flow of data through this component involves transforming a spatial query into a set of discrete points through scaling, grid iteration, and noise application.

> Flow:
> 1.  **Input:** A world-space bounding box (e.g., a chunk volume) is provided to a `points...` method.
> 2.  **Coordinate Scaling:** The world-space bounding box is scaled *down* using the internal `scaleDown` vectors to determine a corresponding bounding box in "cell-space".
> 3.  **Grid Iteration:** The system iterates through every integer coordinate within the cell-space bounding box. Each integer coordinate represents a single grid cell.
> 4.  **Noise Query:** For each cell coordinate, the `FastNoiseLite.pointFor` method is invoked with the `seed`, `jitter`, and cell coordinate. This produces a single, deterministically random point within that unit cell.
> 5.  **Coordinate Un-scaling:** The resulting point from cell-space is scaled *up* using the `scaleUp` vectors to convert it back into a world-space coordinate.
> 6.  **Bounds Check:** The final world-space point is checked to ensure it lies within the original input bounding box. This is a critical step, as the jitter can push a point outside the requested volume.
> 7.  **Output Stream:** If the point is valid, it is passed to the output `Consumer` callback.

