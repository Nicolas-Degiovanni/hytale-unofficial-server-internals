---
description: Architectural reference for ActionApplyEntityEffect
---

# ActionApplyEntityEffect

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat
**Type:** Transient / Command Object

## Definition
```java
// Signature
public class ActionApplyEntityEffect extends ActionBase {
```

## Architecture & Concepts
The ActionApplyEntityEffect class is a concrete implementation of the Command Pattern, operating within the server-side NPC AI framework. It represents a single, atomic, and configurable behavior: applying a server-defined status effect to an entity.

This class acts as a bridge between the abstract decision-making layer of an NPC (such as a Behavior Tree or a finite-state machine within a Role) and the server's core Entity-Component-System (ECS). Its sole responsibility is to resolve a target entity and dispatch a request to that entity's EffectControllerComponent to add a new effect.

Instances of this class are not generic; they are configured during the server's asset loading phase. Each instance represents a specific action, such as "Apply Poison to Target" or "Apply Self-Heal". This data-driven design allows designers to create complex NPC behaviors without writing new code.

## Lifecycle & Ownership
- **Creation:** Instances are created by the NPC asset pipeline, specifically via a corresponding BuilderActionApplyEntityEffect. This process occurs when the server loads NPC definitions from configuration files. The builder reads parameters (e.g., the effect ID and target policy) and constructs an immutable ActionApplyEntityEffect object.
- **Scope:** The object's lifetime is bound to the NPC's loaded definition, typically held within a collection of actions inside a Role asset. It is a reusable command object that persists as long as its parent Role is in memory. It is **not** created on-the-fly for each execution.
- **Destruction:** The object is marked for garbage collection when the server unloads the associated NPC assets or when an NPC's Role is replaced or destroyed.

## Internal State & Concurrency
- **State:** Immutable. The internal fields `entityEffectId` and `useTarget` are final and are exclusively set during construction. This class holds no mutable state related to any specific execution; all contextual data required for execution is passed in via method parameters.
- **Thread Safety:** Conditionally thread-safe. As an immutable object, an instance can be safely referenced by multiple threads. However, the `execute` method is **not** thread-safe as it mutates the shared game state through the `Store<EntityStore>` parameter.

    **WARNING:** All methods on this class, especially `execute`, must be invoked exclusively from the main server thread that owns the corresponding game world instance. Off-thread execution will lead to critical data corruption and race conditions.

## API Surface
The public contract is defined by its parent, ActionBase, and is intended for use by the NPC Role and behavior systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Verifies if the action can be performed. Returns false if `useTarget` is true and the NPC has no valid target. |
| execute(...) | boolean | O(1) | Executes the action. Resolves the target and dispatches the effect application request to the target's EffectControllerComponent. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or invocation by game logic developers. It is a component consumed by the higher-level NPC AI system. The system evaluates a set of potential actions and invokes them during an NPC's update tick.

```java
// Conceptual example of how the NPC system uses this action.
// This code would exist within a Behavior Tree node or Role controller.

// Given an ActionBase instance which is an ActionApplyEntityEffect
ActionBase currentAction = npc.getActiveRole().getAction("apply_poison_touch");

if (currentAction.canExecute(npcRef, role, sensorInfo, dt, worldStore)) {
    currentAction.execute(npcRef, role, sensorInfo, dt, worldStore);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use a builder or constructor to create an instance of ActionApplyEntityEffect at runtime. These actions must be defined in data assets and loaded by the server to ensure the integrity of the AI system.
- **Stateful Misuse:** Do not attempt to extend this class to add state. It is designed as a stateless, reusable command. All execution-specific data must come from the `sensorInfo` or other method parameters.
- **Off-Thread Execution:** Never call `execute` from an asynchronous task or a different thread. All NPC behavior logic must be synchronized with the main server game loop.

## Data Pipeline
The execution of this action initiates a clear flow of data and control through several core server systems.

> Flow:
> NPC Behavior Controller -> **ActionApplyEntityEffect.execute()** -> SensorInfo (for target resolution) -> EntityStore (to get EffectControllerComponent) -> EffectControllerComponent.addEffect() -> Game State Mutation

