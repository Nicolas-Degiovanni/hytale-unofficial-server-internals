---
description: Architectural reference for DistanceTransform
---

# DistanceTransform

**Package:** com.hypixel.hytale.server.worldgen.climate.util
**Type:** Utility

## Definition
```java
// Signature
public class DistanceTransform {
```

## Architecture & Concepts
The DistanceTransform class is a stateless computational utility designed for procedural world generation. Its primary function is to convert a discrete integer-based region map into a continuous, normalized distance field. This is a fundamental operation for creating smooth transitions, or falloffs, between different zones, such as biomes or climate areas.

Under the hood, the system implements a multi-source Dijkstra's algorithm. The process operates on a per-region basis:

1.  **Boundary Detection:** For each unique integer ID in the source IntMap, the algorithm first identifies all cells that lie on the boundary between that region and any other. These boundary cells become the "source nodes" for the distance calculation.
2.  **Distance Propagation:** A Dijkstra-like search is initiated from all boundary cells of a given region simultaneously. The algorithm explores outwards *into the interior* of that same region, calculating the true Euclidean distance for each internal cell to its nearest boundary. A priority queue ensures that cells closer to the boundary are processed first.
3.  **Normalization:** The calculated raw distance for each cell is then clamped by a user-provided radius and normalized to a floating-point value between 0.0 and 1.0. A value of 0.0 represents a cell directly on a boundary, while a value of 1.0 represents a cell that is at least *radius* units or more away from the nearest boundary.

The resulting DoubleMap is effectively a gradient map, ideal for downstream systems that perform blending, texture splatting, or object placement based on proximity to biome edges.

### Lifecycle & Ownership
- **Creation:** As a static utility class, DistanceTransform is never instantiated. Its methods are invoked directly on the class.
- **Scope:** The class is loaded by the JVM and persists for the entire application lifetime. All internal calculations and data structures, such as the priority queue and distance arrays, are method-scoped and exist only for the duration of a single call to the apply method.
- **Destruction:** The class is unloaded when the application terminates. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The DistanceTransform class is entirely stateless. It retains no data between invocations of its methods. All necessary state is created and destroyed within the scope of the apply method.
- **Thread Safety:** The apply method is re-entrant and safe to call from multiple threads, **provided that each thread operates on distinct source and dest map instances**. Concurrent calls operating on the same map instances will result in a race condition and produce corrupted, unpredictable output. The method contains no internal locks or synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(IntMap source, DoubleMap dest, double radius) | void | O(N log N) | Executes the distance transform. Modifies the dest map in-place. Throws IllegalArgumentException if radius is not positive. N is the number of cells in the map. |

## Integration Patterns

### Standard Usage
This utility is intended to be used as a processing step within a larger world generation pipeline. It typically follows a region-defining step (like a Voronoi diagram or cellular automata) and precedes a blending or decoration step.

```java
// Assume 'worldGenContext' provides access to world data maps
IntMap biomeRegions = worldGenContext.getMap("biome.regions");
DoubleMap biomeDistanceField = worldGenContext.createMap("biome.distanceField");

// Create a smooth falloff 24 blocks wide from every biome edge
double falloffRadius = 24.0;
DistanceTransform.apply(biomeRegions, biomeDistanceField, falloffRadius);

// The biomeDistanceField can now be used by other systems
```

### Anti-Patterns (Do NOT do this)
- **Invalid Radius:** Do not call with a radius of 0.0 or less. This is computationally meaningless and will trigger an exception.
- **In-place Modification:** Do not pass the same map object as both the source and dest parameter. The algorithm reads from the source while writing to the destination, and this will lead to undefined behavior.
- **Direct Instantiation:** Do not attempt to create an instance with `new DistanceTransform()`. It is a static utility class.

## Data Pipeline
The flow of data through this component is linear and transformative. It converts a map of discrete integer IDs into a map of continuous floating-point values representing distance.

> Flow:
> Discrete Region Map (IntMap) -> **DistanceTransform.apply** -> Normalized Distance Field (DoubleMap) -> Downstream Consumer (e.g., Biome Blender, Asset Placer)

