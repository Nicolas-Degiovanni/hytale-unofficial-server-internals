---
description: Architectural reference for GlidingCondition
---

# GlidingCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient Data Object

## Definition
```java
// Signature
public class GlidingCondition extends Condition {
```

## Architecture & Concepts
The GlidingCondition is a specific, data-driven predicate used within the server-side Entity Stats and Status Effect systems. It is a concrete implementation of the Strategy pattern, where the base Condition class defines the contract for evaluating a specific state of an entity.

Its primary role is to determine if an entity is currently in the *gliding* state. This class is not intended for direct, imperative use by developers. Instead, it is designed to be defined within external asset files (e.g., JSON configurations for abilities, items, or effects) and deserialized at runtime via its static CODEC. This architecture allows game designers to construct complex, conditional logic without modifying engine source code.

For example, a designer could define a "Wind Rider" buff that grants a speed boost, but only when the `GlidingCondition` evaluates to true. The condition acts as a bridge between the high-level game logic (stat modification) and the low-level entity state (the `MovementStatesComponent`).

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale codec system during asset loading. The static `CODEC` field is used by a higher-level manager to deserialize a block of configuration data into a `GlidingCondition` object. Manual instantiation is strongly discouraged.
- **Scope:** An instance of GlidingCondition is typically short-lived and stateless. It is owned by a parent configuration object, such as a StatModifier or a StatusEffect. Its lifecycle is tied to its owner; it persists as part of an in-memory asset definition for the server's duration. The evaluation itself is a transient operation.
- **Destruction:** Managed by the Java Garbage Collector when the server shuts down or when asset definitions are reloaded.

## Internal State & Concurrency
- **State:** This object is effectively **immutable**. Its only state is the `inverse` boolean inherited from the base `Condition` class, which is set at construction and cannot be changed. It performs a fresh lookup on every evaluation and does not cache any entity state.
- **Thread Safety:** The class itself is thread-safe due to its immutable nature. However, the `eval0` method operates on data from the Entity Component System. All calls to `eval0` **must** be synchronized with the main server world tick to prevent race conditions when accessing component data. The ECS framework is responsible for ensuring safe access, and this class assumes it is being called from a valid execution context (i.e., the main game loop thread).

## API Surface
The public API is minimal, as interaction is primarily driven by the condition evaluation engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| GlidingCondition(boolean inverse) | constructor | O(1) | Creates a condition. The `inverse` flag will cause the evaluation to return true when the entity is *not* gliding. |
| eval0(...) | boolean | O(1) | **Core Logic.** Evaluates the gliding state of the target entity. Throws an exception if the entity lacks a `MovementStatesComponent`. |

## Integration Patterns

### Standard Usage
This class is not used imperatively. It is defined declaratively in an asset file that describes a game mechanic. The engine's condition processor invokes its `eval0` method internally.

A game designer would define it in a JSON asset like this:
```json
// Example: Part of a stat modifier asset
{
  "id": "mod.wind_rider.speed",
  "conditions": [
    {
      "type": "GlidingCondition",
      "inverse": false
    }
  ],
  "effects": [
    // ... effects to apply if gliding
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new GlidingCondition()`. This bypasses the intended data-driven design. Game logic should be configured in assets, not hardcoded.
- **Incorrect Context:** Do not attempt to create and use this class outside of the server's entity processing loop. It is tightly coupled to the server-side `ComponentAccessor` and `EntityStore`, and will fail in any other environment.
- **Missing Component Assumption:** The logic asserts that a `MovementStatesComponent` exists on the entity being evaluated. Attempting to evaluate this condition on an entity that cannot move or glide will result in a server-side error.

## Data Pipeline
GlidingCondition acts as a gate in a data evaluation flow. It does not transform data but rather produces a boolean signal that controls a subsequent process.

> Flow:
> Stat Calculation Trigger -> Condition Evaluation Engine -> **GlidingCondition.eval0** -> Accesses `MovementStatesComponent` -> Returns `boolean` -> Engine permits/denies stat modification

