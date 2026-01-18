---
description: Architectural reference for EuclideanDistanceFunction
---

# EuclideanDistanceFunction

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.distancefunctions
**Type:** Utility / Strategy

## Definition
```java
// Signature
public class EuclideanDistanceFunction extends DistanceFunction {
```

## Architecture & Concepts
The EuclideanDistanceFunction is a concrete implementation of the DistanceFunction strategy. It serves a highly specific and performance-critical role within the procedural world generation pipeline, particularly in density and noise field calculations.

Its primary function is to provide a fast, low-overhead calculation of a point's distance from the origin. Architecturally, it is a leaf-node component in a larger composition of generator nodes, where it is invoked to determine falloff, shape, or other distance-based properties of generated features.

**Warning:** This class calculates the *squared* Euclidean distance (x² + y² + z²), not the true geometric distance. This is a deliberate and common performance optimization in procedural generation to avoid the computationally expensive square root operation. This is sufficient for any algorithm where only the relative comparison of distances is required. The caller is responsible for performing the square root if the true distance is needed.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly by higher-level generator nodes or configuration systems that require this specific distance metric. It is not managed by a central service locator or dependency injection framework.
-   **Scope:** Transient and short-lived. An instance is typically created, used for a series of calculations within a single generation task (e.g., for one chunk), and then becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. No explicit cleanup or resource management is necessary due to its stateless nature.

## Internal State & Concurrency
-   **State:** This object is **stateless** and **immutable**. It contains no member fields, and its behavior is purely a function of its input arguments.
-   **Thread Safety:** Inherently thread-safe. A single instance can be safely shared and invoked from multiple world generation threads simultaneously without any risk of race conditions or data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDistance(Vector3d point) | double | O(1) | Calculates the squared Euclidean distance from the origin. Throws NullPointerException if the input vector is null. |

## Integration Patterns

### Standard Usage
This function is typically instantiated and used within the evaluation logic of a more complex procedural generation node.

```java
// Within a procedural generation node's evaluation logic
DistanceFunction distFunc = new EuclideanDistanceFunction();
Vector3d samplePoint = new Vector3d(10.5, -20.0, 5.2);

// The squared distance is used directly in a density formula
// to avoid a costly square root operation.
double densityModifier = 1.0 / (1.0 + distFunc.getDistance(samplePoint));
```

### Anti-Patterns (Do NOT do this)
-   **Misinterpreting the Result:** Do not use the return value as if it were the true geometric distance. This will lead to incorrect shapes and generation artifacts. If the actual distance is required, the caller must perform the square root.
    ```java
    // INCORRECT: Assumes the result is the true distance
    // double radius = 50.0;
    // if (distFunc.getDistance(point) < radius) { ... }

    // CORRECT: Compare against the squared radius
    double radiusSquared = 50.0 * 50.0;
    if (distFunc.getDistance(point) < radiusSquared) { ... }
    ```
-   **Unnecessary Instantiation:** Avoid creating a new instance inside a tight loop (e.g., per-voxel). Since the object is stateless, instantiate it once per task and reuse it.

## Data Pipeline
This component acts as a pure mathematical function, transforming a position vector into a scalar value.

> Flow:
> Vector3d Coordinate -> **EuclideanDistanceFunction** -> double (Squared Distance)

