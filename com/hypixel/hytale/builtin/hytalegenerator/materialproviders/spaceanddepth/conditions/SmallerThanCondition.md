---
description: Architectural reference for SmallerThanCondition
---

# SmallerThanCondition

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.conditions
**Type:** Transient

## Definition
```java
// Signature
public class SmallerThanCondition implements SpaceAndDepthMaterialProvider.Condition {
```

## Architecture & Concepts
The SmallerThanCondition class is a predicate component within the procedural world generation framework, specifically serving the SpaceAndDepthMaterialProvider. Its sole responsibility is to evaluate a single, simple boolean condition: whether a specific spatial metric at a given voxel coordinate is less than a pre-configured threshold.

This class embodies the Strategy pattern, where it represents one of many possible conditions that a material provider can use to make decisions. It is a fundamental building block, designed to be composed with other conditions to create complex, data-driven rules for material placement. For example, a rule might state "place granite if the space above the floor is *smaller than* 3 blocks". This class encapsulates the "smaller than 3 blocks" logic.

These objects are not intended to be long-lived, stateful services. They are lightweight, immutable value objects that represent a single piece of logic in a larger decision tree.

### Lifecycle & Ownership
- **Creation:** Instances are created by a parent SpaceAndDepthMaterialProvider, typically during the deserialization of world generation asset files. They are not managed by a central registry or dependency injection container.
- **Scope:** The lifetime of a SmallerThanCondition instance is strictly bound to the configuration of its parent material provider. It persists only as long as that configuration is held in memory.
- **Destruction:** The object is eligible for garbage collection as soon as its parent provider's configuration is discarded. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of the `threshold` and `parameter` fields, is established at construction and cannot be modified thereafter. The class performs no caching and its behavior is fully deterministic.
- **Thread Safety:** **Thread-Safe**. Due to its immutable nature and the absence of side effects in its evaluation logic, a single instance can be safely accessed and executed by multiple world generation threads concurrently without any need for external synchronization or locking.

## API Surface
The public contract is minimal, focused entirely on the evaluation of the condition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| qualifies(x, y, z, depthIntoFloor, depthIntoCeiling, spaceAboveFloor, spaceBelowCeiling) | boolean | O(1) | Evaluates if the contextual parameter specified at construction is less than the configured threshold. This is a pure function. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation in gameplay or system logic. It is designed to be used declaratively as part of a SpaceAndDepthMaterialProvider configuration, which is then invoked by the world generator.

```java
// Conceptual usage within the parent provider
// A developer would NOT write this code, but this is how the system uses the class.

// 1. Provider is configured, creating the condition instance
ConditionParameter param = ConditionParameter.SPACE_ABOVE_FLOOR;
int threshold = 4;
SpaceAndDepthMaterialProvider.Condition condition = new SmallerThanCondition(threshold, param);

// 2. During world generation, the provider evaluates the condition for a specific block
boolean result = condition.qualifies(120, 64, -300, 5, 0, 3, 10);

// result is true, because spaceAboveFloor (3) < threshold (4)
if (result) {
    // ...logic to place a specific material
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Logic:** Do not attempt to modify this class to hold mutable state or depend on external services. Its power lies in its simplicity and immutability.
- **Manual Chaining:** Avoid writing complex Java code that manually chains multiple condition objects together. The material provider system is designed to process a declarative list of conditions defined in asset files. Manual composition bypasses this system and leads to brittle, hard-to-maintain code.

## Data Pipeline
The data flow is unidirectional, from the high-level world generator context down to a simple boolean result.

> Flow:
> World Generator Voxel Context -> SpaceAndDepthMaterialProvider -> **SmallerThanCondition.qualifies()** -> Boolean Result -> Material Selection Logic

