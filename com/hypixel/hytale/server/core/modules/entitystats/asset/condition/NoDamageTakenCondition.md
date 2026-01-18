---
description: Architectural reference for NoDamageTakenCondition
---

# NoDamageTakenCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient / Data Object

## Definition
```java
// Signature
public class NoDamageTakenCondition extends Condition {
```

## Architecture & Concepts
The NoDamageTakenCondition is a specific implementation of the base Condition class, designed to evaluate a single, time-based predicate within the server's game logic. It represents the rule: "has this entity avoided all damage for a specified duration?".

This class is a fundamental building block in Hytale's data-driven design philosophy. It is not intended for direct programmatic instantiation but is instead deserialized from asset files using its static CODEC field. This allows game designers to define complex behaviors, such as status effect triggers, AI state transitions, or achievement criteria, without writing Java code.

Architecturally, it functions as a stateless evaluator within an Entity-Component-System (ECS) framework. It queries the state of an entity's DamageDataComponent to make its determination, acting as part of the "System" logic that operates on "Component" data.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale codec system during server startup or when game assets are loaded. The public CODEC field defines the schema for deserializing this object from a configuration file (e.g., JSON). The protected default constructor exists solely for use by the codec.
- **Scope:** The lifetime of a NoDamageTakenCondition instance is tied to the asset that defines it. It is effectively an immutable configuration object that persists as long as its parent asset is held in memory by the server.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup once the asset registry unloads the configuration that contains it. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The internal state consists of a single field, **delay**, of type Duration. This value is set once during deserialization and is never modified thereafter, making the object effectively immutable post-creation.
- **Thread Safety:** This class is inherently thread-safe due to its immutable state. However, the methods it interacts with, specifically ComponentAccessor and the underlying EntityStore, are **not** thread-safe.

    **WARNING:** The eval0 method must only be invoked from the main server thread that has exclusive write access to the entity component data for the current game tick. Calling it from an asynchronous task or a different thread will lead to race conditions, data corruption, or concurrency exceptions.

## API Surface
The primary contract is defined by the overridden `eval0` method from its parent `Condition` class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(accessor, ref, currentTime) | boolean | O(1) | Evaluates if the entity has taken damage within the configured delay. Asserts that the entity has a DamageDataComponent. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly. It is automatically invoked by a higher-level condition evaluation engine which processes lists of Condition objects.

A hypothetical game system would retrieve a list of conditions from an asset and evaluate them in a loop.

```java
// Hypothetical system evaluating a list of conditions for an entity
void checkEntityConditions(Ref<EntityStore> entityRef) {
    // asset is some game object definition loaded from a file
    List<Condition> conditions = asset.getActivationConditions();
    Instant now = world.getCurrentTime();

    for (Condition condition : conditions) {
        // The engine calls eval0 polymorphically.
        // If 'condition' is a NoDamageTakenCondition, its logic is executed here.
        if (condition.eval0(componentAccessor, entityRef, now)) {
            // ... trigger some game logic
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new NoDamageTakenCondition()`. The object will be in an invalid state as its `delay` field will be null, leading to a NullPointerException during evaluation. Always define conditions in data files to be loaded by the codec system.
- **Evaluation on Invalid Entities:** The `eval0` method includes a non-null assertion for the DamageDataComponent. Attempting to evaluate this condition on an entity that lacks this component will cause a server crash via an AssertionError. This is by design to enforce data integrity.

## Data Pipeline
The class functions as a node in two distinct pipelines: an asset loading pipeline and a gameplay evaluation pipeline.

**1. Asset Loading Pipeline:**
> Flow:
> Game Asset File (e.g., JSON) -> Hytale Codec Engine -> **NoDamageTakenCondition Instance** (In-memory representation)

**2. Gameplay Evaluation Pipeline:**
> Flow:
> Game Tick -> Condition Evaluation System -> `eval0` call -> **NoDamageTakenCondition** reads `DamageDataComponent` -> Returns `boolean` result -> Game Logic Branch

