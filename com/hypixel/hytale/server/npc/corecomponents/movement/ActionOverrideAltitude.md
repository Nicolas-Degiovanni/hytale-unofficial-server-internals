---
description: Architectural reference for ActionOverrideAltitude
---

# ActionOverrideAltitude

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient Component

## Definition
```java
// Signature
public class ActionOverrideAltitude extends ActionBase {
```

## Architecture & Concepts
ActionOverrideAltitude is a concrete implementation of the ActionBase class, designed to operate within the server-side NPC Behavior System. Its sole purpose is to enforce a specific flight altitude range on an NPC.

This class acts as a command object within a larger behavior framework, such as a Behavior Tree or a Finite State Machine. When executed, it directly injects a desired altitude configuration into the NPC's active MotionControllerFly. This decouples the high-level behavioral goal—*fly at a certain height*—from the low-level implementation of flight physics and pathfinding, which are managed by the motion controller.

This component is not intended for direct manual instantiation. It is constructed by the NPC asset pipeline from declarative definitions (e.g., JSON files), allowing designers to specify NPC flight behavior without writing code.

## Lifecycle & Ownership
- **Creation:** Instantiated by the NPC asset loading system via its corresponding builder, BuilderActionOverrideAltitude. This occurs when an NPC's behavior graph is constructed from its asset definition at runtime.
- **Scope:** The object's lifetime is tied to the NPC's loaded behavior configuration. It persists as a node within the NPC's behavior graph and is reused across multiple executions of the action.
- **Destruction:** The object is eligible for garbage collection when its parent NPC entity is unloaded or when the NPC's behavior set is reloaded with a new configuration that no longer contains this action.

## Internal State & Concurrency
- **State:** The core state of this class is the desiredRange field, which holds the minimum and maximum desired altitude. This state is **immutable** after construction, as the field is declared final.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be invoked exclusively by the main server thread responsible for ticking the NPC's logic. It contains no internal synchronization mechanisms. Calling its methods from a concurrent thread will lead to race conditions and undefined behavior in the NPC's movement controller.

## API Surface
The public contract is defined by its parent, ActionBase. The key methods are for evaluation and execution within a behavior system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Returns true only if the NPC's active motion controller is of type "Fly". Fails for all ground-based or swimming NPCs. |
| execute(...) | boolean | O(1) | Injects the configured altitude range into the NPC's MotionControllerFly. Assumes canExecute has already returned true. |

## Integration Patterns

### Standard Usage
This class is not invoked directly. It is executed by a higher-level NPC behavior system. The system evaluates `canExecute` to determine if the action is valid in the current context and then calls `execute` to apply its logic.

```java
// Conceptual example of how a Behavior Tree Executor would use this action.
// Developers do not write this code; it is part of the core engine.

ActionBase currentAction = npc.getBehaviorTree().getActiveAction();

if (currentAction instanceof ActionOverrideAltitude) {
    // The executor first checks if the action is valid
    if (currentAction.canExecute(npcRef, role, sensorInfo, dt, store)) {
        // If valid, the executor runs the action's logic
        currentAction.execute(npcRef, role, sensorInfo, dt, store);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ActionOverrideAltitude()`. The object's internal state must be configured via the asset pipeline and its corresponding builder. Direct instantiation bypasses this and results in a non-functional component.
- **Misuse on Non-Flying NPCs:** Attempting to use this action on an NPC that is not configured with a MotionControllerFly is a design error. The `canExecute` method will always return false, preventing the action from running.
- **External State Modification:** Do not use reflection or other means to modify the internal desiredRange field after construction. The immutability of its configuration is a key design assumption.

## Data Pipeline
ActionOverrideAltitude acts as a control-flow component rather than a data-processing one. It injects a static configuration value into a different system.

> Flow:
> NPC Behavior Executor -> `execute()` -> **ActionOverrideAltitude** -> `MotionControllerFly.setDesiredAltitudeOverride(range)` -> NPC Movement System Tick

