---
description: Architectural reference for CuboidPattern
---

# CuboidPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient Component

## Definition
```java
// Signature
public class CuboidPattern extends Pattern {
```

## Architecture & Concepts
The CuboidPattern is a structural component within the procedural world generation framework. It does not represent a material or entity itself, but rather acts as a spatial quantifier. It enforces a volumetric constraint on a nested pattern, known as the *subPattern*.

Architecturally, this class implements the **Composite** design pattern. It composes a single `Pattern` child and delegates matching logic to it across a three-dimensional grid. Its core responsibility is to answer the question: "Does the provided sub-pattern hold true for every single block within this defined rectangular volume?"

This pattern is fundamental for defining solid structures, clearing volumes of space (e.g., rooms filled with air), or ensuring a consistent condition across a geometric area. It is the primary mechanism for transitioning from point-based pattern matching to volume-based assertions.

## Lifecycle & Ownership
-   **Creation:** Instantiated programmatically by higher-level generation logic, such as a `StructureDefinition` or a `BiomeFeatureGenerator`. It is typically created as a node within a larger, more complex pattern tree that defines a complete feature.
-   **Scope:** The object's lifetime is ephemeral, scoped strictly to the execution of a single world generation task. It does not persist beyond the evaluation of the pattern tree it belongs to.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as the root pattern evaluation is complete and the generator no longer holds a reference to it. There is no manual destruction or cleanup required.

## Internal State & Concurrency
-   **State:** This object is **effectively immutable**. Its internal state, consisting of the sub-pattern and the min/max boundary vectors, is established at construction and never modified. The `readSpaceSize` field is a pre-computed cache to optimize boundary checks, also set only during instantiation.
-   **Thread Safety:** The CuboidPattern is inherently **thread-safe**. Its immutable nature allows a single instance to be safely evaluated by multiple world generator threads concurrently without risk of data races or the need for external locking. The `matches` method operates on a clone of the provided context, ensuring that state modifications are local to the current evaluation stack.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(context) | boolean | O(N\*M\*P) | Evaluates the sub-pattern at every coordinate within the cuboid's volume. Returns false immediately if any check fails. The complexity is directly proportional to the volume of the cuboid. |
| readSpace() | SpaceSize | O(1) | Returns a clone of the pre-calculated bounding box for this pattern. Used by the generator for optimization and planning. |

## Integration Patterns

### Standard Usage
The canonical use case is to wrap a simple pattern, such as a `MaterialPattern`, to define a solid, volumetric structure.

```java
// Define a 5x5x5 solid cube of stone as a pattern
Pattern stonePattern = new MaterialPattern(BuiltInMaterials.STONE);
Vector3i minBounds = new Vector3i(0, 0, 0);
Vector3i maxBounds = new Vector3i(4, 4, 4);

// The CuboidPattern enforces that the stonePattern must match everywhere inside the bounds
Pattern stoneCube = new CuboidPattern(stonePattern, minBounds, maxBounds);

// The world generator would then use this pattern to validate or place the structure
boolean canPlaceCube = stoneCube.matches(generatorContext);
```

### Anti-Patterns (Do NOT do this)
-   **Excessive Volume:** Do not define a CuboidPattern with extremely large dimensions (e.g., 256x256x256). The `matches` method iterates through every single block, and its cost grows cubically with the side length. This will cause severe performance stalls during world generation. Use hierarchical patterns or other techniques for large-scale regions.
-   **Invalid Bounds:** Constructing a pattern where any component of `min` is greater than the corresponding component of `max` will result in a pattern that always returns `true` because the iteration loops will never execute. This can lead to silent failures and unexpected generation outcomes.

## Data Pipeline
The CuboidPattern acts as a volumetric iterator and delegator within the pattern evaluation pipeline. It receives a single evaluation request and transforms it into a multitude of requests for its child pattern.

> Flow:
> World Generator invokes `matches(context)` on `CuboidPattern` -> The pattern calculates the absolute world-space scan volume -> It enters a triple-nested loop for X, Y, and Z axes -> For each coordinate, it calls `subPattern.matches(childContext)` -> If any sub-pattern call returns `false`, the entire operation is aborted and returns `false`. If all calls succeed, it returns `true`.

