---
description: Architectural reference for HexCellDistanceFunction
---

# HexCellDistanceFunction

**Package:** com.hypixel.hytale.procedurallib.logic.cell
**Type:** Singleton

## Definition
```java
// Signature
public class HexCellDistanceFunction implements CellDistanceFunction {
```

## Architecture & Concepts
The HexCellDistanceFunction is a specialized, stateless implementation of the CellDistanceFunction interface, designed to provide the geometric calculations for 2D hexagonal cellular noise. It serves as a core mathematical component within the procedural generation library.

Its primary architectural role is to act as a **Strategy** in a Strategy Pattern. The main CellularNoise generator is configured with a distance function to determine the grid structure and distance metric. By providing this implementation, the generator can produce noise patterns based on a hexagonal tiling instead of a standard square grid.

This class encapsulates all the complex logic for transforming between Cartesian (world) coordinates and the axial coordinates of a hexagonal grid. It is responsible for:
1.  Identifying which hexagonal cell a given world point belongs to.
2.  Calculating the positions of pseudo-random feature points within each hexagonal cell.
3.  Iterating over the correct neighborhood of cells to find the nearest feature points to a given world point.

**WARNING:** This implementation is strictly for 2D calculations. All 3D methods will throw an UnsupportedOperationException at runtime.

## Lifecycle & Ownership
-   **Creation:** A public static final instance, DISTANCE_FUNCTION, is instantiated by the JVM during class loading. This is the canonical instance and should be used exclusively.
-   **Scope:** As a static singleton, it persists for the entire lifetime of the application.
-   **Destruction:** The object is eligible for garbage collection only when the application's ClassLoader is unloaded, which typically occurs at shutdown.

## Internal State & Concurrency
-   **State:** The HexCellDistanceFunction is **completely stateless and immutable**. All of its fields are static final constants, representing pre-calculated mathematical values for coordinate transformations or lookup tables. It holds no per-instance or per-request state.
-   **Thread Safety:** This class is inherently **thread-safe**. Its methods can be called concurrently from any number of threads without requiring external synchronization. All calculations depend only on the arguments passed to the methods.

## API Surface
The public API provides the necessary primitives for a noise generator to compute hexagonal cellular noise.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nearest2D(...) | void | O(1) | Finds the closest feature point to a given coordinate. It evaluates a fixed 3x3 neighborhood of cells. The result is written into the provided ResultBuffer. |
| transition2D(...) | void | O(1) | Finds the two closest feature points to a given coordinate, used for generating cell boundaries. Evaluates a fixed, slightly larger neighborhood. |
| collect(...) | void | O(N) | Gathers all feature points within a specified rectangular bounds. Complexity is proportional to the number of cells in the query area. |
| getCellX(x, y) | int | O(1) | Converts a world-space coordinate into a hexagonal grid cell X-coordinate. |
| getCellY(x, y) | int | O(1) | Converts a world-space coordinate into a hexagonal grid cell Y-coordinate. |
| scale(value) | double | O(1) | Applies a normalization scale factor to a distance value, specific to hexagonal grids. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation. It should be used as a singleton strategy provided to a higher-level noise generator. The client code interacts with the generator, which in turn uses this distance function under the hood.

```java
// Hypothetical usage with a CellularNoise generator
CellularNoise noiseGenerator = new CellularNoise.Builder()
    .setSeed(12345)
    .setDistanceFunction(HexCellDistanceFunction.DISTANCE_FUNCTION)
    .setPointEvaluator(new SomePointEvaluator())
    .build();

// The generator now uses the hexagonal logic internally
double noiseValue = noiseGenerator.getValue(x, y);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new HexCellDistanceFunction()`. This creates unnecessary objects and bypasses the intended singleton pattern. Always use the static `HexCellDistanceFunction.DISTANCE_FUNCTION` instance.
-   **Calling 3D Methods:** Invoking `nearest3D`, `transition3D`, or other 3D-suffixed methods will result in an `UnsupportedOperationException` and crash the calling thread. This implementation is strictly 2D.
-   **Misinterpreting Coordinates:** The methods expect world-space coordinates. Do not pre-convert coordinates to a grid system before passing them to methods like `nearest2D`. The function's purpose is to perform that conversion itself.

## Data Pipeline
The flow of data for a single noise evaluation query is a direct, synchronous process orchestrated by a consumer like a CellularNoise generator.

> Flow:
> World Coordinate (x, y) -> **HexCellDistanceFunction.getCellX/Y** -> Central Cell Coordinate -> **HexCellDistanceFunction.nearest2D** -> Iterates 9 Neighbor Cells -> **HexCellDistanceFunction.evalPoint** -> PointEvaluator -> Distance Calculation -> ResultBuffer Update

