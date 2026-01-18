---
description: Architectural reference for OrCondition
---

# OrCondition

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.conditions
**Type:** Transient

## Definition
```java
// Signature
public class OrCondition implements SpaceAndDepthMaterialProvider.Condition {
```

## Architecture & Concepts
The OrCondition class is a foundational component of the procedural world generation system, specifically within the `SpaceAndDepthMaterialProvider` framework. It functions as a **Composite Strategy** object, designed to aggregate multiple `Condition` implementations and evaluate them using boolean OR logic.

Its primary role is to enable the creation of complex, layered rules for material placement without requiring new, monolithic condition classes. By nesting `Condition` objects within an OrCondition, world designers can declaratively specify that a material should be placed if *any* one of a set of criteria is met. For example, a rule might state "place mossy stone if the block is near water OR if the block is below a certain depth". This class is the logical operator that facilitates the "OR" part of such a rule.

This component is a pure data-driven evaluator. It holds no gameplay state and acts as a stateless decision point in the material selection pipeline.

### Lifecycle & Ownership
- **Creation:** OrCondition instances are not intended for manual instantiation by gameplay programmers. They are created by the world generation asset loader during the deserialization of generator configuration files (e.g., JSON definitions). The framework reads a list of condition definitions and constructs this object to group them.
- **Scope:** The lifetime of an OrCondition is bound to its parent `SpaceAndDepthMaterialProvider`. It is effectively an immutable configuration object that persists as long as the parent provider is loaded in memory.
- **Destruction:** The object is marked for garbage collection when the world generator configuration is unloaded or reloaded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** Immutable. The internal `conditions` array is populated once in the constructor from the provided list and is declared final. The contents of this array are never modified post-construction. The behavior of the `qualifies` method is therefore deterministic based on its initial configuration and input arguments.
- **Thread Safety:** This class is inherently thread-safe. Its immutable state guarantees that it can be safely accessed by multiple world generation worker threads simultaneously without locks or other synchronization primitives. This is critical for the performance of the parallelized world generation engine.

## API Surface
The public contract is minimal, consisting only of the constructor and the interface method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| qualifies(x, y, z, ...) | boolean | O(N) | Evaluates the contained conditions. Returns true if any child condition returns true. Evaluation is short-circuited; it stops and returns immediately upon the first success. |

## Integration Patterns

### Standard Usage
This class is not used imperatively. It is defined declaratively within world generation asset files. The system's asset loader instantiates it, and the `SpaceAndDepthMaterialProvider` invokes it during its evaluation process.

A developer would never write the following code, but this is how the *system* uses the class after loading it from a configuration:

```java
// Conceptual example of how the generator invokes a loaded condition.
// This code would exist deep within the SpaceAndDepthMaterialProvider.

// 'loadedCondition' would be an instance of OrCondition deserialized from a file.
SpaceAndDepthMaterialProvider.Condition loadedCondition = ...;

// The generator checks if the condition passes for a specific block coordinate.
boolean result = loadedCondition.qualifies(100, 50, 200, 5, 10, 15, 2);

if (result) {
    // Place the associated material...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid `new OrCondition()` in dynamic game code. World generation rules should be treated as static data. Modifying these rules at runtime is an unsupported pattern and can lead to unpredictable generation results.
- **Empty Condition List:** Constructing an OrCondition with an empty list of sub-conditions is valid but logically useless. The `qualifies` method will always return false in this case. Configurations should be audited to prevent this.
- **Null Child Conditions:** The constructor performs strict validation and will throw an `IllegalArgumentException` if the input list contains any null elements. This is a fatal configuration error.

## Data Pipeline
OrCondition acts as a conditional gate within the material selection data flow. It does not transform data but rather produces a boolean decision that directs the subsequent steps of the pipeline.

> **Flow:**
> World Generator queries block at (X, Y, Z) -> `SpaceAndDepthMaterialProvider` receives query -> Provider invokes its root `Condition` -> **OrCondition.qualifies(...)** evaluates child conditions -> Boolean `true`/`false` is returned -> Provider uses result to select a material or continue evaluation.

