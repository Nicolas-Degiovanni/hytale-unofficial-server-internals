---
description: Architectural reference for ActionTriggerSpawnBeacon
---

# ActionTriggerSpawnBeacon

**Package:** com.hypixel.hytale.server.spawning.corecomponents
**Type:** Transient Behavior

## Definition
```java
// Signature
public class ActionTriggerSpawnBeacon extends ActionBase {
```

## Architecture & Concepts
The ActionTriggerSpawnBeacon is a concrete implementation of the `ActionBase` contract, operating within the server-side NPC AI and behavior system. Its primary architectural function is to serve as a command object that bridges an NPC's decision-making process with the world's entity spawning mechanics.

This class does not implement spawning logic itself. Instead, it acts as a highly specific delegator. When executed by an NPC's behavior controller (such as a behavior tree or state machine), it performs a targeted search for a `SpawnBeacon` entity in the world that matches a predefined `beaconId`. Upon finding the correct beacon, it invokes the beacon's `manualTrigger` method, initiating a spawn event.

This design decouples the NPC's intent ("I need to spawn reinforcements here") from the mechanics of spawning ("how and what to spawn"). It allows game designers to create complex, scripted spawning encounters driven by NPC actions, rather than relying on simple proximity or timed triggers.

## Lifecycle & Ownership
- **Creation:** Instances are not created dynamically during gameplay. They are instantiated by the server's asset loading pipeline when an NPC's behavior definition is loaded from game data files. The `BuilderActionTriggerSpawnBeacon` class is responsible for parsing the asset definition and constructing this object.
- **Scope:** The lifetime of an ActionTriggerSpawnBeacon instance is tied to its parent NPC's `Role` component. It exists as one of many potential actions in the NPC's behavioral repertoire for as long as the NPC is active in the world.
- **Destruction:** The object is marked for garbage collection when the parent NPC entity is unloaded from the world or when its behavior definition is reloaded or swapped.

## Internal State & Concurrency
- **State:** Immutable. The core configuration fields—`beaconId`, `range`, and `targetSlot`—are final and are set once during construction from the asset builder. This object holds no mutable state and does not cache data between executions. Each call to `execute` is an independent, stateless operation.
- **Thread Safety:** Conditionally safe. While the object itself is immutable, its methods are designed to operate on mutable, live game-world state (via the `Store<EntityStore>` parameter). These methods are not thread-safe and must be called exclusively from the main server thread responsible for the entity's world zone.

**WARNING:** Invoking any method on this class from an asynchronous task or a different thread will result in severe world state corruption, race conditions, and server instability.

## API Surface
The public contract is defined by its `ActionBase` parent. The key overrides provide the specialized spawning behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Performs a precondition check. Verifies that if a `targetSlot` is defined, a valid entity exists in that slot on the NPC's `MarkedEntitySupport`. |
| registerWithSupport(Role) | void | O(1) | A setup hook called when the action is added to an NPC. It instructs the NPC's `PositionCache` to begin tracking nearby `SpawnBeacon` entities. |
| execute(...) | boolean | O(N) | The core logic. Iterates through nearby beacons (N) to find and trigger the one matching `beaconId`. Always returns true to signify the action is complete. |

## Integration Patterns

### Standard Usage
This class is intended to be used declaratively within NPC asset files. A game designer would define this action as a node in an NPC's behavior tree. The AI system is solely responsible for invoking the `execute` method when the behavior tree logic dictates it.

```java
// PSEUDOCODE: How the AI system uses this action
// This code is conceptual and does not exist in this class.

ActionBase currentAction = npc.getBehaviorTree().selectAction();

if (currentAction instanceof ActionTriggerSpawnBeacon) {
    if (currentAction.canExecute(...)) {
        currentAction.execute(...);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ActionTriggerSpawnBeacon()`. The object's configuration is data-driven and must be loaded through the `BuilderActionTriggerSpawnBeacon` via the asset system. Direct instantiation will result in a misconfigured and non-functional action.
- **Stateful Modification:** Do not attempt to modify this class to hold state across multiple ticks. The `ActionBase` contract assumes actions are stateless commands.
- **Ignoring Preconditions:** Calling `execute` without a preceding successful `canExecute` check violates the component's contract and may lead to `NullPointerException` or failed assertions if a required `targetSlot` entity is missing.

## Data Pipeline
This component acts as a trigger within a command flow, not a traditional data transformation pipeline.

> Flow:
> NPC Behavior Tree selects action -> AI System invokes **ActionTriggerSpawnBeacon.execute()** -> Queries `Role.PositionCache` for nearby beacons -> Finds matching `SpawnBeacon` component -> Invokes `SpawnBeacon.manualTrigger()` -> Spawning System creates new entity

