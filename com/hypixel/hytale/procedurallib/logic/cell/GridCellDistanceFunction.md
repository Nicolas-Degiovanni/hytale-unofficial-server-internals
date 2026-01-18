---
description: Architectural reference for GridCellDistanceFunction
---

# GridCellDistanceFunction

**Package:** com.hypixel.hytale.procedurallib.logic.cell
**Type:** Singleton (Stateless Utility)

## Definition
```java
// Signature
public class GridCellDistanceFunction implements CellDistanceFunction {
```

## Architecture & Concepts
The GridCellDistanceFunction is a foundational component within the procedural generation library, specifically serving as a core algorithm for Cellular Noise, also known as Worley or Voronoi noise. Its purpose is to define and execute a search pattern for finding the closest "feature points" to a given coordinate in 2D or 3D space.

This class implements a standard and highly efficient grid-based neighborhood search. For any given input coordinate, it correctly assumes that the nearest feature point must lie within the coordinate's own grid cell or one of its immediate neighbors. Consequently, it iterates over a fixed 3x3 (for 2D) or 3x3x3 (for 3D) kernel of cells. This bounded search domain makes the core operation have a constant time complexity, which is critical for performance in large-scale world generation.

Architecturally, this class is a stateless implementation of the **Strategy Pattern**. It provides the generic algorithm for *how* to find and process neighboring cell points but delegates the specific calculation or action—the *what*—to a `PointEvaluator` object provided by the caller. This decouples the grid traversal logic from the specific noise function being computed (e.g., distance to the nearest point F1, distance to the second-nearest point F2, or returning a cell's unique color).

### Lifecycle & Ownership
- **Creation:** A public static final instance, `DISTANCE_FUNCTION`, is instantiated by the JVM at class-loading time. The class is not intended for direct instantiation by consumers.
- **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the application.
- **Destruction:** The object is managed by the JVM and is garbage collected during class unloading when the application terminates. No manual resource management is required.

## Internal State & Concurrency
- **State:** This class is **completely stateless and immutable**. It possesses no instance fields. All its methods are pure functions whose outputs depend exclusively on their input arguments. It relies on external static lookup tables like `CellularNoise.CELL_2D`, which are themselves immutable.

- **Thread Safety:** This class is **unconditionally thread-safe**. Due to its stateless nature, the singleton instance can be safely shared and invoked by multiple threads concurrently without any need for external synchronization or locks.

    **WARNING:** While this class is thread-safe, the `PointEvaluator` and `ResultBuffer` objects passed into its methods may not be. It is the caller's responsibility to ensure that any stateful objects passed as arguments are handled in a thread-safe manner when used in a concurrent environment.

## API Surface
The public API is designed for executing bounded searches for cellular noise calculations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nearest2D(...) | void | O(1) | Evaluates the 9 cells in a 3x3 grid. Delegates each point to the PointEvaluator, which updates the ResultBuffer. |
| nearest3D(...) | void | O(1) | Evaluates the 27 cells in a 3x3x3 grid. Delegates each point to the PointEvaluator, which updates the ResultBuffer. |
| collect(...) | void | O(N*M) | Iterates over a rectangular region of cells. Delegates each point to a PointConsumer for aggregation. |
| getHash(seed, cellX, cellY) | int | O(1) | Static utility to compute the deterministic hash for a given 2D cell coordinate and seed. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve the static singleton instance and invoke its methods with a specific `PointEvaluator` strategy to populate a result buffer. This is typically orchestrated by a higher-level noise generator.

```java
// Obtain the singleton instance
GridCellDistanceFunction func = GridCellDistanceFunction.DISTANCE_FUNCTION;

// A buffer to store the results of the evaluation
ResultBuffer.ResultBuffer2d buffer = new ResultBuffer.ResultBuffer2d();

// A strategy that defines what to do with each point (e.g., find closest)
PointEvaluator evaluator = new ClosestPointEvaluator();

// Define the world coordinate and the integer cell it belongs to
double x = 42.5;
double y = 18.2;
int cellX = 42;
int cellY = 18;
int seed = 12345;

// Execute the search. The buffer will be mutated with the results.
func.nearest2D(seed, x, y, cellX, cellY, buffer, evaluator);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new GridCellDistanceFunction()`. This creates a redundant object and violates the singleton design pattern. Always use the static `GridCellDistanceFunction.DISTANCE_FUNCTION` instance.
- **Passing Stateful Evaluators in Concurrent Code:** Do not pass a `PointEvaluator` instance that is not thread-safe into this class's methods from multiple threads. While the `GridCellDistanceFunction` itself is safe, it cannot enforce safety on the strategy objects it is given. This can lead to race conditions within the evaluator's logic or the result buffer.

## Data Pipeline
The flow of data for a `nearest2D` call is a multi-stage process that transforms spatial coordinates into a final evaluated result. The `GridCellDistanceFunction` orchestrates this pipeline, but the final step is controlled by the injected `PointEvaluator`.

> Flow:
> Input Coordinate & Seed -> **GridCellDistanceFunction.nearest2D** -> Iterate 3x3 Cell Grid -> For each cell, call `getHash` -> Use Hash to look up Vector from `CellularNoise.CELL_2D` -> Apply `CellJitter` from Evaluator -> **Delegate to PointEvaluator** -> PointEvaluator updates external `ResultBuffer`

