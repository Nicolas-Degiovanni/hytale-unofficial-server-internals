---
description: Architectural reference for ActionFlockSetTarget
---

# ActionFlockSetTarget

**Package:** com.hypixel.hytale.server.flock.corecomponents
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ActionFlockSetTarget extends ActionBase {
```

## Architecture & Concepts
ActionFlockSetTarget is a concrete implementation of the Command Pattern within the server-side NPC AI framework. It represents a single, atomic operation that an NPC can perform: updating its internal memory about a target entity. This class acts as the bridge between an NPC's sensory input system (represented by InfoProvider) and its persistent state (managed by the Role object).

Its primary responsibility is to take a target, often identified by sensors, and commit it to a named "target slot" within the NPC's memory. This allows other, subsequent AI actions in a behavior tree—such as movement or combat actions—to operate on a stable, named reference without needing to re-acquire the target from sensors on every tick.

This action is specifically scoped to entities participating in the flocking system, as enforced by the `FlockPlugin.isFlockMember` check. It is a fundamental building block for coordinated group behaviors, enabling a flock member to "lock on" to a target for the rest of the flock to potentially reference.

The action supports two distinct modes:
1.  **Set Mode:** Assigns a currently sensed entity to a specified target slot.
2.  **Clear Mode:** Removes any entity reference from a specified target slot, effectively making the NPC "forget" its target.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly via their constructor. They are instantiated by the server's asset loading pipeline through a corresponding builder, BuilderActionFlockSetTarget. This occurs when an NPC's behavior definition (e.g., a behavior tree defined in a data file) is parsed and loaded into memory.
-   **Scope:** The object's lifetime is tied to the NPC's loaded AI definition. It is a stateless, reusable component within a larger AI graph (like a Behavior Tree node) and persists as long as the parent NPC's configuration is loaded. It is not created per-tick or per-execution.
-   **Destruction:** The object is eligible for garbage collection when the NPC's Role or the entire NPC entity is unloaded from the world, for instance, during a zone unload or server shutdown.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal fields `clear` and `targetSlot` are final and are configured exclusively at creation time via the builder. This object holds no mutable state and does not cache any world data. All state modifications are performed on external objects passed as parameters to its methods.

-   **Thread Safety:** **Conditionally Safe**. The object's immutable nature makes it inherently thread-safe. However, its methods are designed to manipulate core game state objects (Role, Store) which are **not thread-safe**. Therefore, the `canExecute` and `execute` methods **must** be called exclusively from the main server thread that owns the NPC's game state.

    **WARNING:** Invoking these methods from any other thread will lead to severe concurrency issues, including data corruption, race conditions, and server instability.

## API Surface
The public contract is defined by its parent, ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Evaluates if the action can be performed. Checks for flock membership and, if not clearing, the existence of a valid target in the sensor data. |
| execute(...) | boolean | O(1) | Performs the state modification. Updates the NPC's Role by setting or clearing the entity reference in the specified target slot. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by gameplay code. It is designed to be configured within an NPC's asset files and executed automatically by the server's AI engine (e.g., a Behavior Tree or State Machine) during an NPC's update tick.

The following conceptual example illustrates how the AI engine would use the action:

```java
// Conceptual example from within an AI Behavior Tree node
// This code is part of the engine, not typical user code.

ActionFlockSetTarget action = findConfiguredAction(); // Load from NPC definition
Ref<EntityStore> npcRef = ...;
Role npcRole = ...;
InfoProvider sensorData = ...;
Store<EntityStore> worldStore = ...;

// The engine first checks if the action is valid in the current context
if (action.canExecute(npcRef, npcRole, sensorData, dt, worldStore)) {
    // If valid, the engine executes the action to modify game state
    action.execute(npcRef, npcRole, sensorData, dt, worldStore);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ActionFlockSetTarget()`. The object is data-driven and must be configured and created through the `BuilderActionFlockSetTarget` as part of the asset loading pipeline. Direct creation bypasses critical configuration steps.
-   **Off-Thread Execution:** Do not call `execute` from a worker thread or asynchronous task. All interactions with the `Role` and `Store` must be synchronized with the main server tick.
-   **Stateful Misuse:** Do not cache or hold references to this object outside of the AI system that owns it. It is a stateless command and should be treated as part of an NPC's static definition.

## Data Pipeline
The flow of data and control for this action follows a clear path from sensory input to state change.

> Flow:
> NPC Sensor System (InfoProvider) -> AI Engine Tick -> **ActionFlockSetTarget.execute()** -> Role's Target Memory (MarkedEntitySupport) -> Subsequent AI Actions (e.g., Pathfinding, Combat)

