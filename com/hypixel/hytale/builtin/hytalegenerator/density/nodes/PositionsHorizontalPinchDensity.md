---
description: Architectural reference for PositionsHorizontalPinchDensity
---

# PositionsHorizontalPinchDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class PositionsHorizontalPinchDensity extends Density {
```

## Architecture & Concepts
The PositionsHorizontalPinchDensity is a spatial deformation node within the procedural world generation's density function graph. Its primary function is not to generate density values itself, but to modify the 3D coordinate space before querying a downstream Density node. This process is often referred to as domain warping.

This node effectively "pinches" or "pulls" the terrain horizontally towards or away from a collection of specified points. It achieves this by calculating a "warp vector" for any given sample position. This vector represents the direction and magnitude of the spatial distortion at that point.

The core algorithm operates as follows:
1.  For a given input coordinate, it queries a PositionProvider to find all relevant "pinch points" within a defined `maxDistance`.
2.  For each nearby point, it calculates an individual warp vector. The magnitude of this vector is determined by a user-provided `pinchCurve` function, which maps distance to warp strength.
3.  All individual warp vectors are blended together, weighted by their proximity to the sample coordinate. Closer points have a stronger influence.
4.  The final, blended warp vector is added to the original sample coordinate.
5.  This new, warped coordinate is then passed to the `input` Density function to retrieve the final density value.

This component is essential for creating non-uniform, organic features like canyons, riverbeds, or other formations that require localized, horizontal displacement of the terrain geometry.

## Lifecycle & Ownership
-   **Creation:** Instances are constructed by a higher-level world generator factory or builder during the initialization of the density graph. This class is not intended for direct, ad-hoc instantiation during the generation loop. Its configuration is typically loaded from data files that define the entire generator pipeline.
-   **Scope:** An instance of PositionsHorizontalPinchDensity lives for the entire lifetime of the world generator it is a part of. It is a persistent component of the generation algorithm for a specific world or dimension.
-   **Destruction:** The object is eligible for garbage collection when the parent world generator is discarded, for example, when a server shuts down or loads a different world configuration.

## Internal State & Concurrency
-   **State:** This class is stateful. It maintains references to its `input` density node, the `positions` provider, the `pinchCurve` function, and various configuration parameters. Its most critical stateful component is the `threadData` field, which implements a per-thread cache. This cache stores the last calculated warp vector for a given X/Z column, preventing redundant calculations for vertically stacked samples.

-   **Thread Safety:** This class is designed for high-concurrency environments and is thread-safe through a lock-free, thread-partitioning strategy.
    -   It does **not** use explicit locks or synchronized blocks, which would introduce contention.
    -   Instead, it leverages a `WorkerIndexer.Data` structure, which provides each worker thread with its own private `Cache` object, indexed by the `workerId` from the `Density.Context`.
    -   All mutable state related to the generation process (the cached warp vector) is confined to these thread-local caches. This design eliminates data races and ensures high performance in the parallelized world generation engine.

    **WARNING:** While the `process` method is thread-safe, the object's configuration fields (e.g., `maxDistance`, `input`) are not. Modifying these fields after the generator has started is an unsupported operation and will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(N) | Primary entry point. Calculates the warped position and queries the input density. N is the number of points returned by the PositionProvider within the search radius. |
| setInputs(Density[] inputs) | void | O(1) | Wires the downstream `input` density node into this object. Part of the graph construction phase. |
| calculateWarpVector(Density.Context context) | Vector3d | O(N) | The core logic. Queries for nearby points and computes the blended warp vector. |

## Integration Patterns

### Standard Usage
This class must be used as a node within a larger density graph. It is configured and instantiated by a factory, and its `input` is set via the `setInputs` method. It acts as a filter or modifier that sits between other nodes.

```java
// Conceptual example of graph setup
// This code would exist within a world generator factory

// Create the source of points (e.g., from a structure spawner)
PositionProvider myPoints = new SomePositionProvider(...);

// Create the density function to be warped (e.g., basic noise)
Density terrainNoise = new SimplexNoiseDensity(...);

// Create and configure the pinch node
PositionsHorizontalPinchDensity pinch = new PositionsHorizontalPinchDensity(
    terrainNoise,
    myPoints,
    distance -> 1.0 - distance, // A simple linear pinch curve
    256.0, // maxDistance
    true,  // distanceNormalized
    0.0,   // positionsMinY
    255.0, // positionsMaxY
    context.getThreadCount()
);

// The 'pinch' node would then be used by other parts of the generator
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PositionsHorizontalPinchDensity()` in game logic. It must be configured and managed as part of the world generator's initialization.
-   **State Mutation:** Do not modify public or internal fields after the density graph has been built and the generator is active. Treat the object as immutable post-construction to prevent inconsistent world generation.
-   **Incorrect Thread Count:** The `threadCount` parameter in the constructor must match the number of worker threads in the generator. A mismatch will cause `ArrayIndexOutOfBoundsException` or incorrect caching.

## Data Pipeline
The flow of data for a single `process` call demonstrates its role as a coordinate transformer.

> Flow:
> `Density.Context` (Original Position) -> **PositionsHorizontalPinchDensity.process()** -> Query `PositionProvider` -> **calculateWarpVector()** -> Blend vectors using `pinchCurve` -> Create new `Density.Context` (Warped Position) -> `input.process()` -> Final `double` density value

