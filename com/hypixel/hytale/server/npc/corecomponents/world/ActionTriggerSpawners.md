---
description: Architectural reference for ActionTriggerSpawners
---

# ActionTriggerSpawners

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class ActionTriggerSpawners extends ActionBase {
```

## Architecture & Concepts
The ActionTriggerSpawners class is a concrete implementation of the Action command pattern, used within the server-side NPC behavior system. Its sole responsibility is to locate and activate nearby spawn markers, which are entities responsible for spawning other entities into the world.

This action serves as a bridge between an NPC's AI logic (e.g., a behavior tree) and the world's spawning mechanics. Rather than performing a costly, brute-force search of all entities in the world, it leverages a critical optimization: the NPC Role's PositionCache. During its initialization phase, this action registers its maximum effective range with the cache. The PositionCache is then responsible for maintaining an updated, localized list of relevant entities, ensuring that the execution of this action is highly performant.

The action supports two primary modes of operation determined by its configuration:
1.  **Broadcast Trigger:** Activates all valid manual-trigger spawners within the specified range.
2.  **Sampled Trigger:** Activates a specific, randomly-sampled count of valid spawners within range.

This component is fundamental for creating dynamic, scripted events such as ambushes, reinforcements, or environmental interactions driven by NPC behavior.

## Lifecycle & Ownership
-   **Creation:** An instance of ActionTriggerSpawners is not created directly. It is instantiated by the server's asset loading pipeline via its corresponding builder, BuilderActionTriggerSpawners. This typically occurs when an NPC's personality or behavior asset file is parsed and loaded into memory.
-   **Scope:** The object's lifetime is tied to the NPC's loaded behavior set. It persists as a configured, reusable component of the NPC's AI. It is not created per-tick or per-execution.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC is unloaded from the world or when its AI configuration is reloaded with a new set of actions.

## Internal State & Concurrency
-   **State:** This class is stateful. It holds immutable configuration data (spawner ID, range, count) and mutable, transient state used during a single execution cycle (parentRef, triggerList). The triggerList, which stores the sampled spawners, is cleared at the end of each execute call, making the action instance safe for reuse on subsequent server ticks.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and executed exclusively by a single NPC's update logic within the main server game loop. The internal state, particularly parentRef and triggerList, is not protected by locks. Concurrent calls to execute will result in race conditions and undefined behavior.

## API Surface
The primary contract is defined by the ActionBase interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerWithSupport(Role role) | void | O(1) | Registers the action's range with the NPC's PositionCache. **WARNING:** Must be called during NPC initialization. |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(N) | Executes the trigger logic. N is the number of spawners in the PositionCache. Always returns true. |

## Integration Patterns

### Standard Usage
This action is not intended to be invoked directly by developers. It is executed by a higher-level AI system, such as a behavior tree or a state machine, which manages the NPC's decision-making process. The system is responsible for calling `registerWithSupport` at setup and `execute` during the appropriate AI state.

```java
// PSEUDOCODE: Simplified representation of a Behavior Tree node executing the action.

// During NPC initialization:
ActionTriggerSpawners myAction = npc.getBehavior().getAction("triggerAmbushSpawners");
myAction.registerWithSupport(npc.getRole());

// During the server tick, when the behavior node is active:
boolean result = myAction.execute(npc.getRef(), npc.getRole(), ...);
// The action handles finding and triggering spawners internally.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ActionTriggerSpawners()`. The object must be configured and created via the `BuilderActionTriggerSpawners` during asset loading to ensure its parameters (range, count, etc.) are correctly initialized.
-   **Skipping Registration:** Failure to call `registerWithSupport` during the NPC's setup phase will cause the action to fail silently. The `PositionCache` will not be aware of the action's needs and will likely provide an empty list of spawners to `execute`.
-   **Concurrent Execution:** Do not share a single ActionTriggerSpawners instance across multiple NPCs or access it from asynchronous tasks. Each NPC should have its own instance as part of its behavior definition.

## Data Pipeline
The flow of control and data for this action is linear and synchronous within a single server tick.

> Flow:
> NPC Behavior System -> **ActionTriggerSpawners.execute()** -> Role.PositionCache.getSpawnMarkerList() -> **ActionTriggerSpawners.filterMarker()** -> SpawnMarkerEntity.trigger() -> Server Spawning System

