---
description: Architectural reference for NotCondition
---

# NotCondition

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.conditions
**Type:** Transient Utility

## Definition
```java
// Signature
public class NotCondition implements SpaceAndDepthMaterialProvider.Condition {
```

## Architecture & Concepts
The NotCondition class is a fundamental logical operator within the world generation framework, specifically for the SpaceAndDepthMaterialProvider. It embodies the Decorator pattern, wrapping a single child Condition and inverting its boolean outcome.

Its primary role is to provide expressive power to the declarative rule engine that governs material placement. By allowing the negation of a rule, architects can define complex placement logic such as "place this material only if it is *not* near water" or "generate this feature only when it is *not* exposed to the sky".

This class is a compositional building block, designed to be combined with other logical conditions like AndCondition and OrCondition to form a decision tree that is evaluated for each block position during chunk generation.

### Lifecycle & Ownership
-   **Creation:** NotCondition is instantiated during the parsing of world generation configuration files or when a procedural rule set is constructed in memory. It is almost never created directly in gameplay logic, but rather as part of a larger, static definition for a biome or world feature.
-   **Scope:** The object's lifetime is bound to its containing rule set, which is typically owned by a SpaceAndDepthMaterialProvider. It is effectively a transient configuration object, used and discarded for each generation task.
-   **Destruction:** As a simple, stateless object, it holds no unmanaged resources. It is reclaimed by the Java garbage collector once the parent provider and its associated rule tree are no longer in scope.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal reference to the wrapped Condition is final and assigned at construction. The object itself holds no mutable state, making its behavior entirely deterministic based on its initial configuration and input parameters.
-   **Thread Safety:** **Fully thread-safe**. Its immutable nature is a critical design choice that allows instances to be safely shared and executed across multiple world generation worker threads without any synchronization or locking. This is essential for achieving high-performance, parallelized chunk generation.

## API Surface
The public contract is minimal, consisting only of the interface method it implements.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| qualifies(int, int, int, int, int, int, int) | boolean | O(C) | Evaluates the wrapped condition for the given world coordinates and spatial context, returning the logical inverse. Complexity is determined by the wrapped condition C. |

## Integration Patterns

### Standard Usage
NotCondition is never used in isolation. It is composed with other conditions to build a complete logical statement for the material provider.

```java
// Example: Define a rule to place material only when NOT deep underground.
// This is a conceptual example of how rule sets are built.

// A condition that is true when a block is deep underground
Condition deepUnderground = new DepthIntoFloorCondition(10);

// The NotCondition inverts this logic
Condition notDeepUnderground = new NotCondition(deepUnderground);

// This 'notDeepUnderground' condition would then be passed to a material provider.
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Composition:** Do not wrap a Condition that contains mutable state. The world generator provides no guarantee on the order or timing of evaluation. A stateful nested condition can lead to non-deterministic generation, visual artifacts, and chunk border seams.
-   **Null Wrapping:** Constructing a NotCondition with a null inner condition will result in a NullPointerException at runtime. The configuration loader and rule builders must enforce non-null constraints.

## Data Pipeline
NotCondition acts as a simple transformation node within the material selection data flow. It does not source or sink data, but rather alters the logical flow based on its input.

> Flow:
> World Generator provides coordinates & context -> SpaceAndDepthMaterialProvider evaluates its rule tree -> **NotCondition** receives a boolean from its child -> Inverts the boolean -> Returns the result up the tree -> Final decision determines material placement.

