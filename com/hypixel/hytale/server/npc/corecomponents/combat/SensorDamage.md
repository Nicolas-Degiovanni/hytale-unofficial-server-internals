---
description: Architectural reference for SensorDamage
---

# SensorDamage

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorDamage extends SensorBase {
```

## Architecture & Concepts
The SensorDamage class is a predicate component within the server-side NPC Artificial Intelligence framework. It functions as a sensory input mechanism, allowing an NPC to detect and react to various types of incoming damage. Its primary role is to answer the question: "Have I recently taken a type of damage that I care about?"

This component acts as a bridge between the low-level damage system, which records damage events in an NPCEntity's DamageData component, and the high-level behavior system, such as a Behavior Tree or Finite State Machine. By evaluating the recent damage history against its configured filters (e.g., only combat damage, ignore friendly fire), it provides a simple boolean signal that drives AI decision-making.

A SensorDamage instance is highly configurable through its corresponding builder, allowing designers to create nuanced NPC reactions declaratively within asset files. For example, one NPC might use this sensor to flee from environmental damage, while another uses it to retaliate against a specific attacker.

## Lifecycle & Ownership
- **Creation:** An instance of SensorDamage is not created directly. It is instantiated by the BuilderSensorDamage during the server's asset loading phase when an NPC's behavior graph is constructed. Each NPC with this sensor in its definition will have its own unique instance.
- **Scope:** The lifecycle of a SensorDamage object is tightly coupled to the lifecycle of the parent NPCEntity to which it belongs. It persists as long as the NPC is active in the world.
- **Destruction:** The object is eligible for garbage collection when its parent NPCEntity is unloaded or destroyed. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** SensorDamage is a **stateful** component. Its primary internal state is managed by the EntityPositionProvider field, which caches the reference and location of the last detected damage source. This state is volatile and is typically cleared or updated on every evaluation of the matches method. The component's filtering behavior (e.g., combatDamage, friendlyDamage) is configured at creation and is immutable for the lifetime of the object.

- **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. The Hytale server's NPC AI system operates on a single-threaded model per world or region. All interactions with this component, especially calls to the matches method, are expected to occur exclusively on the main server thread for that NPC. Unsynchronized access would lead to race conditions and unpredictable behavior due to the mutable state of the positionProvider.

## API Surface
The public contract is minimal, designed for integration with the internal NPC behavior loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates if the NPC has recently suffered damage matching the sensor's configuration. This is the core logic method. |
| getSensorInfo() | InfoProvider | O(1) | Returns the internal provider that holds information about the damage source, such as its entity reference and position. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in code. Instead, it is defined declaratively as part of an NPC's behavior asset. The game engine's AI scheduler is responsible for invoking the matches method during the NPC's update tick.

A behavior node, such as a decorator in a Behavior Tree, would be configured to use this sensor as a condition.

```java
// Conceptual Example: How the engine uses the sensor
// This code is internal to the AI scheduler and not written by users.

// Inside an NPC's AI update loop...
boolean wasDamaged = sensorDamageInstance.matches(npcRef, npcRole, deltaTime, worldStore);

if (wasDamaged) {
    // The Behavior Tree can now react.
    // For example, it might retrieve the attacker's location.
    InfoProvider info = sensorDamageInstance.getSensorInfo();
    
    // Transition to a "retaliate" or "flee" state.
    behaviorTree.handleEvent("OnDamageTaken", info);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new SensorDamage(). The object's configuration is complex and must be handled by the asset loading pipeline via its corresponding builder. Direct instantiation will result in a misconfigured and non-functional component.
- **State Mismanagement:** Do not call getSensorInfo and expect valid data without a preceding, successful call to matches within the same AI tick. The internal state is only guaranteed to be valid immediately after matches returns true.
- **External Invocation:** Do not call the matches method from outside the designated NPC AI update loop. Doing so can disrupt the component's state and cause unpredictable AI behavior.

## Data Pipeline
The flow of information through this component is linear and reactive, triggered by an in-game damage event.

> Flow:
> External Damage Event (e.g., Player Sword Swing) -> Server Damage Module -> **NPCEntity.DamageData** component is updated -> NPC AI Tick begins -> Behavior System invokes **SensorDamage.matches()** -> Sensor reads from DamageData -> If true, internal **EntityPositionProvider** is updated -> Boolean result is returned to Behavior Tree -> AI state transition occurs.

