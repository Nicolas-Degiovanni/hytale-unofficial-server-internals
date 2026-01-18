---
description: Architectural reference for SensorHasTask
---

# SensorHasTask

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.npc
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorHasTask extends SensorBase {
```

## Architecture & Concepts
SensorHasTask is a specialized component within the server-side NPC AI framework. It functions as a predicate, or a conditional check, within an NPC's "Sense-Think-Act" decision-making loop. Specifically, it belongs to the **Sense** phase.

Its primary role is to bridge the generic AI system with the specific gameplay logic of the quest and objective system, managed by the NPCObjectivesPlugin. The sensor evaluates a single, precise condition: "Does the entity I am currently interacting with have an active task assigned to them that matches one of my configured task IDs?"

The result of this check, a simple boolean, is consumed by higher-level AI constructs like Behavior Trees or Finite State Machines to determine the NPC's subsequent actions. For example, an NPC might only offer a specific line of dialogue or transition to a "follow" behavior if this sensor returns true.

## Lifecycle & Ownership
-   **Creation:** SensorHasTask is not instantiated directly via code. It is created by the NPC asset loading pipeline when an NPC's definition is parsed. The constructor's dependency on a BuilderSensorHasTask and BuilderSupport confirms its origin within a builder pattern, which translates asset data into live game objects.
-   **Scope:** The lifetime of a SensorHasTask instance is tightly coupled to the NPC entity that owns it. It is a stateless component held within the NPC's configured set of sensors and persists as long as the NPC is active in the world.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC entity is unloaded or destroyed. It has no explicit cleanup or destruction method.

## Internal State & Concurrency
-   **State:** The component's primary state is the `tasksById` array, which holds the list of objective task identifiers to check for. This array is marked `final` and is initialized only once in the constructor, making the component's configuration **immutable** after creation. The `matches` method is otherwise stateless between invocations.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be executed exclusively on the main server thread as part of the synchronous game tick. It reads from the global `Store<EntityStore>` (the ECS world state) and interacts with the `NPCObjectivesPlugin` singleton. These operations are not protected by locks and assume a single-threaded execution context to prevent data races.

## API Surface
The public contract is minimal, consisting almost entirely of the `matches` method inherited from `SensorBase`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | Evaluates the sensor's condition. N is the number of task IDs configured. Returns true if the NPC's interaction target has any of the specified tasks. **Warning:** This method has a side effect of calling `addTargetPlayerActiveTask` on the NPC's `EntitySupport` upon a successful match. |
| getSensorInfo() | InfoProvider | O(1) | Returns metadata for debugging tools. Currently unimplemented (returns null). |

## Integration Patterns

### Standard Usage
This component is not intended to be invoked directly from gameplay code. Instead, it is configured declaratively within an NPC's asset files (e.g., a JSON or HOCON file). The engine's AI system then invokes the `matches` method automatically during the NPC's update cycle.

A conceptual asset configuration might look like this:
```json
// Example NPC Asset Snippet
{
  "ai": {
    "sensors": [
      {
        "type": "SensorHasTask",
        "tasks": [
          "hytale:quests/main/collect_wood",
          "hytale:quests/side/find_artifact"
        ]
      }
    ]
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SensorHasTask()`. The object is deeply tied to the asset loading and NPC builder systems. Manual creation will result in a misconfigured and non-functional sensor.
-   **Ignoring Side Effects:** The `matches` method is not a pure function. It modifies the NPC's `EntitySupport` state by adding the matched task ID. Logic that relies on this sensor must be aware that a `true` result also signals this state change, which may be used by other AI components in the same tick.
-   **Cross-Thread Access:** Invoking `matches` from any thread other than the main server thread will lead to severe concurrency issues, likely causing crashes or data corruption.

## Data Pipeline
The flow of data during a single `matches` call is linear and synchronous.

> Flow:
> NPC AI Tick -> **SensorHasTask.matches()** -> Read `Role` for interaction target -> Read `Store` for NPC and Target UUIDs -> Query `NPCObjectivesPlugin` with UUIDs and Task IDs -> (If Match) Write to `EntitySupport` -> Return `boolean` to AI Behavior Tree

