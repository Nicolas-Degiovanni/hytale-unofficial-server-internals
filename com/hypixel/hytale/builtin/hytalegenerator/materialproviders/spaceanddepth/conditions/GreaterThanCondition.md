---
description: Architectural reference for GreaterThanCondition
---

# GreaterThanCondition

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.conditions
**Type:** Transient

## Definition
```java
// Signature
public class GreaterThanCondition implements SpaceAndDepthMaterialProvider.Condition {
```

## Architecture & Concepts
The GreaterThanCondition is a fundamental building block within the procedural world generation system, specifically serving the SpaceAndDepthMaterialProvider. It functions as a stateless, single-purpose predicate object.

Its core responsibility is to answer a simple boolean question: "Is a specific environmental metric at a given block coordinate greater than a pre-configured value?" The environmental metric, such as *space above the floor* or *space below the ceiling*, is selected at configuration time.

These condition objects are not intended to be used in isolation. They are composed by a parent material provider to build a chain of rules that collectively determine which material should be placed at a specific location. This design allows for complex, data-driven generation logic without requiring new code for every rule variation.

### Lifecycle & Ownership
- **Creation:** Instances are typically not created manually in code. They are instantiated by the world generation framework during the deserialization of generator configuration assets (e.g., JSON files). The framework reads a threshold and a parameter type from the asset and constructs the object.
- **Scope:** The lifetime of a GreaterThanCondition is bound to its owning SpaceAndDepthMaterialProvider. It is created when the provider is loaded and persists in memory for the duration of the world generation process.
- **Destruction:** The object is eligible for garbage collection once its parent provider is unloaded. No explicit destruction or resource cleanup is necessary.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of the threshold and parameter fields, is established at construction and cannot be modified thereafter. Both fields are declared as final.
- **Thread Safety:** **Fully thread-safe**. Its immutable nature guarantees that a single instance can be safely evaluated by multiple world generation worker threads concurrently without risk of race conditions or data corruption. This is critical for the performance of a parallelized world generator.

## API Surface
The public contract is minimal, consisting of a single evaluation method defined by the Condition interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| qualifies(x, y, z, ...) | boolean | O(1) | Evaluates the condition against the provided world context. Returns true if the selected parameter's value is strictly greater than the internal threshold. |

## Integration Patterns

### Standard Usage
This class is an internal component of the world generator and is not intended for direct use in gameplay systems. Its primary consumer is a parent provider which iterates through a list of conditions to make a decision.

```java
// Hypothetical usage within a Material Provider
// Note: This is a conceptual example. Do not call this directly.

// 'conditions' would be a list of Condition objects, including GreaterThanCondition
// 'context' would be an object holding the integer parameters
boolean allConditionsMet = true;
for (Condition condition : conditions) {
    if (!condition.qualifies(context.x, context.y, context.z, ...)) {
        allConditionsMet = false;
        break;
    }
}

if (allConditionsMet) {
    // Place the material
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Avoid creating instances with `new GreaterThanCondition()`. World generation logic should be data-driven. Define conditions in generator asset files to be loaded by the engine.
- **Stateful Modification:** Do not attempt to alter this class to add mutable state. Its immutability is a core design principle that enables safe, parallel execution of the world generator. Introducing state would require complex locking and defeat this purpose.

## Data Pipeline
The GreaterThanCondition acts as a filter or gate within the material selection data flow. It consumes spatial context and produces a simple boolean signal.

> Flow:
> World Generator Voxel Iterator -> Spatial Context Sampler -> **GreaterThanCondition.qualifies(context)** -> Boolean Result -> Material Selection Logic

