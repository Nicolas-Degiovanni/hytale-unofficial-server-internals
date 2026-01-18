---
description: Architectural reference for CellNoise
---

# CellNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class CellNoise implements NoiseFunction {
```

## Architecture & Concepts
The CellNoise class is a fundamental component of the procedural generation library, providing an implementation for cellular noise, often known as Worley or Voronoi noise. It conforms to the NoiseFunction interface, allowing it to be used interchangeably with other noise algorithms within a larger noise graph.

Its core architectural purpose is to partition N-dimensional space into a grid of cells. For any given input coordinate, it identifies the nearest "feature point" within a cell and computes a value based on a configurable rule. This makes it ideal for generating organic, crystal-like, or cracked patterns.

The design of CellNoise is heavily based on the **Strategy Pattern** and **Dependency Injection**. The main class is a generic orchestrator that handles the high-level logic of grid traversal and nearest-point searching. The specific behavior is delegated to injected dependencies:

-   **CellDistanceFunction:** Defines the shape of the grid and the metric for measuring distance (e.g., Euclidean, Manhattan).
-   **PointEvaluator:** Determines how many feature points exist within each cell and where they are located.
-   **CellFunction:** The final step in the pipeline, this strategy defines *what* value is computed based on the nearest feature point. For example, it could return the distance to the point, a random value associated with the cell, or a value from another noise function sampled at the point's location.

The nested CellMode enum acts as a factory, providing several common and highly optimized CellFunction implementations. This combination of a generic container and swappable strategies makes CellNoise an extremely versatile and composable system for world generation.

## Lifecycle & Ownership
-   **Creation:** CellNoise instances are not managed by a central service locator. They are constructed directly by a higher-level system, typically a noise graph builder or a world generation orchestrator. All dependencies and strategies must be provided at construction time.
-   **Scope:** An instance of CellNoise represents a complete, configured noise algorithm. Its lifetime is tied to the configuration that created it. It is expected to be created once during the initialization of a world generator and reused for all subsequent noise evaluations. It holds no per-request state and is safe to reuse indefinitely.
-   **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods. It is reclaimed when the noise graph or generator that holds a reference to it is itself reclaimed.

## Internal State & Concurrency
-   **State:** The CellNoise object is **effectively immutable**. All of its dependent strategies are stored in final fields and are set only once via the constructor. The object itself does not mutate its state during a call to the get method.

-   **Thread Safety:** **CRITICAL WARNING:** This class is designed for high-performance, multi-threaded environments, but its thread safety depends entirely on the implementation of the shared `ResultBuffer`. The methods `localBuffer2d` and `localBuffer3d` retrieve a buffer object to avoid per-call heap allocations. If this buffer is implemented as a `ThreadLocal`, the class is fully thread-safe. If it is a simple static field, the class is **NOT THREAD-SAFE**, and concurrent calls to `get` will produce catastrophic race conditions and incorrect noise values. All engine-level usage must assume a ThreadLocal buffer is provided.

## API Surface
The public contract is defined by the NoiseFunction interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, offsetSeed, x, y) | double | O(1) | Computes the 2D cellular noise value. Complexity is constant time, as the neighbor search is bounded. |
| get(seed, offsetSeed, x, y, z) | double | O(1) | Computes the 3D cellular noise value. Complexity is constant time, as the neighbor search is bounded. |

## Integration Patterns

### Standard Usage
A CellNoise instance should be configured once and stored for reuse. The CellMode enum provides the most common configurations for the CellFunction strategy.

```java
// Example: Create a noise function that returns the distance to the nearest cell center.
CellDistanceFunction distFunc = new StandardCellDistanceFunction(0.1); // Scale the grid
PointEvaluator pointEval = new SinglePointEvaluator(); // One point at the center of each cell

// Construct the CellNoise object using a pre-defined mode
NoiseFunction worleyNoise = new CellNoise(
    distFunc,
    pointEval,
    CellMode.DISTANCE.getFunction(),
    null // noiseLookup is not needed for DISTANCE mode
);

// Reuse the instance to evaluate multiple points
double value = worleyNoise.get(worldSeed, 0, 123.4, 567.8);
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in a Loop:** Never create a new CellNoise instance inside a tight loop (e.g., for each block in a chunk). This is highly inefficient and defeats the purpose of its reusable design.
-   **Incorrect CellFunction for Mode:** Providing a null NoiseProperty dependency when using the NOISE_LOOKUP CellMode will result in a NullPointerException at runtime. The dependencies must match the selected strategy.
-   **Ignoring Thread Safety:** Do not share a single CellNoise instance across threads unless you can guarantee the underlying ResultBuffer implementation is a ThreadLocal. Assuming it is thread-safe without verification is dangerous.

## Data Pipeline
The flow of data for a single noise evaluation is a multi-stage process orchestrated by the `get` method.

> Flow:
> 1. Input Coordinates (x, y, z) & Seed
> 2. -> **CellDistanceFunction**: Scales coordinates and determines the integer grid cell.
> 3. -> **Internal Orchestrator**: Acquires a thread-safe ResultBuffer to store intermediate calculations.
> 4. -> **CellDistanceFunction.nearest**: Searches the current and neighboring cells for the closest feature point(s), using the **PointEvaluator** to find points within each cell. The findings are written to the ResultBuffer.
> 5. -> **CellFunction.eval**: The chosen strategy is executed. It reads from the populated ResultBuffer to compute a final raw value (e.g., distance, hash, etc.).
> 6. -> **GeneralNoise.limit**: The raw value is clamped and normalized to the standard engine range of [-1.0, 1.0].
> 7. -> Output: double

---

