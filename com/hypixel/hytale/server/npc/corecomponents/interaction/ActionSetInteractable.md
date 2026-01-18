---
description: Architectural reference for ActionSetInteractable
---

# ActionSetInteractable

**Package:** com.hypixel.hytale.server.npc.corecomponents.interaction
**Type:** Transient

## Definition
```java
// Signature
public class ActionSetInteractable extends ActionBase {
```

## Architecture & Concepts
The ActionSetInteractable class is a concrete implementation of the Command Pattern, designed to operate within the server-side NPC (Non-Player Character) behavior system. It functions as a single, atomic "verb" that an NPC can execute as part of a larger behavior tree or state machine.

Its primary architectural role is to decouple the *intent* to change an entity's interaction state from the underlying systems that manage that state. This class encapsulates the *what* (set interactable to true/false, with a specific hint) while the NPC's `Role` and the behavior tree control the *when* and *why*.

This action is data-driven. Instances are not created dynamically in game logic but are defined in NPC asset files and instantiated by the asset loading system via its corresponding builder, `BuilderActionSetInteractable`. It typically operates on a target entity that has been previously identified and stored in the NPC's `StateSupport` component, making it a common follow-up action in a sequence.

## Lifecycle & Ownership
- **Creation:** Instantiated by the `BuilderActionSetInteractable` during the server's asset loading phase. It is part of an NPC's pre-defined behavior template, not created dynamically during gameplay.
- **Scope:** The object's lifetime is bound to the NPC's loaded `Role` definition. As a stateless command object, the same instance is reused every time the NPC's behavior tree executes this specific action.
- **Destruction:** The object is marked for garbage collection when the parent `Role` or the entire NPC entity is unloaded from the world.

## Internal State & Concurrency
- **State:** Immutable. Its core parameters—`setTo`, `hint`, and `showPrompt`—are final fields set once at construction. This object holds no mutable world state; it is a pure data container for the action's configuration.
- **Thread Safety:** The object itself is inherently thread-safe due to its immutability. However, its methods are not. The `execute` method performs write operations on the world's `EntityStore`.

    **WARNING:** The `execute` and `canExecute` methods must **only** be called from the main server thread responsible for the NPC's update tick. Invoking them from any other thread will lead to critical race conditions, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Checks if the action can be performed. Critically, it verifies that `StateSupport` has a valid `interactionIterationTarget`. |
| execute(...) | boolean | O(1) | Modifies the target entity's interaction state via the NPC's `StateSupport`. Always returns true to signal successful completion to the behavior tree. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by developers. It is designed to be configured within an NPC asset file and executed automatically by the NPC's behavior engine. The engine is responsible for calling `canExecute` and then `execute` as it traverses the behavior tree.

A conceptual asset definition might look like this:

```json
// Hypothetical NPC Behavior Asset
{
  "behaviorTree": {
    "root": {
      "type": "Sequence",
      "children": [
        { "type": "FindNearestPlayer", "setTargetAs": "interactionIterationTarget" },
        {
          "type": "ActionSetInteractable",
          "setTo": true,
          "hint": "Press E to talk",
          "showPrompt": true
        }
      ]
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionSetInteractable()`. The object must be configured via its builder, which is handled by the asset system. Direct instantiation bypasses critical configuration steps.
- **Execution Without Target:** Calling `execute` when `canExecute` would return false is a logical error. This typically happens if the NPC's `StateSupport` does not have an `interactionIterationTarget` set, which will result in a NullPointerException or other undefined behavior.

## Data Pipeline
ActionSetInteractable acts as a terminal node in a logic pipeline, initiating a state change rather than transforming data. The control flow leading to its execution is paramount.

> Flow:
> NPC Behavior Tree Tick -> Logic Node (e.g., FindEntity) sets `interactionIterationTarget` -> **ActionSetInteractable.canExecute()** -> (if true) -> **ActionSetInteractable.execute()** -> `StateSupport.setInteractable()` -> `EntityStore` Component Update -> State replicated to clients -> Client UI displays interaction prompt

