---
description: Architectural reference for MaterialPattern
---

# MaterialPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class MaterialPattern extends Pattern {
```

## Architecture & Concepts
The MaterialPattern is an atomic component within the Hytale world generation framework. It represents the most fundamental conditional check: "Does the block at a given coordinate match a specific material?". It serves as a concrete implementation of the abstract Pattern class, embodying a leaf node in a potential tree of generation rules.

This class is a key building block for creating complex procedural logic. Higher-order patterns, such as sequence or composite patterns, use instances of MaterialPattern to define the specific block types required for a larger structure or feature to be generated. For example, a "wall" pattern might be composed of multiple MaterialPattern checks for stone or brick materials in a specific arrangement.

Its primary role is to decouple the logic of *what* to check for (the Material) from the engine that performs the check.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by higher-level generator services, such as a biome or structure generator. A new instance is created for each unique material condition required by a generation rule. Example: `new MaterialPattern(BuiltinMaterials.OAK_LOG)`.
- **Scope:** Short-lived and function-scoped. An instance typically exists only for the duration of a single, larger pattern evaluation. It holds no resources and is designed to be lightweight.
- **Destruction:** The object is eligible for garbage collection as soon as the evaluation that created it completes and no further references exist. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Immutable. The core state is the `material` field, which is marked as `final` and set exclusively via the constructor. The class does not modify its own state or any external state during its operation.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutable nature, a single instance can be safely shared and accessed by multiple generator threads simultaneously without locks or synchronization primitives. The `matches` method is a pure function with no side effects.

## API Surface
The public contract is minimal, adhering to the interface defined by the parent `Pattern` class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Pattern.Context context) | boolean | O(1) | The core evaluation method. Returns true if the material at the context position matches the internal material. |
| readSpace() | SpaceSize | O(1) | Returns the spatial footprint required for evaluation, which is always a single 1x1x1 block volume. |

## Integration Patterns

### Standard Usage
A MaterialPattern is almost never used in isolation. It is constructed and passed to more complex patterns or rule evaluators as part of a larger generation algorithm.

```java
// A higher-level system defines a rule that a tree trunk must be made of oak.
Material oakLogMaterial = BuiltinMaterials.OAK_LOG;
Pattern trunkPattern = new MaterialPattern(oakLogMaterial);

// The world generator provides the context for a specific location.
Pattern.Context evaluationContext = createGeneratorContextAt(new Vector3i(10, 20, 30));

// The pattern is evaluated. The generator uses this result to guide its next action.
if (trunkPattern.matches(evaluationContext)) {
    // Place leaves on top of this block
}
```

### Anti-Patterns (Do NOT do this)
- **Null Construction:** Do not pass a null `Material` to the constructor. The `@Nonnull` annotation enforces this contract, and violating it will result in a runtime exception.
- **Stateful Wrapping:** Do not wrap this class in a stateful manager. Its design as a lightweight, immutable value object is intentional. Caching or pooling these objects is unnecessary and complicates the generator design for negligible performance gain.

## Data Pipeline
The data flow for a MaterialPattern is a simple, linear evaluation. It acts as a predicate function within the generator's data stream.

> Flow:
> World Generator -> `Pattern.Context` (Contains target position and a `MaterialSpace` view of the world) -> **MaterialPattern.matches()** -> Material lookup at position -> Comparison -> `boolean` result -> World Generator Decision Logic

