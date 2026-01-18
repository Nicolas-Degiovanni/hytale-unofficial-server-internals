---
description: Architectural reference for RandomCoordinateCondition
---

# RandomCoordinateCondition

**Package:** com.hypixel.hytale.server.worldgen.util.condition
**Type:** Transient

## Definition
```java
// Signature
public class RandomCoordinateCondition implements ICoordinateCondition {
```

## Architecture & Concepts
RandomCoordinateCondition is a fundamental component within the procedural world generation framework. It serves as a concrete implementation of the ICoordinateCondition strategy interface, providing a simple yet powerful predicate for probabilistic object placement.

Its core architectural purpose is to enable **deterministic randomness**. By leveraging the HashUtil utility, it guarantees that for a given world seed and coordinate, the evaluation result will be identical across all executions. This property is non-negotiable for ensuring that worlds can be regenerated repeatably.

This class is not a service or a manager. It is a lightweight, data-driven object that encapsulates a single piece of logic: "Should something happen at this location, based on a random chance?". It is designed to be composed by higher-level systems, such as biome generators or feature placement rules, to control the density and distribution of world elements like trees, ores, or decorative foliage.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by world generation configuration loaders or rule engines. They are typically instantiated when a biome definition or a feature placement rule is parsed, with the `chance` parameter being supplied directly from the configuration data.
- **Scope:** The lifetime of a RandomCoordinateCondition instance is tied to its owning rule or configuration object. It is a short-lived, throwaway object that persists only as long as the rule it defines is in scope.
- **Destruction:** Managed entirely by the Java Garbage Collector. When the parent generator or rule set is unloaded, the condition instance becomes eligible for collection. No explicit cleanup is necessary.

## Internal State & Concurrency
- **State:** Immutable. The internal `chance` field is declared as final and is initialized only once via the constructor. The object holds no other mutable state and performs no caching.
- **Thread Safety:** This class is inherently thread-safe. Because its state is immutable and its `eval` methods are pure functions (their output depends only on their inputs), it can be safely invoked by multiple world generation worker threads concurrently without any need for locks or synchronization. This design is critical for achieving high performance in parallel chunk generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(seed, x, y) | boolean | O(1) | Evaluates the condition for a 2D coordinate. Returns true if the deterministic random value is less than or equal to the configured chance. |
| eval(seed, x, y, z) | boolean | O(1) | Evaluates the condition for a 3D coordinate. Returns true if the deterministic random value is less than or equal to the configured chance. |

## Integration Patterns

### Standard Usage
This class is intended to be used by the world generation system, not directly by gameplay logic. A rule engine would instantiate it as part of a larger set of conditions to determine if a feature should be placed.

```java
// A procedural rule engine would use the condition as follows:
// This would typically be loaded from a data file.
double placementChance = 0.05; // 5% chance
ICoordinateCondition condition = new RandomCoordinateCondition(placementChance);

// During chunk generation for a specific block column...
boolean shouldPlaceFeature = condition.eval(worldSeed, blockX, blockY, blockZ);

if (shouldPlaceFeature) {
    // ... execute feature placement logic (e.g., spawn a tree)
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Modification:** Do not attempt to modify this class to hold mutable state. Its immutability is a core design feature that guarantees thread safety and predictable behavior.
- **Using Non-Deterministic Sources:** Never replace the call to HashUtil with a non-deterministic random source like java.util.Random. Doing so would violate the fundamental requirement of repeatable world generation, leading to inconsistent worlds from the same seed.
- **Invalid Chance Values:** While not enforced by the constructor, the `chance` value should always be within the range of 0.0 to 1.0. Values outside this range will result in trivial outcomes (always false for negative values, always true for values greater than 1.0), defeating the purpose of the condition.

## Data Pipeline
This component acts as a predicate or a gate within a larger data processing flow. It does not transform data but rather controls whether a subsequent step in the pipeline is executed.

> Flow:
> World Generation Rule -> **RandomCoordinateCondition.eval()** -> Boolean Result -> Conditional Feature Placer

