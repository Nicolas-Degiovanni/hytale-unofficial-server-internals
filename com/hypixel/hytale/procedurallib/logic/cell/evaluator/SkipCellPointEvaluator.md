---
description: Architectural reference for SkipCellPointEvaluator
---

# SkipCellPointEvaluator

**Package:** com.hypixel.hytale.procedurallib.logic.cell.evaluator
**Type:** Transient

## Definition
```java
// Signature
public class SkipCellPointEvaluator implements PointEvaluator {
```

## Architecture & Concepts
The SkipCellPointEvaluator is a specialized implementation of the PointEvaluator interface that acts as a **decorator** or **filter**. Its primary function is to conditionally bypass the evaluation of grid cells based on a deterministic, periodic pattern. This is a critical performance optimization strategy within the procedural generation library.

In a large-scale procedural world, evaluating every single cell for features can be computationally prohibitive. This class provides a mechanism to create sparse or multi-resolution feature maps by ensuring that an underlying, potentially expensive, PointEvaluator is only invoked for a subset of cells. For example, it can be used to place a major feature in every 16th cell, creating a coarse grid, while skipping all intermediate cells.

It operates on the cell's integer coordinates (cellX, cellY) and uses bitwise operations for high-performance filtering. It supports two primary filtering patterns: a simple grid and a checkerboard layout. This component is fundamental for implementing Level of Detail (LOD) systems or for layering different densities of procedural content.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level procedural generation controller or graph system. It is constructed by wrapping an existing PointEvaluator instance, effectively intercepting calls to it. Multiple distinct instances of SkipCellPointEvaluator can and will exist simultaneously, each with its own skipping period, mode, and wrapped evaluator.
-   **Scope:** The object's lifetime is tied to the specific procedural generation task it is configured for. It is typically a short-lived object, created for the generation of a specific world region or feature layer and then discarded.
-   **Destruction:** Managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The internal state is **immutable** after construction. The wrapped PointEvaluator, Mode, and the calculated integer fields *mask* and *mid* are set once in the constructor and are never modified. The state is derived entirely from the constructor's *period* argument, which is normalized to the next power of two to facilitate efficient bitwise checks.
-   **Thread Safety:** This class is inherently thread-safe. As its internal state is immutable, the same instance can be safely shared and accessed by multiple worker threads during a parallelized world generation process. However, its overall thread safety is contingent upon the thread safety of the wrapped PointEvaluator it delegates to. This class introduces no new concurrency hazards.

## API Surface
The public API mirrors the PointEvaluator interface. The core logic resides in the conditional delegation within these methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SkipCellPointEvaluator(evaluator, mode, period) | constructor | O(1) | Creates a new evaluator that wraps an existing one with a specific skipping pattern. |
| getJitter() | CellJitter | O(1) | Delegates the call directly to the wrapped PointEvaluator. |
| evalPoint(seed, x, y, ...) | void | O(1) | Performs the skip check. If the cell is not skipped, it delegates the call to the wrapped evaluator. Otherwise, it returns immediately. |
| collectPoint(cellHash, cellX, cellY, ...) | void | O(1) | Performs the skip check. If the cell is not skipped, it delegates the call to the wrapped evaluator. Otherwise, it returns immediately. |

**WARNING:** The 3D evaluation methods, `evalPoint` and `evalPoint2` which accept a Z coordinate, are currently empty stubs. This implementation is strictly for 2D cell skipping and will silently fail to produce any 3D output.

## Integration Patterns

### Standard Usage
This class is intended to be used as part of a chain or composition of PointEvaluators. A base evaluator is created first, then wrapped by the SkipCellPointEvaluator to control its invocation frequency.

```java
// 1. Define the base evaluator that performs the actual work
PointEvaluator expensiveEvaluator = new SomeComplexFeatureEvaluator();

// 2. Wrap it with a SkipCellPointEvaluator to run it only on a 16x16 grid
int period = 16;
PointEvaluator sparseEvaluator = new SkipCellPointEvaluator(
    expensiveEvaluator,
    SkipCellPointEvaluator.Mode.GRID,
    period
);

// 3. Use the sparseEvaluator in the generation pipeline.
//    Calls for most cells will now return instantly.
generator.evaluateWith(sparseEvaluator);
```

### Anti-Patterns (Do NOT do this)
-   **Misunderstanding Thread Safety:** Do not assume this class makes an underlying evaluator thread-safe. If you wrap a non-thread-safe PointEvaluator, the resulting system will remain non-thread-safe.
-   **Expecting 3D Evaluation:** Do not use this class in a 3D generation pipeline expecting it to skip cells in 3D space. The 3D evaluation methods are non-functional and will produce no points.
-   **Using a Zero Period:** While the constructor sanitizes the input, providing a period of 0 or 1 is logically flawed as it defeats the purpose of the class, leading to either no skipping or unpredictable behavior depending on the power-of-two alignment.

## Data Pipeline
The data flow is a simple conditional gate. The decision to proceed is made at the very beginning of the evaluation call based on the cell's grid coordinates.

> Flow:
> Generation System -> `evalPoint(cellX, cellY)` -> **SkipCellPointEvaluator.skip(cellX, cellY)** -> [IF SKIPPED: return] -> [IF NOT SKIPPED: `wrappedEvaluator.evalPoint()`] -> ResultBuffer

