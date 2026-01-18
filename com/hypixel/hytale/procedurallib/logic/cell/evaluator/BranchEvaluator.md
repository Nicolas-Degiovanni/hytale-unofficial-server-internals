---
description: Architectural reference for BranchEvaluator
---

# BranchEvaluator

**Package:** com.hypixel.hytale.procedurallib.logic.cell.evaluator
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class BranchEvaluator implements PointEvaluator {
```

## Architecture & Concepts
The BranchEvaluator is a concrete implementation of the PointEvaluator strategy interface. It is a specialized component within the procedural generation library responsible for creating linear, branching Worley noise patterns. These patterns are essential for generating features like rivers, canyons, cave networks, and veins of ore.

Unlike standard point-based evaluators that measure distance to a single cell's feature point, the BranchEvaluator calculates the distance from a given world coordinate to a *line segment*. This segment connects the feature point of a parent cell to the feature point of a deterministically chosen adjacent cell. This "branching" behavior is the core of its function.

The choice of the adjacent cell is governed by the configured Direction enum and the hash of the parent cell, allowing for controlled, repeatable patterns that can flow outwards from a source, inwards towards a sink, or in a pseudo-random direction. The result is a field of values representing the distance to the nearest "branch," which can then be used to carve features into the world geometry.

## Lifecycle & Ownership
- **Creation:** A BranchEvaluator is instantiated by a higher-level procedural generation service or factory. It is never created directly by game logic. Its constructor requires a complex set of dependencies, including distance and point functions, a jitter algorithm, and scaling parameters, which are typically configured per-biome or per-feature type.
- **Scope:** The object's lifetime is ephemeral, strictly tied to a single, discrete generation task (e.g., generating the data for one world region). It holds no state related to the overall world.
- **Destruction:** Once the generation task is complete, the BranchEvaluator is dereferenced by the orchestrator and becomes eligible for garbage collection. It does not manage any native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The BranchEvaluator is effectively **immutable** post-construction. Its core parameters are stored in final fields, and several derived values used for normalization and scaling are pre-calculated in the constructor for performance. This design ensures that its behavior is perfectly consistent and predictable for a given configuration.
- **Thread Safety:** This class is **thread-safe**. Its immutable nature means that a single instance can be safely shared and used by multiple worker threads simultaneously to evaluate different world coordinates. The primary evaluation method, evalPoint, is a pure function with no side effects on the evaluator's internal state.

    **Warning:** While the evaluator itself is thread-safe, the caller is responsible for ensuring that concurrent calls do not cause race conditions on the shared ResultBuffer object passed as an argument. Each thread should operate on a separate buffer or use a thread-safe buffer implementation.

## API Surface
The public contract is minimal, focused entirely on the PointEvaluator interface. The 3D evaluation methods are intentionally unimplemented, as this evaluator is designed for 2D planar operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| evalPoint(seed, x, y, ...) | void | O(1) | Core evaluation method. Calculates distance to the nearest branch and registers the result in the provided buffer. |
| getJitter() | CellJitter | O(1) | Returns the configured jitter algorithm instance. |
| evalPoint2(...) | void | O(1) | No-op. This evaluator does not support the secondary evaluation pass. |
| evalPoint(..., z, ...) | void | O(1) | No-op. This is a 2D-only evaluator; 3D calls are ignored. |

## Integration Patterns

### Standard Usage
The BranchEvaluator is not used in isolation. It is provided to a procedural generation orchestrator, which invokes it for each point within a target area. The results are collected in a buffer for further processing.

```java
// A procedural orchestrator uses the evaluator to populate a result buffer.
PointEvaluator branchEval = generatorFactory.createBranchEvaluator(config);
ResultBuffer.ResultBuffer2d results = new ResultBuffer.ResultBuffer2d();

// For each point (x, y) in the generation region...
// ...retrieve the parent cell data (cax, cay, ax, ay)...
branchEval.evalPoint(seed, x, y, hashA, cax, cay, ax, ay, results);

// After evaluation, the 'results' buffer contains the final distance data.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BranchEvaluator()`. The constructor's parameters are highly interdependent and must be configured by a factory that understands the scaling and normalization rules of the broader procedural system. Incorrect parameters will lead to visual artifacts and discontinuities in the generated world.
- **Ignoring 2D Limitation:** Do not attempt to use this evaluator in a 3D procedural context. The `evalPoint` methods accepting a Z coordinate are empty stubs and will produce no output.
- **Stateful Misuse:** Do not attempt to modify the evaluator's state after creation via reflection or other means. The system relies on its immutability for both thread safety and predictable output.

## Data Pipeline
The evaluator transforms a world coordinate into a distance value by executing a well-defined sequence of steps.

> Flow:
> World Coordinate (x, y) & Parent Cell Data -> **BranchEvaluator.evalPoint** -> Determine Connection Vector -> Identify Adjacent "Branch" Cell -> Calculate Jittered Branch Point -> Calculate Distance to Line Segment -> Normalize Distance -> Write to ResultBuffer

