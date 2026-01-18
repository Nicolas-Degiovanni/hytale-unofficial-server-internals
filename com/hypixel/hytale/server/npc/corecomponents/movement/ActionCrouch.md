---
description: Architectural reference for ActionCrouch
---

# ActionCrouch

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient Command

## Definition
```java
// Signature
public class ActionCrouch extends ActionBase {
```

## Architecture & Concepts
ActionCrouch is a concrete implementation of the Command Pattern within the server-side NPC AI framework. It represents a single, atomic, and instantaneous action that an NPC can perform as part of a larger behavior sequence.

Its sole responsibility is to mutate the state of an entity's MovementStatesComponent. This class acts as a leaf node within an NPC's Behavior Tree, directly translating an AI decision into a low-level state change on a game entity. It does not contain complex, multi-tick logic; it is a simple state mutator that completes its work within the same game tick it is executed.

This decouples the high-level AI decision-making (e.g., "take cover") from the low-level implementation of character movement states.

### Lifecycle & Ownership
- **Creation:** ActionCrouch instances are not created dynamically during the game loop. They are instantiated once by a higher-level AI builder system (e.g., a Behavior Tree parser reading from a definition file) when an NPC's behavioral profile is loaded into memory.
- **Scope:** The object's lifetime is tied to the NPC's Role or behavior definition. It is a reusable, stateless command object that persists as long as the NPC's AI definition is loaded.
- **Destruction:** The object is marked for garbage collection when the server unloads the corresponding NPC definition, for instance, during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** Immutable. The core state, a boolean named crouching, is declared final and is set exclusively via the constructor. The object itself carries no per-execution or per-NPC state, making it highly reusable.
- **Thread Safety:** The ActionCrouch object is inherently thread-safe due to its immutable nature. However, the execution context is critical. The execute method modifies component data within an EntityStore. This operation is **not** safe to call from arbitrary threads. It is designed to be invoked exclusively from the main server thread that owns the entity's world, which guarantees serialized, single-threaded access to component data for a given tick.

## API Surface
The public contract is limited to the execute method inherited from ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Modifies the crouching state on the target entity's MovementStatesComponent. Always returns true, signifying the action is instantaneous and successful. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is executed by the NPC's internal AI scheduler or Behavior Tree runner. The system retrieves the pre-configured ActionCrouch instance associated with a behavior and calls its execute method.

```java
// CONCEPTUAL: How the AI system would invoke this action.
// This code would exist deep within an NPC's Role or BehaviorTree processor.

// Assume 'currentAction' is an ActionCrouch instance
// loaded from the NPC's definition.
boolean actionCompleted = currentAction.execute(entityRef, role, sensorInfo, dt, entityStore);

if (actionCompleted) {
    // The behavior tree can now transition to the next action.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionCrouch()` in gameplay logic. Actions must be defined declaratively within an NPC's behavior files and managed by the AI system. Direct instantiation bypasses the AI framework and can lead to undefined behavior.
- **Stateful Subclassing:** Do not extend ActionCrouch to manage a state that persists across multiple ticks. This class is designed to be an instantaneous command. For long-running or multi-stage behaviors, use a more appropriate construct like a Goal or a sequence of Actions.

## Data Pipeline
ActionCrouch acts as a terminal point in the AI decision pipeline, translating a behavior into a data mutation. This mutation is then consumed by other downstream systems.

> Flow:
> NPC Behavior Tree Evaluator -> **ActionCrouch.execute()** -> Mutates MovementStatesComponent -> Physics & Animation Systems (on subsequent ticks)

