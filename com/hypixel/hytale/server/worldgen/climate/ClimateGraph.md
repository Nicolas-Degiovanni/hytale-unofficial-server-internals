---
description: Architectural reference for ClimateGraph
---

# ClimateGraph

**Package:** com.hypixel.hytale.server.worldgen.climate
**Type:** Transient

## Definition
```java
// Signature
public class ClimateGraph {
```

## Architecture & Concepts

The ClimateGraph is a foundational component of the server-side world generation system, responsible for mapping abstract climate parameters—specifically temperature and intensity—to concrete ClimateType definitions. It functions as a pre-computed spatial acceleration structure, transforming a complex, hierarchical graph of climate definitions into a highly optimized 2D lookup table.

At its core, the system solves the problem of efficiently determining which climate or biome should exist at a given point in a conceptual 2D space. Instead of performing expensive nearest-neighbor calculations against all possible climate points during real-time chunk generation, ClimateGraph performs this calculation once at initialization. It generates a discrete grid, or table, of a fixed RESOLUTION (512x512), where each cell stores the ID of the dominant ClimateType for that coordinate.

The generation process is analogous to creating a Voronoi diagram. The input is a tree of ClimateType objects, where each type is defined by one or more ClimatePoint anchors in the 2D space. The `populateTable` method iterates through every pixel of the internal grid and finds the nearest ClimatePoint, thereby assigning that pixel to the corresponding ClimateType.

A secondary but critical output is the `fade` map. This map is generated using a DistanceTransform algorithm, which calculates each pixel's distance to the nearest climate boundary. This data is essential for downstream systems to create smooth, natural transitions between biomes, preventing hard edges.

In essence, ClimateGraph is an optimization pattern that trades a one-time memory and CPU cost during startup for extremely fast, constant-time O(1) lookups during the performance-critical phase of world generation.

### Lifecycle & Ownership
-   **Creation:** An instance is created by a higher-level world generation manager, likely during server startup or when a new world's dimension data is loaded. The constructor requires a complete, configured graph of parent ClimateType objects.
-   **Scope:** The object is long-lived. Due to the significant computational cost of its constructor, it is designed to persist for the entire lifecycle of a server session or until the world's climate configuration is reloaded.
-   **Destruction:** The object has no explicit destruction or cleanup methods. It is reclaimed by the Java garbage collector when the world generation service that created it is shut down and all references to it are released.

## Internal State & Concurrency
-   **State:** ClimateGraph is a highly stateful object. Its primary state consists of the `table` (IntMap) and `fade` (DoubleMap), which are large, mutable arrays populated during construction. After the constructor completes, the object's state is effectively immutable unless the `refresh` method is invoked. It also maintains several lookup maps like `id2TypeLookup` to bridge between integer IDs and ClimateType objects.

-   **Thread Safety:** The class is **not** inherently thread-safe for mutations. While read operations like `getType` or `getFade` are safe to call concurrently from multiple world-generation threads (as they only read from the pre-computed arrays), any call to the `refresh` method is a write operation that rebuilds the internal tables.

    **WARNING:** Calling `refresh` from one thread while other threads are performing lookups will result in race conditions, undefined behavior, and likely server instability. The responsibility for synchronizing access during a refresh operation lies entirely with the calling system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| refresh() | void | High | Re-runs the entire table population and distance transform process. This is a very expensive operation. |
| getType(double x, double y) | ClimateType | O(1) | The primary query method. Converts normalized coordinates into a final ClimateType. |
| getId(double x, double y) | int | O(1) | Retrieves the raw integer ID of the climate at the given coordinates. Faster than getType if the caller can cache the ID-to-Type lookup. |
| getFade(double x, double y) | double | O(1) | Retrieves the calculated fade value, scaled by the configured fadeDistance. This indicates proximity to a biome border. |
| getType(int id) | ClimateType | O(1) | Converts a previously fetched climate ID back into its corresponding ClimateType object. |

## Integration Patterns

### Standard Usage

The ClimateGraph should be instantiated once by a central world or dimension manager and stored for reuse. During chunk generation, worker threads query the shared instance with normalized temperature and intensity values produced by noise generators.

```java
// During world initialization
ClimateType[] climateRoots = loadClimateConfiguration();
ClimateGraph climateGraph = new ClimateGraph(512, climateRoots, FadeMode.CHILDREN, 0.1, 20.0);
world.setClimateGraph(climateGraph);

// During chunk generation (potentially on a worker thread)
ClimateGraph graph = world.getClimateGraph();
double temperature = noise.getTemperature(x, z); // Value in [0.0, 1.0)
double intensity = noise.getIntensity(x, z);   // Value in [0.0, 1.0)

ClimateType finalClimate = graph.getType(temperature, intensity);
// Use finalClimate to determine the biome and generate terrain
```

### Anti-Patterns (Do NOT do this)
-   **Per-Chunk Instantiation:** Never create a new ClimateGraph for each chunk or query. The cost of the constructor is prohibitive and defeats the purpose of the class.
-   **Unsynchronized Refresh:** Do not call `refresh` on a live, shared instance without ensuring all world generation threads have paused their queries. This will lead to data corruption.
-   **Out-of-Bounds Coordinates:** Providing coordinate values outside the normalized range of [0.0, 1.0) is not supported. While it may not crash immediately, it will produce incorrect and unpredictable climate lookups.

## Data Pipeline

The ClimateGraph acts as a transformation pipeline, converting a declarative data structure into a procedural lookup table.

> Flow:
> 1.  **Input:** A hierarchical graph of `ClimateType` objects, loaded from configuration files.
> 2.  **Indexing:** The constructor walks the graph to build fast integer ID lookup maps (`id2TypeLookup`, `type2IdLookup`).
> 3.  **Rasterization (`populateTable`):** The system iterates over a 512x512 grid. For each grid cell, it performs a nearest-neighbor search across all `ClimatePoint`s to determine the "winning" `ClimateType`. The winner's ID is stored in the `table` IntMap.
> 4.  **Distance Transform:** The `DistanceTransform.apply` method is executed on the resulting ID table. It calculates, for each cell, the distance to the nearest cell with a different ID. This distance field is stored in the `fade` DoubleMap.
> 5.  **Runtime Query:** A world generator provides normalized (temperature, intensity) coordinates. These are scaled and used to perform an O(1) lookup into the `table` and `fade` maps, returning the final `ClimateType` and fade value.

