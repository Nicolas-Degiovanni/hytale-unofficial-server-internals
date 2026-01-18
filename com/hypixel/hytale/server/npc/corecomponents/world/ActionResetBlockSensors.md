---
description: Architectural reference for ActionResetBlockSensors
---

# ActionResetBlockSensors

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class ActionResetBlockSensors extends ActionBase {
```

## Architecture & Concepts
ActionResetBlockSensors is a concrete implementation of the ActionBase command pattern, operating within the server-side NPC Artificial Intelligence framework. Its sole responsibility is to invalidate the cached results of an NPC's environmental block sensors, forcing a re-evaluation of the surrounding world geometry on a subsequent query.

This component is a fundamental tool for managing an NPC's situational awareness. NPCs use block sensors to perceive their environment—for example, to detect walls, obstacles, or specific material types. These sensor results are often cached for performance. This action serves as the mechanism to explicitly clear that cache.

It is typically employed within a behavior tree or finite-state machine when an NPC's behavior requires a fresh, unbiased perception of its environment. Common use cases include:
*   At the beginning of a new task, such as mining or building, to ensure the NPC starts with up-to-date world information.
*   After the world has been significantly altered by the NPC or a player.
*   As a recovery mechanism when an NPC's pathfinding or behavior appears to be stuck based on stale sensor data.

The action can be configured to reset all block sensors simultaneously or to target specific, enumerated sets of sensors via the internal *blockSets* array.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly at runtime. They are instantiated by the Hytale asset pipeline via a corresponding builder, BuilderActionResetBlockSensors, during the loading and parsing of NPC behavior definition files (e.g., JSON or HOCON assets). The BuilderSupport context object is used to resolve dependencies and register the action's intent with the sensor system.
-   **Scope:** An ActionResetBlockSensors object is immutable and stateless after creation. It is owned by the larger AI construct that contains it, such as a behavior tree node. Its lifetime is tied to the lifetime of the loaded NPC behavior asset.
-   **Destruction:** The object is eligible for garbage collection when the NPC behavior asset it belongs to is unloaded from memory.

## Internal State & Concurrency
-   **State:** The internal state is **Immutable**. The primary state, the integer array *blockSets*, is initialized in the constructor and marked as final. The object itself does not hold or cache any runtime data; it is a pure command object.
-   **Thread Safety:** This class is inherently thread-safe due to its immutable nature. However, the execute method operates on a mutable Role object. The NPC AI execution engine is responsible for guaranteeing that an individual NPC's Role and its associated components are accessed and modified from a single thread or within a synchronized context during an AI tick. This action does not introduce any new concurrency hazards.

## API Surface
The public contract is exclusively defined by the overridden *execute* method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(N) | Executes the sensor reset logic. N is the number of configured block sets. Always returns true, indicating successful, immediate completion. |

## Integration Patterns

### Standard Usage
This action is not intended to be invoked directly from game logic code. It is designed to be embedded within an NPC behavior asset file and executed by the AI scheduler as part of a larger behavioral sequence.

A conceptual behavior tree node definition might look like this:

```json
// Hypothetical NPC Behavior Asset
{
  "sequence": [
    {
      "action": "ActionResetBlockSensors",
      "params": {
        // Omitting params resets ALL sensors
      }
    },
    {
      "action": "ActionFindNearestResource",
      "params": {
        "resourceType": "Hytale:Ore"
      }
    },
    {
      "action": "ActionMoveToTarget"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never attempt to create an instance using `new ActionResetBlockSensors()`. The constructor requires builder-specific context that is only available during asset loading. Failure to use the asset pipeline will result in an improperly configured and non-functional action.
-   **Runtime Configuration:** Do not attempt to modify the internal *blockSets* array after initialization. The object is designed to be immutable, and its configuration should be treated as static data defined in the source asset.

## Data Pipeline
ActionResetBlockSensors acts as a command initiator rather than a data processor. Its execution triggers a state change in a downstream system.

> Flow:
> NPC AI Scheduler -> **ActionResetBlockSensors.execute()** -> Role.getWorldSupport() -> Internal Block Sensor Cache (State is invalidated)

