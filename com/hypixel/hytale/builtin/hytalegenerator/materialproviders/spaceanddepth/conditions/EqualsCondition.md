---
description: Architectural reference for EqualsCondition
---

# EqualsCondition

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.conditions
**Type:** Transient

## Definition
```java
// Signature
public class EqualsCondition implements SpaceAndDepthMaterialProvider.Condition {
```

## Architecture & Concepts
The EqualsCondition is a specific, stateless implementation of the Condition interface, designed to function as a predicate within the world generation pipeline. It embodies the Specification design pattern, encapsulating a single, atomic rule: "Does a specific contextual value equal a predefined constant?"

This component is a fundamental building block for the SpaceAndDepthMaterialProvider. The provider evaluates a collection of these Condition objects to make complex decisions about which material to place at a given coordinate. By composing simple, immutable conditions like EqualsCondition, the world generator can define intricate material placement logic in a declarative, robust, and highly parallelizable manner. Its sole responsibility is to return a boolean result based on the world-space context provided to it.

### Lifecycle & Ownership
-   **Creation:** Instances are created during the parsing of world generation configuration files. The SpaceAndDepthMaterialProvider instantiates this class, providing the target value and parameter from the configuration data. It is not intended for manual instantiation during the game loop.
-   **Scope:** The object's lifetime is bound to its parent SpaceAndDepthMaterialProvider. It is effectively a configuration value object.
-   **Destruction:** The object is marked for garbage collection when its parent provider is unloaded. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal fields *value* and *parameter* are declared as final and are set exclusively at construction time. The object's state cannot be modified after it is created.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, an instance of EqualsCondition can be safely read and executed by multiple world generation threads concurrently without any need for locks or synchronization primitives. This is a critical architectural feature for ensuring high-performance, parallel world generation.

## API Surface
The public contract is minimal, consisting of the constructor and the interface method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| qualifies(int...) | boolean | O(1) | Evaluates the condition against the provided world-space context. Returns true if the selected context parameter equals the configured value. |

## Integration Patterns

### Standard Usage
This class is not typically used directly. It is configured as part of a larger material provider definition, which is then consumed by the world generator. A programmatic example would involve constructing it and adding it to a provider's rule set.

```java
// This logic typically resides within a configuration loader.
// Assume 'provider' is an instance of SpaceAndDepthMaterialProvider.

// Rule: Qualifies if the space above the floor is exactly 3 blocks.
Condition condition = new EqualsCondition(3, ConditionParameter.SPACE_ABOVE_FLOOR);

// The provider will later invoke condition.qualifies(...) for each block.
provider.addCondition(condition);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Subclassing:** Do not extend this class to add mutable state. This violates its core design contract of immutability and will break thread safety, leading to severe and difficult-to-diagnose concurrency bugs in the world generator.
-   **Misconfiguration:** Providing a ConditionParameter that does not logically apply to an equality check can lead to nonsensical world generation rules. The system relies on the configuration author to provide meaningful rules.

## Data Pipeline
The data flow for this component is a simple, one-way evaluation. It receives contextual data from the generator, processes it, and returns a boolean result.

> Flow:
> World Generator Voxel Loop -> SpaceAndDepthMaterialProvider -> **EqualsCondition.qualifies(context)** -> Boolean Result -> Material Selection Logic

