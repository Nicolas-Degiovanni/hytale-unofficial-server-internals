---
description: Architectural reference for SprintingCondition
---

# SprintingCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient / Data Object

## Definition
```java
// Signature
public class SprintingCondition extends Condition {
```

## Architecture & Concepts
The SprintingCondition is a concrete implementation of the **Condition** strategy pattern. It serves as a predicate to evaluate a single, specific piece of an entity's state: whether it is currently sprinting.

This class is a fundamental building block of the server's data-driven logic systems, particularly for entity stats and behaviors. Its primary role is to enable game designers to create complex rules in external asset files (e.g., JSON) without writing Java code. The static CODEC field facilitates the serialization and deserialization of this condition, allowing it to be embedded within definitions for items, abilities, or AI behaviors.

Architecturally, SprintingCondition operates as a pure, read-only function within the Entity Component System (ECS) paradigm. It queries the state of a MovementStatesComponent but never modifies it, ensuring predictable and side-effect-free evaluation.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the asset loading pipeline via the provided BuilderCodec. Game designers define this condition declaratively in asset files, which are then deserialized into SprintingCondition objects at server startup or during a resource reload.
- **Scope:** The object's lifetime is bound to the parent asset that contains it. It is effectively an immutable data structure once loaded and persists as long as its containing asset definition is held in memory.
- **Destruction:** The object is marked for garbage collection when the parent asset is unloaded from the game. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** This class is stateless regarding the evaluation process. Its only internal state is the *inverse* boolean inherited from the base Condition class, which is set at creation time and is immutable thereafter. The `eval0` method is a pure function of its inputs.
- **Thread Safety:** The `eval0` method is conditionally thread-safe. It performs a read-only lookup on an entity's component. The safety of concurrent execution depends entirely on the thread-safety guarantees of the underlying ComponentAccessor and EntityStore. In practice, condition evaluation is expected to occur on the main server tick thread to prevent race conditions with systems that modify component state, such as the movement system.

## API Surface
The primary contract is the `eval0` method, which is an internal implementation detail of the `Condition` interface's public `eval` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(accessor, ref, time) | boolean | O(1) | Checks if the entity identified by `ref` has its sprinting state set to true in its MovementStatesComponent. Throws AssertionError if the component is missing. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most systems. Instead, higher-level systems (e.g., a stat modifier system) will operate on a collection of generic Condition objects. These systems iterate and evaluate their conditions for a given entity to determine if a specific game mechanic should be applied.

```java
// A hypothetical system evaluating a list of conditions for an entity
// The system does not know or care that one of them is a SprintingCondition

for (Condition condition : statModifier.getConditions()) {
    if (condition.eval(componentAccessor, entityRef, now)) {
        // Apply stat modifier because the entity is sprinting
        applyModifier(entity);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SprintingCondition()`. This bypasses the data-driven design and hardcodes game logic. All conditions should be defined in asset files and loaded via the CODEC.
- **Incorrect Context:** Evaluating this condition on an entity that does not possess a MovementStatesComponent is a logical error and will result in a server-side assertion failure. This condition should only be used in contexts where the target entity is guaranteed to be a mobile entity.

## Data Pipeline
SprintingCondition acts as a gate in a data pipeline, transforming an entity reference into a boolean decision.

> Flow:
> Stat Application System -> `condition.eval(entityRef)` -> **SprintingCondition.eval0** -> ComponentAccessor lookup -> MovementStatesComponent.sprinting -> `boolean` result -> System applies or ignores effect

