---
description: Architectural reference for ActionMakePath
---

# ActionMakePath

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class ActionMakePath extends ActionBase {
```

## Architecture & Concepts
ActionMakePath is a concrete implementation of the ActionBase class, operating within the server-side NPC Behavior System. Its sole responsibility is to dynamically generate a navigational path for an NPC and inject it into the NPC's pathfinding component.

This class functions as a single-use command object within an NPC's behavior tree or finite state machine. It translates a high-level, data-driven path definition, represented by TransientPathDefinition, into a concrete, traversable IPath instance at runtime. This allows NPC behavior to be authored in asset files, with path generation triggered by specific in-game events or states.

The action is fundamentally a one-shot operation. Once it successfully generates and assigns a path to the target NPC, it internally flags itself as complete and will not execute again.

## Lifecycle & Ownership
-   **Creation:** An ActionMakePath instance is not created directly. It is instantiated by the NPC asset loading pipeline, specifically through its corresponding builder, BuilderActionMakePath. This typically occurs when the server loads an NPC's behavior profile from a definition file.
-   **Scope:** The lifetime of this object is ephemeral and is strictly tied to the specific node in the NPC's behavior tree that defines it. It persists only as long as that behavior state is active.
-   **Destruction:** The object is eligible for garbage collection as soon as the NPC's behavior controller transitions to a new state and drops the reference to this action. It does not manage any unmanaged resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** This component is stateful and mutable. It maintains an internal boolean flag, *built*, which transitions from false to true upon a single successful execution. This state change is permanent for the lifetime of the object and serves to make the action idempotent. The core path definition it holds is immutable after construction.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and executed exclusively by the main server thread responsible for the NPC update loop. All interactions with the EntityStore and its components are assumed to be synchronized by the calling game loop. Unsynchronized access from other threads will lead to severe data corruption and unpredictable behavior.

## API Surface
The public contract is defined by its parent, ActionBase, and is intended for consumption by the NPC behavior system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Returns true only if the base conditions are met and the path has not yet been built. |
| execute(...) | boolean | O(N) | Generates the path from the definition and assigns it to the NPC's PathManager. Complexity is dependent on the number of waypoints (N) in the path definition. Modifies the target NPC's state. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically managed and executed by an NPC's Role or an equivalent behavior processing system. The system identifies the current active action for an NPC and invokes its methods during the server tick.

```java
// Conceptual example of how the behavior system uses this action.
// This code does not exist in a single place but represents the logical flow.

// During an NPC's update tick...
ActionBase currentAction = npc.getActiveBehavior().getAction();

if (currentAction instanceof ActionMakePath) {
    if (currentAction.canExecute(ref, role, sensorInfo, dt, store)) {
        // The system calls execute, which internally generates the path
        // and injects it into the NPCEntity component.
        currentAction.execute(ref, role, sensorInfo, dt, store);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ActionMakePath()`. The object is data-driven and must be constructed via its builder during asset loading to ensure its internal path definition is correctly initialized.
-   **State Reset:** Do not attempt to manually reset the internal *built* flag to force re-execution. This violates the design contract of a one-shot action. If a new path is needed, the NPC's behavior tree should transition to a new state that contains a new ActionMakePath instance.
-   **Off-Thread Execution:** Invoking `execute` from any thread other than the primary server entity-update thread is strictly forbidden. This will cause race conditions when accessing and modifying components in the EntityStore.

## Data Pipeline
The primary function of this class is to transform a static data definition into a live, in-world component state.

> Flow:
> NPC Asset File -> BuilderActionMakePath -> **ActionMakePath Instance** -> `execute()` on Server Tick -> IPath Object -> NPCEntity PathManager State Update

