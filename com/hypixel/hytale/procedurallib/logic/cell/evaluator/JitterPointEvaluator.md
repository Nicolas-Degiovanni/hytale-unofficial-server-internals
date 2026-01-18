---
description: Architectural reference for JitterPointEvaluator
---

# JitterPointEvaluator

**Package:** com.hypixel.hytale.procedurallib.logic.cell.evaluator
**Type:** Transient

## Definition
```java
// Signature
public class JitterPointEvaluator implements PointEvaluator {
```

## Architecture & Concepts
The JitterPointEvaluator is a specialized implementation of the PointEvaluator interface that follows the **Decorator** design pattern. Its primary role is not to perform evaluation itself, but to associate a specific CellJitter strategy with another PointEvaluator instance.

This class acts as a container or a composition object. It wraps a concrete PointEvaluator and holds a reference to a CellJitter. Higher-level systems, such as a CellEvaluator, can then query this object for its jittering strategy via the getJitter method before proceeding with the actual point evaluation.

Critically, the JitterPointEvaluator **does not apply the jitter logic itself**. All evaluation methods, such as evalPoint, are direct, unmodified pass-through calls to the wrapped PointEvaluator. This design cleanly separates the concern of *how* to jitter cell points from the concern of *what* calculation to perform at those points. It allows for the flexible composition of various jittering and evaluation algorithms within the procedural generation pipeline.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly by a factory or builder responsible for assembling a procedural generation pipeline. It requires a pre-existing PointEvaluator and CellJitter during construction.
-   **Scope:** Its lifetime is ephemeral, scoped to a single procedural generation task. It is not a long-lived service.
-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup as soon as the generation task that created it completes and all references to it are dropped. No explicit cleanup is required.

## Internal State & Concurrency
-   **State:** The internal state consists of two final fields, pointEvaluator and jitter, which are assigned at construction. The state of JitterPointEvaluator itself is therefore **immutable**.
-   **Thread Safety:** This class contains no mutable state or synchronization primitives, making it inherently thread-safe. However, its overall thread safety is entirely dependent on the thread safety of the wrapped PointEvaluator and CellJitter objects. If the composed objects are not thread-safe, then concurrent use of this class will also be unsafe. All evaluation calls are delegated directly, so any concurrency constraints of the wrapped implementation apply.

## API Surface
The public contract is dominated by direct delegation, with getJitter being the only method that provides unique behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| JitterPointEvaluator(evaluator, jitter) | constructor | O(1) | Constructs the decorator, wrapping a target evaluator and associating a jitter strategy. |
| getJitter() | CellJitter | O(1) | Returns the associated jittering strategy. This is the primary method used by consuming systems. |
| evalPoint(...) | void | *Delegated* | Forwards the call directly to the wrapped PointEvaluator. Complexity is determined by the wrapped implementation. |
| evalPoint2(...) | void | *Delegated* | Forwards the call directly to the wrapped PointEvaluator. Complexity is determined by the wrapped implementation. |
| collectPoint(...) | void | *Delegated* | Forwards the call directly to the wrapped PointEvaluator. Complexity is determined by the wrapped implementation. |

## Integration Patterns

### Standard Usage
The class is used to compose a jitter strategy with a point evaluation strategy. The resulting object is then passed to a higher-level system that understands how to interact with this composition.

```java
// 1. Define the base evaluation logic
PointEvaluator baseEvaluator = new DistanceEvaluator();

// 2. Define the jittering strategy
CellJitter jitterStrategy = new RandomJitter(0.5);

// 3. Compose them using the JitterPointEvaluator
PointEvaluator compositeEvaluator = new JitterPointEvaluator(baseEvaluator, jitterStrategy);

// 4. Pass the composite object to a higher-level consumer
// This consumer will call getJitter() and evalPoint() separately.
cellGrid.evaluateWith(compositeEvaluator);
```

### Anti-Patterns (Do NOT do this)
-   **Assuming Jitter Application:** Do not assume that this class applies the jitter to coordinates before passing them to the wrapped evaluator. It does not. It is merely a data carrier. The consuming system is responsible for retrieving the jitter strategy and applying it.
-   **Null Composition:** The constructor does not perform null checks. Constructing this class with a null pointEvaluator or a null jitter will result in a NullPointerException during a subsequent method call. Always provide valid, non-null instances.

## Data Pipeline
JitterPointEvaluator does not participate in a data transformation pipeline directly. Instead, it facilitates a *control flow* pattern where a consumer orchestrates the interaction between the jitter and the evaluator.

> Flow:
> `CellEvaluator` requests evaluation -> `CellEvaluator` calls `getJitter()` on **JitterPointEvaluator** -> `CellJitter` strategy is returned -> `CellEvaluator` uses the strategy to calculate jittered coordinates -> `CellEvaluator` calls `evalPoint()` on **JitterPointEvaluator** with the new coordinates -> Call is delegated to the wrapped `PointEvaluator` -> `ResultBuffer` is populated by the wrapped evaluator.

