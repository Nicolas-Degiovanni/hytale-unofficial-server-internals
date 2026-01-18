---
description: Architectural reference for SensorPath
---

# SensorPath

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorPath extends SensorBase {
```

## Architecture & Concepts
The SensorPath is a fundamental component within the server-side Non-Player Character (NPC) Artificial Intelligence framework. It functions as a "sensor", responsible for detecting and evaluating paths in the game world relative to an NPC's position and state. Its primary purpose is to answer the question: "Is there a suitable path nearby that I should be aware of or follow?"

This component acts as the bridge between the abstract concept of a path (defined by designers in prefabs or the world) and the concrete decision-making logic of an NPC. When its conditions are met—for instance, an NPC is within a specified range of a valid path—the sensor's **matches** method returns true. This signals to the AI's behavior tree or state machine that a path-related action can be initiated.

SensorPath operates in several distinct modes, configured via the **PathType** enumeration:
*   **WorldPath:** Searches for a globally defined, named path within the world configuration.
*   **CurrentPrefabPath:** Searches for a path specifically within the prefab instance the NPC belongs to. This is crucial for creating self-contained, reusable points of interest.
*   **AnyPrefabPath:** Performs a spatial query to find the nearest path marker from *any* nearby prefab, allowing for more dynamic and emergent interactions between world geometry and NPCs.
*   **TransientPath:** Listens for a temporary path assigned programmatically, typically for quests, commands, or debugging.

Upon successfully identifying a path, the SensorPath updates the NPC's **PathManager** component, effectively telling the NPC which path it should now consider active. It also populates its internal **InfoProvider** with contextual data, such as the coordinates of the closest waypoint, for consumption by subsequent AI actions.

## Lifecycle & Ownership
- **Creation:** A SensorPath instance is not created directly. It is instantiated by the NPC asset loading pipeline when parsing an NPC's definition file. A corresponding **BuilderSensorPath** object reads configuration data (e.g., range, path name, path type) and constructs the sensor as part of an NPC's **Role**.
- **Scope:** The lifecycle of a SensorPath is tightly coupled to the lifecycle of the NPC entity that owns it. It persists as long as the NPC exists in the world.
- **Destruction:** The object is eligible for garbage collection when its parent NPC entity is unloaded from the world or destroyed. There is no manual destruction method.

## Internal State & Concurrency
- **State:** SensorPath is a stateful, mutable component. It maintains significant internal state between ticks to optimize performance and track context. This includes:
    - The location of the closest waypoint found (**closestWaypoint**).
    - A set of paths to ignore (**disallowedPaths**), which is critical when an NPC is instructed to find a *new* path.
    - The last known revision of the world's path data (**pathChangeRevision**) to avoid redundant searches.
    - The loading status of a path (**loadStatus**), as paths may be streamed in and not immediately available.

- **Thread Safety:** **WARNING:** This class is not thread-safe and is designed for single-threaded access. It must only be accessed and manipulated from the primary server thread responsible for ticking the NPC's world. Its methods rely on thread-local caches (e.g., **SpatialResource.getThreadLocalReferenceList**) and assume a non-concurrent modification of the entity-component store. Unsynchronized access from other threads will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is minimal, centered on the standard Sensor interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | The core evaluation method. Returns true if a valid path is found within range. Complexity depends on the PathType, ranging from map lookups to spatial queries over N nearby objects. |
| getSensorInfo() | InfoProvider | O(1) | Returns a provider object containing data from the last successful match, such as the target waypoint position. Throws an exception if called without a preceding successful match. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly by most game logic. It is configured within an NPC asset file and is invoked automatically by the NPC's AI processing loop. An AI behavior might then query the sensor's results to make a decision.

```java
// This logic is representative of an AI Behavior, not direct usage.
// The AI system calls sensor.matches() automatically.
// If it returns true, a behavior like this might run.

InfoProvider info = sensor.getSensorInfo();
if (info instanceof PositionProvider) {
    PositionProvider positionProvider = (PositionProvider) info;
    Vector3d targetWaypoint = positionProvider.getTarget();
    
    // The NPC's navigation system would now be instructed
    // to move towards the targetWaypoint.
    npc.getNavigationComponent().setMoveTarget(targetWaypoint);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call **new SensorPath()**. The component's state and configuration are complex and must be injected by the asset loading system via its corresponding builder. Direct creation will result in a non-functional sensor.
- **State Manipulation:** Do not attempt to modify the internal state of the sensor from outside the class. For example, clearing **disallowedPaths** manually will break the logic for finding new paths.
- **Asynchronous Execution:** Do not call the **matches** method from a separate thread or asynchronous task. All interactions must be on the main world tick thread.

## Data Pipeline
The flow of data begins with the NPC's state and ends with either a boolean decision or populated data providers for use by other AI systems.

> Flow:
> NPC AI Tick -> **SensorPath.matches()** -> World State Query (SpatialResource or WorldPathData) -> Path Candidate Found -> Closest Waypoint Calculated -> Range Check -> **PathManager** Updated -> **PositionProvider** Populated -> Boolean Result Returned

