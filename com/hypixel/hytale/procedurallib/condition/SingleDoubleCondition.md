---
description: Architectural reference for SingleDoubleCondition
---

# SingleDoubleCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Value Object / Transient

## Definition
```java
// Signature
public class SingleDoubleCondition implements IDoubleCondition {
```

## Architecture & Concepts
The SingleDoubleCondition class is a fundamental predicate component within the procedural generation library. It serves as a concrete implementation of the IDoubleCondition strategy interface, encapsulating a single, immutable threshold for a "less than" comparison.

Its primary role is to act as a simple, high-performance rule in complex decision-making systems, such as biome placement, terrain feature generation, or resource distribution. By representing a condition as an object, the engine can compose, serialize, and reuse generation logic dynamically. This class forms the most granular, atomic unit of a conditional check against a continuous double value, typically sourced from a noise map or other mathematical function.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by higher-level procedural systems, such as a BiomeRuleEvaluator or a TerrainGenerator. It is not a managed service and should be created with the specific threshold value required for a given rule.
- **Scope:** Short-lived and transient. The lifecycle of a SingleDoubleCondition instance is typically bound to the evaluation of a single procedural rule or the generation of a single world chunk.
- **Destruction:** The object holds no external resources and is managed entirely by the Java Garbage Collector. Once all references are dropped, it is eligible for collection.

## Internal State & Concurrency
- **State:** **Immutable**. The internal threshold, *value*, is a final field set exclusively during construction. The object's state cannot be modified post-instantiation, making its behavior perfectly predictable.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, an instance of SingleDoubleCondition can be safely shared and accessed by multiple threads simultaneously without any requirement for external locking or synchronization. This is a critical design feature for enabling parallelized world generation.

## API Surface
The public contract is minimal, focusing exclusively on the evaluation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(double value) | boolean | O(1) | Evaluates if the provided input *value* is strictly less than the internal threshold. |

## Integration Patterns

### Standard Usage
The class is intended to be instantiated and used as a predicate function. It is often held by a parent system that applies the condition to a stream of input values.

```java
// Example: Determine if a terrain height is below sea level
double seaLevelThreshold = 64.0;
IDoubleCondition belowSeaLevel = new SingleDoubleCondition(seaLevelThreshold);

// In a generation loop
for (int x = 0; x < 16; x++) {
    double currentHeight = noise.get(x);
    if (belowSeaLevel.eval(currentHeight)) {
        // Place water block
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instantiation in a Loop:** Avoid creating new instances within high-frequency loops. The object is lightweight, but repeated allocation can cause unnecessary GC pressure. Instantiate it once outside the loop.
- **Misinterpretation of Logic:** The evaluation is a strict "less than" (`<`) comparison. Do not use this class when a "less than or equal to" (`<=`) check is required. Use a different condition implementation for that purpose.

## Data Pipeline
SingleDoubleCondition acts as a filter or gate within a larger data processing flow, converting a continuous value into a binary decision.

> Flow:
> Noise Function -> Raw double value -> **SingleDoubleCondition.eval()** -> Boolean result -> World Generation Logic

