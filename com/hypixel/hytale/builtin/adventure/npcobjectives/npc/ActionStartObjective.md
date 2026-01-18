---
description: Architectural reference for ActionStartObjective
---

# ActionStartObjective

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.npc
**Type:** Transient

## Definition
```java
// Signature
public class ActionStartObjective extends ActionBase {
```

## Architecture & Concepts
The ActionStartObjective class is a concrete implementation of the ActionBase, representing a single, atomic operation within the server-side NPC Behavior system. Its primary architectural role is to act as a bridge between an NPC's artificial intelligence and the player-facing quest or objective system.

When an NPC's behavior tree or state machine determines that this action should run, its purpose is to initiate a specific objective for a targeted player. The action is configured with a unique *objectiveId*, which identifies the objective to be started.

Crucially, this class is decoupled from the specific entity it targets. The target, typically a player, is resolved at runtime by querying the NPC's current state via the Role object, specifically the *interactionIterationTarget*. This design allows a single, statically defined action to be reused for any player the NPC interacts with, without requiring a unique action instance per target. The actual logic for starting the objective is delegated to the NPCObjectivesPlugin, making this class a lightweight command object.

## Lifecycle & Ownership
- **Creation:** Instances of ActionStartObjective are not created procedurally during gameplay. They are instantiated by the server's NPC asset loading pipeline when parsing an NPC's behavior definition, typically from a JSON asset file. The corresponding BuilderActionStartObjective is responsible for its construction.
- **Scope:** An instance persists for the lifetime of the NPC's loaded behavior asset. It is part of a shared, immutable behavior template.
- **Destruction:** The object is eligible for garbage collection when the NPC's behavior asset is unloaded from memory, for example, when a world is shut down or a hot-reload of assets occurs.

## Internal State & Concurrency
- **State:** The internal state of this class is immutable. Its only significant field, *objectiveId*, is final and set during construction. It does not cache data or maintain state across executions. All contextual information required for execution is passed as method arguments.
- **Thread Safety:** This class is inherently thread-safe due to its immutable state. However, its methods are designed to operate on the core game state, which is not thread-safe.

    **WARNING:** The *canExecute* and *execute* methods must **only** be called from the main server tick thread that owns the NPC entity. Invoking them from any other thread will lead to severe concurrency violations, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(ref, role, ...) | boolean | O(1) | Returns true if the base conditions are met and the NPC has a valid interaction target. |
| execute(ref, role, ...) | boolean | O(1) | Triggers the objective start via NPCObjectivesPlugin for the current interaction target. Always returns true. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code by developers. Instead, it is configured declaratively within an NPC's behavior asset files. The engine's asset systems are responsible for its instantiation and integration into the NPC's behavior tree.

A conceptual asset configuration might look like this:
```json
// Example NPC Behavior Asset
{
  "id": "quest_giver_behavior",
  "actions": [
    {
      "type": "ActionStartObjective",
      "objectiveId": "com.my.mod.quests.collect_ten_rocks"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call *new ActionStartObjective()*. The object is missing critical context when not created through its designated builder during the asset loading process.
- **Missing Interaction Target:** Do not place this action in a behavior state where the NPC will not have an *interactionIterationTarget* set. The *canExecute* method will consistently return false, and the action will never run.
- **Invalid Objective ID:** Configuring the action with an *objectiveId* that is not registered with the NPCObjectivesPlugin will result in a runtime failure or a silent drop of the action.

## Data Pipeline
ActionStartObjective functions as a command trigger, not a data processor. Its execution initiates a change in the game state.

> Flow:
> NPC Behavior Tick -> **ActionStartObjective.execute()** -> NPCObjectivesPlugin.startObjective() -> Objective System -> Player Component Update -> Network Sync to Client

