---
description: Architectural reference for PointGenerator
---

# PointGenerator

**Package:** com.hypixel.hytale.procedurallib.logic.point
**Type:** Transient

## Definition
```java
// Signature
public class PointGenerator implements IPointGenerator {
```

## Architecture & Concepts
The PointGenerator is a foundational component within the procedural generation library. It serves as a configurable orchestrator for calculating spatial point data, such as finding the nearest point-of-interest to a given coordinate or collecting all points within a boundary.

Architecturally, this class implements a **Strategy Pattern**. It does not contain the point generation or distance calculation logic itself. Instead, it delegates these complex operations to injected dependencies:
*   **CellDistanceFunction:** Defines the algorithm for measuring distances and searching neighboring grid cells (e.g., Euclidean, Manhattan).
*   **PointEvaluator:** Defines how a point's properties (e.g., position, value) are determined for a given cell.

This separation of concerns makes the PointGenerator highly flexible. Different combinations of CellDistanceFunction and PointEvaluator can be injected to produce a wide variety of procedural noise patterns and feature placements without altering the core orchestration logic of this class. Its primary responsibility is to manage the high-level workflow: scaling input coordinates, identifying the relevant cell region, and invoking the appropriate strategies.

## Lifecycle & Ownership
- **Creation:** An instance of PointGenerator is created directly via its public constructor. It is typically instantiated by a higher-level system, such as a biome or world generator, which assembles the required procedural logic for a specific task.
- **Scope:** The lifetime of a PointGenerator instance is generally short and tied to a specific generation operation. It is not a long-lived service. A new instance may be created to generate a single chunk's features and then be discarded.
- **Destruction:** The object is managed by the Java garbage collector and is eligible for cleanup once it falls out of scope. It holds no native resources and requires no explicit destruction method.

## Internal State & Concurrency
- **State:** The internal state of a PointGenerator instance, consisting of the seedOffset, cellDistanceFunction, and pointEvaluator, is **immutable**. These dependencies are provided at construction and marked as final, ensuring that the generator's behavior is consistent throughout its lifetime.

- **Thread Safety:** **CRITICAL WARNING:** This class is **NOT THREAD-SAFE**. Methods such as nearest2D and nearest3D obtain a reference to a shared, static, and mutable ResultBuffer. Concurrent calls to any method on *any* PointGenerator instance from different threads will cause a race condition, leading to corrupted data and unpredictable behavior. All operations involving this class must be confined to a single thread or be protected by external synchronization mechanisms.

## API Surface
The public API provides methods for querying point data in 2D and 3D space.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nearest2D(seed, x, y) | ResultBuffer2d | O(1) | Finds the single closest point to the given 2D coordinates. |
| nearest3D(seed, x, y, z) | ResultBuffer3d | O(1) | Finds the single closest point to the given 3D coordinates. |
| transition2D(seed, x, y) | ResultBuffer2d | O(1) | Finds the two closest points, used for calculating boundaries and gradients. |
| transition3D(seed, x, y, z) | ResultBuffer3d | O(1) | Finds the two closest points in 3D space. |
| collect(seed, minX, minY, maxX, maxY, consumer) | void | O(N) | Gathers all points N within a 2D bounding box, invoking the consumer for each. |

## Integration Patterns

### Standard Usage
A PointGenerator should be configured and instantiated for a specific procedural task. The caller is responsible for providing the concrete strategies that define the desired output.

```java
// 1. Define the generation strategies
CellDistanceFunction distanceFunc = new EuclideanDistanceFunction(32.0);
PointEvaluator pointEval = new WhiteNoisePointEvaluator();

// 2. Instantiate the generator for a specific task
PointGenerator noiseGenerator = new PointGenerator(12345, distanceFunc, pointEval);

// 3. Use the generator to get data for a coordinate
// WARNING: The returned buffer is a shared resource. Copy data immediately.
ResultBuffer.ResultBuffer2d result = noiseGenerator.nearest2D(worldSeed, 150.5, -45.2);
double closestPointX = result.x;
double closestPointY = result.y;
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Access:** Never call methods on a PointGenerator instance from multiple threads simultaneously. The shared internal buffer will lead to data corruption. If multithreaded generation is required, each worker thread must have its own exclusive PointGenerator instances or use external locking.
- **Result Caching:** Do not hold a reference to the returned ResultBuffer. The buffer is a shared, mutable object that will be overwritten by the next call to any query method on any PointGenerator instance. If the results are needed later, they must be copied into a separate, locally-owned data structure immediately.

## Data Pipeline
The PointGenerator processes input coordinates and transforms them into structured point data by orchestrating its dependencies. The flow for a typical query is as follows:

> Flow:
> Input Coordinates & Seed -> **PointGenerator** (scales coordinates, determines cell) -> CellDistanceFunction (searches neighbor cells) -> PointEvaluator (generates point candidates) -> **PointGenerator** (populates shared ResultBuffer) -> Return ResultBuffer to Caller

