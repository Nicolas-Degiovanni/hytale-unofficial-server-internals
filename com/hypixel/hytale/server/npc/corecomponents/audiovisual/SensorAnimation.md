---
description: Architectural reference for SensorAnimation
---

# SensorAnimation

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual
**Type:** Transient

## Definition
```java
// Signature
public class SensorAnimation extends SensorBase {
```

## Architecture & Concepts
The **SensorAnimation** class is a specialized component within the server-side NPC sensory system. It functions as a declarative condition, designed to answer a single, specific question: "Is this NPC currently playing a specific animation in a designated slot?".

This class acts as a bridge between the abstract logic of an NPC's Behavior Tree or Finite State Machine and the concrete state of the entity in the game world. It directly queries the low-level **ActiveAnimationComponent** to determine an entity's current visual state.

Architecturally, it is a leaf node in the NPC's sensory apparatus. It is not intended to be instantiated dynamically during gameplay. Instead, instances of **SensorAnimation** are defined in static NPC configuration files (e.g., JSON or HOCON assets) and are instantiated by a builder pipeline when the server loads its NPC archetypes. This design ensures that AI behavior is data-driven and decoupled from imperative code.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the **BuilderSensorAnimation** during the server's asset loading phase. The builder pattern populates its immutable state from parsed NPC definition files. Direct instantiation is a critical anti-pattern.
- **Scope:** The object's lifetime is tied to its parent NPC archetype or **Role** definition. It persists in memory as long as that NPC type is loaded and available for spawning. It is not tied to a specific NPC entity instance but is shared across all instances of that archetype.
- **Destruction:** The object is marked for garbage collection when its parent **Role** or NPC definition is unloaded from memory, for example, during a zone change or server shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of the **NPCAnimationSlot** and the **animationId**, is set once at construction via its `final` fields. The object itself is stateless with respect to gameplay; its purpose is to evaluate the state of other objects.
- **Thread Safety:** **Thread-safe and reentrant**. The **SensorAnimation** object contains no mutable state. Its primary method, `matches`, is a pure function with respect to the object itself, operating only on the arguments provided to it. Concurrency safety of an entire AI tick depends on the guarantees provided by the **Store<EntityStore>** implementation, but this class introduces no race conditions or synchronization requirements.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates if the target entity is playing the configured animation. Returns false if the entity lacks an **ActiveAnimationComponent**. |
| getSensorInfo() | InfoProvider | O(1) | Returns metadata for debugging. The current implementation returns null. |

## Integration Patterns

### Standard Usage
This sensor is not invoked directly. It is registered as part of an NPC's **Role** definition and is evaluated automatically by the core AI update loop each tick. The system iterates through sensors to build a complete picture of the world state before making a behavioral decision.

A conceptual example of how the system uses it:
```java
// This logic exists within the NPC's core AI processor, not in user code.
boolean isAttacking = someRole.getAttackSensor().matches(entityRef, role, dt, store);

if (isAttacking) {
    // Prevent starting a new action, as the attack animation is still active.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SensorAnimation()`. The constructor requires internal builder objects that are only available during the asset loading pipeline. All sensors must be defined in data files.
- **Stateful Subclassing:** Do not extend this class to add mutable state. The sensory system is designed around immutable, declarative components. Adding state will break thread-safety assumptions and lead to unpredictable behavior.
- **Misconfiguration:** The `assert activeAnimationComponent != null;` line implies a strict contract. Any entity evaluated by this sensor *must* have an **ActiveAnimationComponent**. Failure to ensure this will result in an **AssertionError** in development environments.

## Data Pipeline
The **SensorAnimation** acts as a read-only gate in the NPC's data-flow. It does not transform data; it validates it.

> Flow:
> NPC AI Tick -> Behavior Tree Node Evaluation -> **SensorAnimation.matches()** -> Read **EntityStore** -> Access **ActiveAnimationComponent** -> Boolean Result

