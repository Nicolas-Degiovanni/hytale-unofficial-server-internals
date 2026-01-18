---
description: Architectural reference for NormalPointEvaluator
---

# NormalPointEvaluator

**Package:** com.hypixel.hytale.procedurallib.logic.cell.evaluator
**Type:** Utility / Strategy

## Definition
```java
// Signature
public class NormalPointEvaluator implements PointEvaluator {
```

## Architecture & Concepts
The NormalPointEvaluator is a fundamental component within the procedural generation library, specifically serving the cellular (Worley/Voronoi) noise generation pipeline. It embodies the *Strategy* design pattern, encapsulating a specific distance calculation algorithm.

Its sole responsibility is to compute the distance between a given sample point and a cell's feature point. The result of this computation is then registered into a ResultBuffer, which tracks the closest cell points found so far. The choice of distance function (e.g., Euclidean, Manhattan) directly influences the final visual appearance of the generated noise pattern, producing distinct shapes and textures.

This class is designed to be a stateless, high-performance computation unit. The provision of pre-configured static instances for common distance metrics (EUCLIDEAN, MANHATTAN) is a performance optimization using the *Flyweight* pattern, minimizing object allocation during intensive generation tasks.

## Lifecycle & Ownership
- **Creation:** Instances are created in one of two ways:
    1.  **Static Access:** The common evaluators (EUCLIDEAN, MANHATTAN, etc.) are static final fields, instantiated once at class-loading time.
    2.  **Factory Method:** The static factory method of(PointDistanceFunction) is the preferred constructor. It will return a shared static instance if the provided function is a known type, otherwise it will allocate a new NormalPointEvaluator.
- **Scope:** Static instances are application-scoped and persist for the entire runtime. Dynamically created instances are typically transient, scoped to a single procedural generation job.
- **Destruction:** There is no explicit destruction logic. Instances are managed by the Java Garbage Collector.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state consists of a single final field, distanceFunction, which is set at construction time and never modified.
- **Thread Safety:** **Thread-safe**. Due to its immutable nature and lack of side effects, a single NormalPointEvaluator instance can be safely shared and used concurrently by multiple worker threads during parallelized world generation without requiring any locks or synchronization.

## API Surface
The public API is minimal, focusing exclusively on the evaluation contract defined by the PointEvaluator interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| evalPoint(...) | void | O(1) | Calculates 2D distance and registers the result in the buffer. |
| evalPoint2(...) | void | O(1) | Calculates 2D distance and registers using the buffer's secondary slot (for F2 noise). |
| evalPoint(...) | void | O(1) | Calculates 3D distance and registers the result in the buffer. |
| evalPoint2(...) | void | O(1) | Calculates 3D distance and registers using the buffer's secondary slot. |
| of(distanceFunction) | PointEvaluator | O(1) | Static factory. Returns a shared instance for known functions or a new one. |

## Integration Patterns

### Standard Usage
The evaluator is typically obtained via its static factory or pre-defined fields and passed into a higher-level noise generator. It should be retrieved once and reused for the entire generation process.

```java
// Retrieve a shared, pre-configured evaluator for standard Euclidean distance
PointEvaluator evaluator = NormalPointEvaluator.EUCLIDEAN;

// In a generation loop, the noise algorithm invokes the evaluator
// (Conceptual example)
for (Cell cell : nearbyCells) {
    evaluator.evalPoint(seed, sampleX, sampleY, cell.hash, ..., resultBuffer);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NormalPointEvaluator()` with a standard distance function. This bypasses the Flyweight optimization and creates unnecessary objects. Always prefer the `NormalPointEvaluator.of()` factory or the static fields.
- **Instantiation in a Loop:** Never create a NormalPointEvaluator instance inside a tight generation loop. This will cause significant performance degradation due to excessive garbage collection pressure. An evaluator should be considered a long-lived service object for the duration of a task.

## Data Pipeline
This component acts as a computational step in the cellular noise data flow. It does not initiate or terminate a pipeline but is a critical processing node.

> Flow:
> Noise Generator -> Iterates Grid Cells -> **NormalPointEvaluator.evalPoint** -> Writes to ResultBuffer -> Noise Generator reads final values from ResultBuffer

