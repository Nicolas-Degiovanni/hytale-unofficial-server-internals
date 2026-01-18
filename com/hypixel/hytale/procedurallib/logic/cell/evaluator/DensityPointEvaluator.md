---
description: Architectural reference for DensityPointEvaluator
---

# DensityPointEvaluator

**Package:** com.hypixel.hytale.procedurallib.logic.cell.evaluator
**Type:** Transient Decorator

## Definition
```java
// Signature
public class DensityPointEvaluator implements PointEvaluator {
```

## Architecture & Concepts
The DensityPointEvaluator is a foundational component in the procedural generation pipeline, implementing the **Decorator** design pattern. Its primary function is to act as a conditional filter or a stochastic gate for another PointEvaluator.

It wraps a concrete PointEvaluator and intercepts all evaluation calls (evalPoint, collectPoint). Before delegating the call to the wrapped evaluator, it consults an internal density condition. This condition, evaluated against a deterministic hash of the cell coordinates (cellHash), decides whether the point evaluation should proceed.

This mechanism allows for the control of feature density in world generation. For example, a DensityPointEvaluator can be used to thin out the placement of trees or rocks, ensuring they only appear in a certain percentage of cells without requiring complex noise map calculations. It effectively culls evaluation requests based on a probabilistic check, making it a highly efficient tool for controlling the distribution of procedurally placed content.

## Lifecycle & Ownership
- **Creation:** A DensityPointEvaluator is instantiated by a higher-level system responsible for configuring procedural generation layers or passes. It is constructed by composing a target PointEvaluator with a density condition (either an IIntCondition or an IDoubleCondition).
- **Scope:** The object's lifetime is ephemeral, typically scoped to a single world generation task or the evaluation of a specific region. It does not persist between sessions or across distinct generation passes.
- **Destruction:** The object is managed by the Java garbage collector. Once the generation task that created it is complete, the DensityPointEvaluator is eligible for collection. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state, consisting of the wrapped pointEvaluator and the density condition, is established at construction and is **immutable**. The fields are final, and the object provides no methods for altering its configuration post-instantiation. It does not cache evaluation results.
- **Thread Safety:** This class is inherently **thread-safe**. Its methods are pure functions of their inputs and contain no mutable state. However, its overall thread safety is contingent upon the thread safety of the wrapped PointEvaluator and IIntCondition instances provided during construction. In standard engine usage, these dependencies are also designed to be thread-safe, allowing multiple world generation chunks to be processed in parallel.

## API Surface
The public contract mirrors the PointEvaluator interface. The core logic is identical across all evaluation methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DensityPointEvaluator(evaluator, condition) | constructor | O(1) | Constructs the decorator, wrapping a target evaluator with a density condition. |
| getJitter() | CellJitter | O(1) | Delegates directly to the wrapped PointEvaluator to retrieve its jitter configuration. |
| evalPoint(...) | void | O(1) + O(wrapped) | Evaluates the density condition. If true, delegates to the wrapped evaluator. If false, the call terminates immediately. |
| evalPoint2(...) | void | O(1) + O(wrapped) | A secondary evaluation path. Behaves identically to evalPoint regarding the density check. |
| collectPoint(...) | void | O(1) + O(wrapped) | Evaluates the density condition. If true, delegates the point collection to the wrapped evaluator. |

## Integration Patterns

### Standard Usage
The DensityPointEvaluator is designed to be chained as part of a larger evaluation pipeline. A developer configures a base evaluator and then wraps it to control its invocation frequency.

```java
// A base evaluator that places a specific feature
PointEvaluator featurePlacer = new FeaturePlacerEvaluator(...);

// A condition that allows evaluation in ~30% of cells
IDoubleCondition densityThreshold = new DoubleConstantCondition(0.3);

// Wrap the feature placer to apply the density filter
PointEvaluator sparseFeaturePlacer = new DensityPointEvaluator(featurePlacer, densityThreshold);

// Use the wrapped evaluator in the generation system
proceduralLayer.setPointEvaluator(sparseFeaturePlacer);
```

### Anti-Patterns (Do NOT do this)
- **Redundant Wrapping:** Do not wrap a PointEvaluator with a null or always-true density condition. While the system handles this by defaulting to an always-true predicate, it adds unnecessary object allocation and method call overhead. If no filtering is required, use the base PointEvaluator directly.
- **Stateful Conditions:** Avoid using an IIntCondition that relies on external, mutable state. The evaluation of the condition should be a pure function of the input cellHash to ensure deterministic and thread-safe world generation.

## Data Pipeline
The component acts as a conditional gateway in the data flow of point evaluation.

> Flow:
> Generation System -> **DensityPointEvaluator**.evalPoint(cellHash, ...) -> Density Condition Check -> (if false, terminate) -> (if true) -> Wrapped **PointEvaluator**.evalPoint(...) -> ResultBuffer

