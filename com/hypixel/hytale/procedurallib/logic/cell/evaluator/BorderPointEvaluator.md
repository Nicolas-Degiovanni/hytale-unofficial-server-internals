---
description: Architectural reference for BorderPointEvaluator
---

# BorderPointEvaluator

**Package:** com.hypixel.hytale.procedurallib.logic.cell.evaluator
**Type:** Singleton

## Definition
```java
// Signature
public class BorderPointEvaluator implements PointEvaluator {
```

## Architecture & Concepts

The BorderPointEvaluator is a specialized, stateless algorithm that implements the PointEvaluator strategy interface. It is a core component within the procedural generation library, specifically designed for cellular noise functions that produce Worley-like patterns.

Unlike standard point evaluators which calculate the distance from a sample point to a cell's feature point, the BorderPointEvaluator calculates the distance to the **perpendicular bisector** between a cell's feature point and the origin cell's feature point. This bisector represents the theoretical border or edge in a Voronoi diagram.

By finding the minimum distance to these borders instead of the cell centers, this evaluator enables the generation of noise patterns resembling crystalline structures, cracked mud, or other cellular boundary-based textures. It is fundamentally a 2D-only algorithm; 3D operations are explicitly not supported and will fail at runtime.

## Lifecycle & Ownership

-   **Creation:** A single, public static final instance named INSTANCE is instantiated eagerly by the JVM during class loading. This is a canonical implementation of the Singleton pattern.
-   **Scope:** Application-scoped. The single INSTANCE persists for the entire lifetime of the application once the class has been loaded.
-   **Destruction:** The object is managed by the JVM and is destroyed only upon application shutdown. No manual cleanup is required or possible.

## Internal State & Concurrency

-   **State:** The BorderPointEvaluator is **completely stateless**. It contains no instance fields and all its computational logic relies exclusively on the arguments passed into its methods. The stateful data it operates on, the ResultBuffer, is owned and managed by the caller.
-   **Thread Safety:** This class is inherently **thread-safe**. The public INSTANCE can be shared and invoked by any number of threads concurrently without risk of race conditions or data corruption.

    **WARNING:** While the evaluator itself is thread-safe, the caller is responsible for ensuring that each thread operates on a unique, non-shared instance of ResultBuffer to prevent data races within the buffer object.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| evalPoint(..., ResultBuffer2d) | void | O(1) | Calculates the distance to the cell border and updates the buffer's primary distance if a new minimum is found. |
| evalPoint2(..., ResultBuffer2d) | void | O(1) | Calculates distance to the cell border and updates the buffer's primary and secondary distances to track the two closest borders. |
| evalPoint(..., ResultBuffer3d) | void | N/A | **Unsupported.** Throws UnsupportedOperationException. This evaluator is strictly for 2D contexts. |
| evalPoint2(..., ResultBuffer3d) | void | N/A | **Unsupported.** Throws UnsupportedOperationException. This evaluator is strictly for 2D contexts. |

## Integration Patterns

### Standard Usage

The BorderPointEvaluator is not intended to be used directly. It should be supplied as a strategy to a higher-level cellular noise generator which will manage the iteration and buffer lifecycle. Always use the provided static INSTANCE.

```java
// A hypothetical noise generator receives the evaluator strategy
CellularNoiseGenerator2D noiseGenerator = new CellularNoiseGenerator2D(seed);
ResultBuffer.ResultBuffer2d result = new ResultBuffer.ResultBuffer2d();

// The generator uses the supplied evaluator internally to calculate the final noise value
float noiseValue = noiseGenerator.getValue(x, y, BorderPointEvaluator.INSTANCE, result);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new BorderPointEvaluator()`. This defeats the purpose of the singleton pattern, creating unnecessary garbage and bypassing any potential future optimizations on the static instance. Always use `BorderPointEvaluator.INSTANCE`.
-   **3D Evaluation:** Do not attempt to use this evaluator in a 3D context. The 3D method overloads are not implemented and will throw an UnsupportedOperationException, causing a runtime crash.
-   **Misinterpreting Output:** Do not assume the distance stored in the ResultBuffer is the distance to a cell's feature point. It is the distance to the nearest *border* between cells. This is a critical conceptual difference.

## Data Pipeline

The BorderPointEvaluator functions as a single, critical step within a larger cellular noise calculation pipeline. It does not own data but rather transforms it in-place.

> Flow:
> Cellular Noise Algorithm -> Iterates over a grid of neighbor cells -> For each cell, invokes **BorderPointEvaluator.evalPoint** -> The evaluator calculates the distance to the Voronoi edge -> The mutable ResultBuffer is updated with the new minimum distance -> The final value in ResultBuffer is used to determine the output noise value.

