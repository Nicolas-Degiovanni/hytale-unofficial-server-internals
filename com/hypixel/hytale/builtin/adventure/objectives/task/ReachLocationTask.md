---
description: Architectural reference for ReachLocationTask
---

# ReachLocationTask

**Package:** com.hypixel.hytale.builtin.adventure.objectives.task
**Type:** Transient State Object

## Definition
```java
// Signature
public class ReachLocationTask extends ObjectiveTask {
```

## Architecture & Concepts
The ReachLocationTask is a concrete implementation of the ObjectiveTask contract, designed to track a player's progress toward a specific, named location within the world. It serves as the runtime state machine for a "go to location" quest step.

Architecturally, this class acts as a bridge between the abstract quest system (the parent Objective) and the concrete world entity system (the EntityStore). It does not store hardcoded coordinates. Instead, it is configured with a logical identifier, `targetLocationId`, defined in its corresponding ReachLocationTaskAsset.

Upon activation, the task dynamically queries the world's EntityStore to find an entity that possesses a ReachLocationMarker component matching this identifier. It then calculates the closest matching entity and creates a client-side map marker, guiding the player. The task's lifecycle is managed entirely by its parent Objective, which is responsible for its creation, serialization, and event propagation.

The presence of a static CODEC field indicates that this class is designed for persistence. Its state, primarily the `completed` flag, is serialized and deserialized as part of the parent Objective's data, allowing quest progress to be saved and loaded.

## Lifecycle & Ownership
- **Creation:** An instance of ReachLocationTask is created by the adventure mode objective system when a parent Objective containing this task type is activated for a player or group. The parent Objective is the sole owner of the task instance.
- **Scope:** The instance persists as long as its parent Objective is active. Its state is serialized along with the Objective, making it durable across server restarts or player sessions.
- **Destruction:** The object is eligible for garbage collection when the parent Objective is completed, failed, or otherwise removed from the system. There is no explicit destruction method; its lifetime is bound to its owner.

## Internal State & Concurrency
- **State:** This object is mutable and stateful. Its primary state is tracked by two boolean flags:
    - **completed:** Tracks whether the player has reached the destination.
    - **markerLoaded:** Tracks whether the map marker has been successfully identified and sent to the client.
    It does not cache world data, instead querying the EntityStore on demand during setup.

- **Thread Safety:** **This class is not thread-safe.** All methods that interact with the world state, such as `setup0` and `onPlayerReachLocationMarker`, must be executed on the server's main world-tick thread. Unsynchronized access from other threads will result in race conditions, data corruption, and unpredictable behavior. Concurrency is managed by the calling game loop.

## API Surface
The public API is primarily intended for internal use by the objective system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup0(objective, world, store) | TransactionRecord[] | O(N) | Scans all world entities with a ReachLocationMarker component to find the closest target. High-cost operation that should only be called once upon task activation. |
| onPlayerReachLocationMarker(store, ref, locationMarkerId, objective) | void | O(1) | Event handler called by the engine when a player interacts with a location marker. Triggers the completion logic. |
| checkCompletion() | boolean | O(1) | Returns the current completion state. |
| toPacket(objective) | com.hypixel.hytale.protocol.ObjectiveTask | O(1) | Serializes the task's current state into a network packet for client-side UI updates. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is managed by the parent Objective. The engine's event system is responsible for invoking its methods in the correct sequence.

1.  An Objective is activated, which instantiates ReachLocationTask.
2.  The Objective calls `setup0` to find the target and create a map marker.
3.  The player moves to the location, triggering a world event.
4.  The event system invokes `onPlayerReachLocationMarker` on the task instance.
5.  The task updates its state and calls back into the parent Objective to notify it of completion.

```java
// Conceptual example of the engine's interaction
// THIS CODE DOES NOT EXIST AND IS FOR ILLUSTRATION ONLY

// On objective activation:
Objective objective = ...;
ReachLocationTask task = objective.getTask(ReachLocationTask.class);
task.setup0(objective, world, store);

// When a player enters a trigger volume:
// Engine dispatches an event to the objective system
String markerId = "some_location_id";
objective.onPlayerEvent(new PlayerReachedLocationEvent(playerRef, markerId));

// The objective system then routes the event to the task:
task.onPlayerReachLocationMarker(store, playerRef, markerId, objective);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ReachLocationTask()`. Instances must be created and managed by the Objective framework to ensure proper state tracking and persistence.
- **Manual State Modification:** Do not set the `completed` field directly. This bypasses critical completion logic, such as sending network packets, consuming conditions, and notifying the parent Objective, leaving the system in an inconsistent state.
- **Calling `setup0` Repeatedly:** This method performs an expensive world query. Calling it more than once per task activation will cause unnecessary server load.

## Data Pipeline
The flow of data and events for this task follows two distinct paths: activation and completion.

> **Activation Flow:**
> Objective Activation -> `ReachLocationTask.setup0()` -> Scans `EntityStore` for `ReachLocationMarker` -> Identifies closest target entity -> `addMarker(new MapMarker(...))` -> Map Marker data sent to Client

> **Completion Flow:**
> Player enters world trigger -> Engine fires internal event -> `Objective.onPlayerReachLocationMarker()` -> State changes (`completed = true`) -> `Objective.markDirty()` -> `sendUpdateObjectiveTaskPacket()` -> Client UI updates -> `Objective.checkTaskSetCompletion()`

