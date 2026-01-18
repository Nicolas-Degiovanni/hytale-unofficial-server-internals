---
description: Architectural reference for ActionSetBlockToPlace
---

# ActionSetBlockToPlace

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class ActionSetBlockToPlace extends ActionBase {
```

## Architecture & Concepts
ActionSetBlockToPlace is a command object within the server-side NPC AI framework. It represents a single, atomic instruction within an NPC's behavior tree or state machine. Its sole responsibility is to set the *intent* for an NPC to place a specific type of block in the world.

This class does not perform the block placement itself. Instead, it modifies the internal state of the NPC's active Role, specifically by updating the WorldSupport component. This decouples the decision-making logic (what block to place) from the world-interaction logic (the physical act of placing it). A subsequent action, such as ActionPlaceBlock, is expected to consume this state and execute the placement.

This pattern allows for more complex and flexible AI behaviors. For example, an NPC can decide on a block type based on one set of conditions and then wait for a different set of conditions to be met before actually placing it.

## Lifecycle & Ownership
- **Creation:** Instances are created by the NPC asset loading pipeline. The constructor's dependency on a BuilderActionSetBlockToPlace and BuilderSupport indicates that it is instantiated during the deserialization of an NPC's configuration files, not created dynamically at runtime by game logic.
- **Scope:** The object is extremely short-lived. It is created as part of an NPC's behavior definition and is effectively stateless beyond its initial configuration. Its methods are invoked during the NPC's update tick, and the instance itself is not persisted between ticks.
- **Destruction:** The object is eligible for garbage collection immediately after the NPC's behavior evaluation for a given tick is complete. It is not managed by a registry or service locator.

## Internal State & Concurrency
- **State:** The internal state is **Immutable**. The core data, blockType, is a final string initialized in the constructor. The object itself does not change after creation.
- **Thread Safety:** The class is inherently thread-safe due to its immutability. However, it is designed to be used exclusively within the single-threaded context of an individual NPC's update loop. The execute method mutates the passed-in Role object, which is not thread-safe.

**WARNING:** While the object itself is thread-safe, its methods cause side effects on mutable state (the Role). Invoking its methods from outside the designated NPC update thread will lead to race conditions and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Checks if the action can be run. Verifies that the configured blockType exists in the global BlockType asset map. |
| execute(...) | boolean | O(1) | Sets the block-to-place state on the NPC's WorldSupport component. Always returns true to signal successful execution. |

## Integration Patterns

### Standard Usage
This action is not intended to be invoked directly by developers. It is configured within an NPC's behavior files and executed by the server's AI processing system. The system evaluates the NPC's current behavior tree and calls the execute method on the active action node.

```java
// Engine-level code that would execute this action
// Note: This is a conceptual example of the engine's behavior loop.

// Inside an NPC's update method...
ActionBase currentAction = npc.getBehaviorTree().getActiveAction();

if (currentAction instanceof ActionSetBlockToPlace) {
    if (currentAction.canExecute(ref, role, sensorInfo, dt, store)) {
        currentAction.execute(ref, role, sensorInfo, dt, store);
        // The role's WorldSupport now holds the blockType string.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ActionSetBlockToPlace()`. The class requires a builder context that is only available during the server's asset loading phase. Actions must be defined in data files.
- **State Stagnation:** Configuring this action without a corresponding consumer action (e.g., ActionPlaceBlock) in the behavior tree will lead to logical errors. The NPC will set its intent to place a block but will never act upon it.

## Data Pipeline
This component acts as a state setter in the NPC data flow. It translates a static configuration value into a dynamic runtime state within the NPC's Role.

> Flow:
> NPC Behavior Tree Evaluation -> **ActionSetBlockToPlace.execute()** -> Writes blockType string to Role.WorldSupport -> Subsequent Action (e.g., ActionPlaceBlock) reads blockType from Role.WorldSupport

