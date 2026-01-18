---
description: Architectural reference for DistancePointEvaluator
---

# DistancePointEvaluator

**Package:** com.hypixel.hytale.procedurallib.logic.cell.evaluator
**Type:** Transient Strategy

## Definition
```java
// Signature
public class DistancePointEvaluator implements PointEvaluator {
```

## Architecture & Concepts
The DistancePointEvaluator is a fundamental component within the procedural generation library, specifically designed for cellular noise algorithms (e.g., Worley noise, Voronoi diagrams). Its role is to compute a scalar value, typically distance, between a world-space sample point and a specific cell's feature point.

This class is a concrete implementation of the **Strategy Pattern**. It is intentionally designed to be stateless and configurable, decoupling the core cellular evaluation loop from the specific mathematics of distance calculation and result modification. At construction, it is injected with two key strategies:

1.  **PointDistanceFunction:** A strategy defining the distance metric. This allows the same evaluation framework to produce visually distinct results by swapping out the function (e.g., Euclidean for standard circular cells, Manhattan for square-like cells).
2.  **ISeedDoubleRange:** A strategy for transforming the raw distance value. This is used to introduce controlled variation, such as scaling, clamping, or applying deterministic randomness to the distance, which can break up the uniformity of the resulting noise pattern.

The evaluator's primary function is to be invoked repeatedly by a higher-level generator that iterates over a grid of cells surrounding a sample point. For each cell, the evaluator calculates the distance to its feature point, modifies it, and registers the result in a shared ResultBuffer.

## Lifecycle & Ownership
-   **Creation:** An instance is created by a parent procedural generator or noise graph during its initialization and configuration phase. It is not managed by a dependency injection container and is constructed directly with its required strategy objects.
-   **Scope:** The object's lifetime is bound to the procedural generation task for which it was configured. It is typically a short-lived, transient object used for a single noise generation operation and then discarded.
-   **Destruction:** The object holds no native resources and requires no explicit cleanup. It becomes eligible for garbage collection as soon as the reference held by its parent generator is released.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal fields, `distanceFunction` and `distanceMod`, are `final` and are assigned exclusively during construction. The class holds no mutable state, and each method call is a pure function of its inputs.
-   **Thread Safety:** **Thread-safe**. Due to its immutable and stateless design, a single DistancePointEvaluator instance can be safely invoked by multiple threads concurrently without any risk of race conditions or data corruption. This makes it highly suitable for parallelized procedural generation tasks.

## API Surface
The public API consists of four overloaded evaluation methods. The `evalPoint` and `evalPoint2` variants target the first and second result slots in the ResultBuffer, respectively. This is a common pattern in Worley noise for calculating F1 and F2 values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| evalPoint(seed, x, y, ...) | void | O(1) | Calculates 2D distance, modifies it, and registers the result in the primary slot of the ResultBuffer. |
| evalPoint2(seed, x, y, ...) | void | O(1) | Calculates 2D distance, modifies it, and registers the result in the secondary slot of the ResultBuffer. |
| evalPoint(seed, x, y, z, ...) | void | O(1) | Calculates 3D distance, modifies it, and registers the result in the primary slot of the ResultBuffer. |
| evalPoint2(seed, x, y, z, ...) | void | O(1) | Calculates 3D distance, modifies it, and registers the result in the secondary slot of the ResultBuffer. |

## Integration Patterns

### Standard Usage
The DistancePointEvaluator is configured once as part of a larger algorithm's definition and then passed into the core processing loop.

```java
// 1. Define the distance calculation strategy (e.g., Euclidean distance)
PointDistanceFunction distanceMetric = new EuclideanDistanceFunction();

// 2. Define an optional distance modification strategy
IDoubleRange distanceScalar = new DoubleRange(0.8, 1.2);

// 3. Construct the evaluator with its strategies
PointEvaluator evaluator = new DistancePointEvaluator(distanceMetric, distanceScalar);

// 4. A procedural engine uses the configured evaluator to process points.
//    The ResultBuffer collects the two closest points for F1/F2 noise.
ResultBuffer.ResultBuffer2d buffer = new ResultBuffer.ResultBuffer2d();
engine.processNeighboringCells(sampleX, sampleY, evaluator, buffer);
double f1 = buffer.d1;
double f2 = buffer.d2;
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation Within a Loop:** Never create a new DistancePointEvaluator inside a per-pixel or per-voxel loop. This is highly inefficient and defeats the purpose of its reusable, stateless design. Configure it once before the loop begins.
-   **Stateful Strategies:** Do not inject implementations of PointDistanceFunction or ISeedDoubleRange that contain mutable state if the evaluator is to be used across multiple threads. This would break the thread-safety guarantee of the DistancePointEvaluator itself.

## Data Pipeline
The class acts as a specific processing stage in a larger data flow. It consumes spatial coordinates and outputs a modified distance value into a buffer.

> Flow:
> (Sample Point, Cell Feature Point) -> **PointDistanceFunction** -> Raw Distance -> **ISeedDoubleRange** -> Modified Distance -> **DistancePointEvaluator** -> ResultBuffer.register()

