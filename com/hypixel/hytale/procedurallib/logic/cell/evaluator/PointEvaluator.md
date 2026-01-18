---
description: Architectural reference for PointEvaluator
---

# PointEvaluator

**Package:** `com.hypixel.hytale.procedurallib.logic.cell.evaluator`
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface PointEvaluator {
    // Methods
}
```

## Architecture & Concepts

The PointEvaluator interface defines a core strategic contract within Hytale's procedural generation library, specifically for cellular and Worley noise algorithms. It represents a single, pluggable step in a larger data processing pipeline responsible for evaluating a candidate point within a procedural grid cell. Its primary function is to calculate a value (typically distance-based) and write the result into a shared buffer.

This interface is the foundation of a highly flexible system built using the **Decorator** and **Factory** design patterns. The static factory method, `PointEvaluator.of`, acts as a builder. It composes a chain of PointEvaluator implementations, where each decorator adds a specific behavior such as jittering, density-based culling, or distance modification.

This compositional approach allows for the dynamic creation of complex evaluation logic from simple, reusable components. Instead of a rigid inheritance hierarchy, the engine can construct a bespoke evaluation pipeline tailored to the specific requirements of a biome or procedural feature. This system is central to generating varied and controllable procedural patterns.

## Lifecycle & Ownership

-   **Creation:** PointEvaluator instances are almost exclusively created via the static `PointEvaluator.of` factory method. This construction typically occurs during the initialization phase of a larger procedural generation task, such as when a world generation "recipe" or "graph" is being assembled. The entity configuring the generation task is the owner.
-   **Scope:** An instance of a PointEvaluator is a configuration object. Its lifetime is bound to the procedural generation process that uses it. It is not a global singleton and does not persist between sessions. It is created, used to process a large number of points for a specific generation task, and then becomes eligible for garbage collection.
-   **Destruction:** Cleanup is managed by the standard Java garbage collector. Implementations hold no native resources and do not require explicit destruction.

## Internal State & Concurrency

-   **State:** The interface itself is stateless. However, the concrete implementations created by the `of` factory (e.g., JitterPointEvaluator, DensityPointEvaluator) are stateful. They hold references to their configuration objects (like a CellJitter or IDoubleCondition) and the next PointEvaluator in the decorator chain. This state is established at creation and is **effectively immutable** during the evaluation phase.
-   **Thread Safety:** The entire PointEvaluator chain is designed to be **thread-safe** and is critical for performant, parallelized world generation. The evaluation methods (`evalPoint`, `evalPoint2`) are pure functions with respect to the evaluator's internal state. They operate solely on their input arguments and write to the provided ResultBuffer, which is expected to be a thread-local or per-call object.

**WARNING:** Custom implementations of this interface must be fully re-entrant and thread-safe. Modifying internal state during an `evalPoint` call will lead to severe race conditions and non-deterministic generation when executed in a multi-threaded environment.

## API Surface

The public contract is focused on point evaluation and compositional construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(...) | static PointEvaluator | O(1) | Factory method. Constructs a decorated chain of evaluators based on the provided configuration. |
| evalPoint(...) | void | O(C) | Evaluates a 2D point. Writes results to the ResultBuffer. Complexity depends on the composed functions. |
| evalPoint2(...) | void | O(C) | Evaluates a 2D point (alternate signature). Writes results to the ResultBuffer. |
| evalPoint(...) | void | O(C) | Evaluates a 3D point. Writes results to the ResultBuffer. |
| evalPoint2(...) | void | O(C) | Evaluates a 3D point (alternate signature). Writes results to the ResultBuffer. |
| getJitter() | CellJitter | O(1) | Returns the jitter strategy for this evaluator. Defaults to no jitter. |
| collectPoint(...) | void | O(1) | A utility for collecting the final point coordinates after all modifications. |

## Integration Patterns

### Standard Usage

The correct pattern is to use the static `of` factory to build a complete evaluation strategy from various components. This configured object is then passed to a higher-level procedural system that iterates over grid cells.

```java
// 1. Define the components for the evaluation strategy
PointDistanceFunction distanceFunc = PointDistanceFunction.EUCLIDEAN;
IDoubleCondition densityCheck = new SomeDensityCondition(); // e.g., based on another noise layer
CellJitter jitter = new DefaultCellJitter(0.8);
int skipCount = 2;

// 2. Use the factory to compose the PointEvaluator chain
PointEvaluator evaluator = PointEvaluator.of(
    distanceFunc,
    densityCheck,
    null, // No distance modification
    skipCount,
    SkipCellPointEvaluator.Mode.MODE_1,
    jitter
);

// 3. Pass the configured evaluator to a higher-level generation system
// (Conceptual example)
proceduralEngine.generateRegion(regionCoords, evaluator);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not manually instantiate concrete implementations like `JitterPointEvaluator` or `DensityPointEvaluator`. The `PointEvaluator.of` factory guarantees the correct ordering and composition of the decorator chain. Manual construction is brittle and error-prone.
-   **Stateful Evaluation:** Do not implement a PointEvaluator that modifies its own fields inside the `evalPoint` methods. This breaks thread safety and will cause unpredictable results in the parallel generation engine.
-   **Reusing Result Buffers:** The `ResultBuffer` passed into `evalPoint` is not guaranteed to be thread-safe. Do not hold a reference to it or access it from other threads. It is intended for immediate, synchronous writes within the scope of the `evalPoint` call.

## Data Pipeline

The PointEvaluator forms a processing chain where data flows through each decorator before the final calculation. Each step can either modify the data or terminate the evaluation for that point.

> Flow:
> Cell Coordinates & Hash -> **[JitterPointEvaluator]** -> Jittered Coordinates -> **[SkipCellPointEvaluator]** -> (Maybe Terminated) -> **[DensityPointEvaluator]** -> (Maybe Terminated) -> **[DistancePointEvaluator]** -> Modified Distance Params -> **[NormalPointEvaluator]** -> Final Calculation -> **ResultBuffer**

