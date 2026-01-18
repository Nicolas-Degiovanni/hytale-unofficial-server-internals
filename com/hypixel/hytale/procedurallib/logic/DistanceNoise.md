---
description: Architectural reference for DistanceNoise
---

# DistanceNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class DistanceNoise implements NoiseFunction {
```

## Architecture & Concepts

DistanceNoise is an abstract base class that serves as a foundational component for generating cellular noise, commonly known as Worley or Voronoi noise. It is a specialized implementation of the NoiseFunction interface, designed to produce patterns based on the distance from a sample point to a grid of feature points.

Architecturally, this class acts as a template that orchestrates three distinct, pluggable strategies to define its final behavior. This heavy use of the **Strategy Pattern** via constructor injection makes it highly configurable for producing a vast range of procedural textures and patterns.

The three core strategies are:
1.  **CellDistanceFunction:** Defines the shape and properties of the grid (e.g., square, hexagonal) and the metric for measuring distance (e.g., Euclidean, Manhattan). It is responsible for identifying which cells to check around a given sample point.
2.  **PointEvaluator:** Determines the location of one or more feature points within each grid cell. This can range from a simple pseudo-random point to more complex arrangements.
3.  **Distance2Function:** A functional interface that defines how the distances to the two nearest feature points (F1 and F2) are combined. This final combination step is what creates the characteristic patterns, such as solid cells (F1), cell borders (F2 - F1), or web-like structures (F1 + F2).

The class is abstract, deferring the memory management of its internal calculation buffers to concrete subclasses. This is a critical design choice for performance, allowing implementations to use techniques like thread-local storage to avoid heap allocations and locking in performance-sensitive, multi-threaded world generation contexts.

## Lifecycle & Ownership

-   **Creation:** A concrete subclass of DistanceNoise is instantiated by a higher-level system, typically a noise graph builder or a procedural generation controller. It is never instantiated directly via `new DistanceNoise()`. Its dependencies must be provided at construction time.
-   **Scope:** An instance of DistanceNoise is typically long-lived. It is configured once as part of a larger "noise recipe" (e.g., for a specific biome) and persists for the duration of the generation task, serving millions of calls to its `get` method.
-   **Destruction:** The object has no explicit destruction or cleanup logic. It becomes eligible for garbage collection when the procedural generation task that created it is complete and all references to it are released.

## Internal State & Concurrency

-   **State:** The internal state of a DistanceNoise object is defined by its three strategy fields: `cellDistanceFunction`, `pointEvaluator`, and `distance2Function`. These fields are `final` and are set only once during construction, making the object's configuration **immutable**. The class itself does not maintain any mutable state between calls.
-   **Thread Safety:** The class is designed to be **conditionally thread-safe**. Because its configuration is immutable, the primary concurrency concern is the management of the `ResultBuffer` used in calculations. The abstract methods `localBuffer2d` and `localBuffer3d` delegate this responsibility to subclasses. A correct implementation will use a mechanism like `ThreadLocal` to provide a unique buffer for each thread, making the `get` methods safe for concurrent invocation without locks.

**WARNING:** A naive subclass implementation that returns a shared, non-thread-safe buffer will introduce severe race conditions and data corruption in a multi-threaded environment.

## API Surface

The public contract is focused on fulfilling the `NoiseFunction` interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, offset, x, y) | double | O(C) | Calculates the 2D cellular noise value at the given coordinates. Complexity is constant relative to the number of cells checked. |
| get(seed, offset, x, y, z) | double | O(C) | Calculates the 3D cellular noise value at the given coordinates. Complexity is constant relative to the number of cells checked. |

## Integration Patterns

### Standard Usage

A developer should never use this abstract class directly. Instead, they should instantiate a concrete implementation and provide the required strategies. The resulting `NoiseFunction` is then used by the procedural generation engine.

```java
// Example of configuring a concrete subclass (assumed to exist)
// for a classic "crackle" pattern.

// 1. Define the strategies
CellDistanceFunction grid = new SquareCellDistanceFunction(1.0, DistanceMetric.EUCLIDEAN);
PointEvaluator points = new SingleRandomPointEvaluator();
Distance2Function combiner = DistanceNoise.Distance2Mode.SUB.getFunction();

// 2. Instantiate a concrete, thread-safe implementation
NoiseFunction worleyNoise = new ThreadLocalDistanceNoise(grid, points, combiner);

// 3. Use the function during world generation
double value = worleyNoise.get(worldSeed, 0, 123.4, 567.8);
```

### Anti-Patterns (Do NOT do this)

-   **Frequent Re-instantiation:** Do not create new instances of a DistanceNoise implementation inside a generation loop. This is extremely inefficient. Configure it once and reuse the instance.
-   **Incorrect Buffer Management:** When creating a subclass, do not return a shared static buffer from `localBuffer2d` or `localBuffer3d`. This will break thread safety. Always use a thread-local or a new instance if performance is not a concern.
-   **Direct Instantiation:** Do not attempt to call `new DistanceNoise()`. The class is abstract and cannot be instantiated.

## Data Pipeline

The flow of data for a single call to the `get` method is a multi-stage process orchestrated by the DistanceNoise instance.

> Flow:
> (x, y, z) Coordinates & Seed -> `CellDistanceFunction` (Scales coordinates, finds grid cell) -> Iterates over neighbor cells -> `PointEvaluator` (Finds feature point in each cell) -> Calculates distances -> Stores two smallest distances in `ResultBuffer` -> `Distance2Function` (Combines the two distances) -> Final value is normalized to [-1, 1] -> `double` Noise Value

