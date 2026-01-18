---
description: Architectural reference for AliveCondition
---

# AliveCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient

## Definition
```java
// Signature
public class AliveCondition extends Condition {
```

## Architecture & Concepts
The AliveCondition is a concrete implementation of the **Strategy Pattern** within the server's entity statistics and status effect system. It encapsulates a single, reusable piece of logic: determining if an entity is currently considered "alive".

This class acts as a bridge between the data-driven configuration of entity stats (often loaded from asset files) and the live state of the Entity Component System (ECS). Its primary function is to query the ECS for the absence of a specific marker component, the DeathComponent. If an entity's archetype does not contain the DeathComponent, the condition evaluates to true.

The behavior can be inverted during construction, allowing the same class to be used to check if an entity is dead. This design promotes reusability and avoids class proliferation for opposing logical states (e.g., an IsDeadCondition is not needed).

## Lifecycle & Ownership
- **Creation:** AliveCondition instances are primarily created by the asset deserialization pipeline using the provided static CODEC. They are instantiated when the server loads game assets that define rules, such as status effects or abilities which depend on the target's living state. They can also be instantiated programmatically for dynamic, in-memory rule creation.
- **Scope:** Extremely short-lived. An instance of AliveCondition is stateless and typically exists only for the duration of a single evaluation cycle within a larger system. It is invoked, its result is consumed, and it is then eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. As there are no external resources or persistent state, no explicit cleanup is required.

## Internal State & Concurrency
- **State:** Immutable. The only state held by an AliveCondition instance is the boolean `inverse` flag, inherited from its parent `Condition` class. This flag is set at construction and cannot be modified, making the object inherently stateless after creation.
- **Thread Safety:** This class is unconditionally thread-safe. It contains no mutable fields and its evaluation logic is a pure function relative to its own state. The overall thread safety of an operation involving AliveCondition depends on the guarantees provided by the ComponentAccessor and EntityStore passed into the `eval0` method, which are external dependencies.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(accessor, ref, time) | boolean | O(1) | Evaluates the condition against a target entity. Returns true if the entity's archetype does not contain DeathComponent. The result is inverted if the `inverse` flag was set to true at construction. |

## Integration Patterns

### Standard Usage
AliveCondition is not typically used in isolation. It is designed to be part of a collection of conditions that are evaluated by a higher-level system to determine if an action, such as applying a stat modifier or a status effect, should proceed.

```java
// A system evaluates if a stat modifier should apply to a target entity.
// The AliveCondition is typically loaded from an asset definition.

// For demonstration, we create it programmatically.
Condition isAlive = new AliveCondition(false); // Check if alive
Condition isDead = new AliveCondition(true);  // Check if dead

// In a real system, this would be the target entity's reference.
Ref<EntityStore> targetEntityRef = ...;
ComponentAccessor<EntityStore> accessor = ...;

// The system evaluates the condition
if (isAlive.eval0(accessor, targetEntityRef, Instant.now())) {
    // Apply healing effect, as the target is alive.
}
```

### Anti-Patterns (Do NOT do this)
- **Creating Redundant Classes:** Do not create a separate `IsDeadCondition` class. The `inverse` flag on AliveCondition is the correct and intended way to check for the opposite state. `new AliveCondition(true)` is the correct pattern for checking if an entity is dead.
- **Caching Results:** Do not attempt to store the result of an `eval0` call within a custom wrapper or extended class. The living state of an entity is highly volatile. Each check must be a fresh query against the live ECS state.

## Data Pipeline
The primary role of AliveCondition is to act as a gate in a control flow, not to process a stream of data. It produces a boolean value that influences the subsequent logic path.

> Flow:
> Stat Calculation Engine -> Retrieves list of conditions for a modifier -> **AliveCondition.eval0()** -> Boolean Result -> Engine applies or discards modifier based on result

