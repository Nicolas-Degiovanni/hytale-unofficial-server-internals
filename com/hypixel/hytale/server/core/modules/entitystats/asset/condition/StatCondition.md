---
description: Architectural reference for StatCondition
---

# StatCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient Data Model

## Definition
```java
// Signature
public class StatCondition extends EntityStatBoundCondition {
```

## Architecture & Concepts
The StatCondition class is a fundamental data model within the server's entity statistics module. It represents a single, evaluatable rule that compares an entity's statistic against a predefined value. For example, a StatCondition can represent the rule "Is the entity's Health greater than or equal to 50?".

This class is not a service or a manager; it is a passive data structure designed for configuration. Its primary role is to be deserialized from game asset files (e.g., JSON definitions for mob behavior, item effects, or quest objectives). The static **CODEC** field is the key to this architecture, providing the Hytale serialization framework with the instructions to construct a StatCondition object from raw data.

It extends **EntityStatBoundCondition**, inheriting the mechanism to bind this condition to a specific entity statistic via an integer ID. The core logic it adds is the comparison operator (e.g., Greater Than, Equal) and the value to compare against.

## Lifecycle & Ownership
- **Creation:** StatCondition instances are almost exclusively created by the Hytale asset loading and codec system during server startup or when game assets are loaded. The public static **CODEC** field is the designated entry point for instantiation from data. Manual instantiation via its constructor is possible but generally discouraged as it bypasses the asset pipeline.

- **Scope:** The lifetime of a StatCondition object is tied to the lifecycle of the asset that defines it. It is a configuration object, not a live game state object. It persists in memory as part of a larger configuration (e.g., a monster's ability set) and is reused for evaluations across many different entity instances.

- **Destruction:** Instances are managed by the Java garbage collector. They are eligible for cleanup once the containing asset configuration is unloaded and no longer referenced. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The state of a StatCondition consists of the statistic ID it targets (inherited), the comparison type, and the comparison amount. This state is populated during deserialization and should be considered **effectively immutable** afterward. The class provides no public setters, enforcing a read-only pattern after initialization.

- **Thread Safety:** The evaluation method, **eval0**, is stateless and performs a pure function based on its internal configuration and the provided arguments. Therefore, a single StatCondition instance is **thread-safe** and can be safely evaluated by multiple threads concurrently without locks or synchronization.

## API Surface
The public contract is minimal, focusing entirely on evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(Ref, Instant, EntityStatValue) | boolean | O(1) | Evaluates the condition against a given stat value. Returns true if the condition is met. This is the primary operational method. |

## Integration Patterns

### Standard Usage
StatCondition is not intended to be used directly in typical game logic. Instead, it is defined in data files and invoked by systems that process these configurations.

```java
// A hypothetical system that evaluates conditions for a given entity.
// This StatCondition instance would have been loaded from an asset file.

public boolean checkAttackPreconditions(Entity entity, StatCondition healthCondition) {
    // Retrieve the live stat value for the entity
    EntityStatValue currentHealth = entity.getStatManager().getStatValue(healthCondition.getStat());

    // The system calls eval0 to check if the condition is met
    Ref<EntityStore> storeRef = ...; // Obtain reference to the EntityStore
    Instant now = Instant.now();
    
    return healthCondition.eval0(storeRef, now, currentHealth);
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not attempt to modify the internal state of a StatCondition object after it has been loaded. It is designed as a static configuration blueprint.

- **Manual Instantiation:** Avoid using `new StatCondition()`. This bypasses the asset pipeline and the robust deserialization and validation provided by the **CODEC** system. All conditions should be data-driven.

## Data Pipeline
The flow of data for a StatCondition begins in an asset file and ends with a boolean result that influences server logic.

> Flow:
> Game Asset (e.g., JSON) -> Server Asset Loader -> **StatCondition.CODEC** -> **StatCondition Instance** -> Condition Evaluation System -> `eval0()` -> Boolean Result -> Game Logic (e.g., AI Decision, Ability Trigger)

