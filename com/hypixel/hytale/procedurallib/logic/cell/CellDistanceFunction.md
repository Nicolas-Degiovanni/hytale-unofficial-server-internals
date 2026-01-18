---
description: Architectural reference for CellDistanceFunction
---

# CellDistanceFunction

**Package:** com.hypixel.hytale.procedurallib.logic.cell
**Type:** Interface (Strategy Pattern)

## Definition
```java
// Signature
public interface CellDistanceFunction {
```

## Architecture & Concepts
The CellDistanceFunction interface defines a formal contract for spatial query algorithms operating on a grid of cells. It is a core component of the procedural generation library, serving as a strategy interface for various forms of cellular noise, most notably Worley noise and its derivatives.

This abstraction decouples the high-level procedural generator from the specific mathematical implementation of distance calculation. By providing different implementations of this interface (e.g., Euclidean, Manhattan, Chebyshev distance), the engine can produce vastly different procedural patterns—such as organic cellular structures, rigid crystalline formations, or city-like grids—without altering the orchestration logic.

Its primary role is to answer questions about points within a cellular grid, such as: "What is the distance to the nearest feature point?" or "What are the properties of the two closest feature points?". These queries are fundamental for generating biome boundaries, placing points of interest, and creating complex texture patterns. The interface operates in both 2D and 3D contexts, with a parallel set of methods for each dimensionality.

## Lifecycle & Ownership
- **Creation:** As an interface, CellDistanceFunction is not instantiated directly. Concrete implementations are typically stateless, singleton-like objects or lightweight transient objects created by a factory or procedural context builder.
- **Scope:** The lifetime of an implementing object is typically tied to a specific generation task. It is passed as a dependency into noise evaluators and exists only for the duration of that evaluation.
- **Destruction:** Implementations are expected to hold no resources and are garbage collected once the generation task that created them is complete.

## Internal State & Concurrency
- **State:** This interface is designed for stateless implementations. It defines pure functions whose outputs depend solely on their inputs (coordinates, seed, etc.). Any caching or memoization must be handled by the calling system, not within the function itself.
- **Thread Safety:** All implementations of CellDistanceFunction **must be unconditionally thread-safe**. The procedural generation engine heavily relies on parallel execution to generate world chunks concurrently. A stateful or unsynchronized implementation will cause severe race conditions, leading to non-deterministic and visually corrupt world generation.

## API Surface
The API is designed for high-performance, tight-loop execution. It uses a `ResultBuffer` as an output parameter to avoid heap allocations during generation. Arguments are intentionally primitive to minimize overhead.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scale(value) | double | O(1) | Applies a forward transformation to a distance value. |
| invScale(value) | double | O(1) | Applies a reverse transformation to a scaled distance value. |
| getCellX/Y/Z(coords) | int | O(1) | Calculates the integer grid coordinate for a given world coordinate. |
| nearest2D / nearest3D | void | O(k) | Finds the nearest feature point(s) to a given coordinate. The complexity depends on the number of neighboring cells searched. |
| transition2D / transition3D | void | O(k) | Calculates distance values at the boundary between two cells. Crucial for generating smooth biome edges and Voronoi ridges. |
| evalPoint / evalPoint2 | void | O(k) | Evaluates the distance function at a specific coordinate, storing results in the buffer. The distinction often relates to returning the Nth-closest point (e.g., F1 vs F2 noise). |
| collect | void | O(n) | Gathers all feature points within a given bounding box of cells. Complexity depends on the number of points found. |

## Integration Patterns

### Standard Usage
A CellDistanceFunction implementation is typically retrieved from a central registry or factory and passed into a higher-level noise generation service. The service then invokes its methods for a large set of coordinates, populating a buffer with the results.

```java
// A procedural generator obtains a specific distance function strategy.
CellDistanceFunction euclideanDist = ProceduralContext.getDistanceFunction("Euclidean");
ResultBuffer.ResultBuffer2d results = new ResultBuffer.ResultBuffer2d();
PointEvaluator jitteredEvaluator = ProceduralContext.getPointEvaluator("JitteredGrid");
int seed = 12345;

// The generator iterates over a region, invoking the function for each point.
for (int x = 0; x < 16; x++) {
    for (int y = 0; y < 16; y++) {
        double worldX = 100.0 + x;
        double worldY = 250.0 + y;
        int cellX = euclideanDist.getCellX(worldX, worldY);
        int cellY = euclideanDist.getCellY(worldX, worldY);

        // The function populates the result buffer directly.
        euclideanDist.nearest2D(seed, worldX, worldY, cellX, cellY, results, jitteredEvaluator);

        // Use the populated results...
        float noiseValue = results.getF1();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Creating an implementation that stores state between calls is a critical error. This will break concurrent generation and produce unpredictable, inconsistent results. All necessary state must be passed in via method arguments (seed, coordinates).
- **Ignoring the PointEvaluator:** The PointEvaluator collaborator is essential. It determines where feature points are located within each cell. Passing a null or incorrect evaluator will lead to incorrect or empty results.
- **Misusing ResultBuffer:** The ResultBuffer is reused for performance. Failing to clear or reset it between distinct logical evaluations can cause data from a previous calculation to leak into the next one.

## Data Pipeline
The CellDistanceFunction is a processing stage in the procedural noise pipeline. It transforms spatial coordinates into abstract distance metrics.

> Flow:
> World Coordinates & Seed -> **CellDistanceFunction** (with PointEvaluator) -> Populated ResultBuffer -> Noise Value Aggregator -> Final Voxel/Biome Data

