---
description: Architectural reference for EventNotification
---

# EventNotification

**Package:** com.hypixel.hytale.server.npc.blackboard.view.event
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class EventNotification {
```

## Architecture & Concepts
The EventNotification class is a fundamental data structure within the server-side NPC Artificial Intelligence framework. It functions as a lightweight, transient data carrier, designed to communicate occurrences within the game world to an NPC's sensory and decision-making systems.

Its primary role is to decouple event producers (e.g., the physics engine reporting a collision, the sound system propagating a noise) from event consumers (the NPC's AI logic). By encapsulating the core details of an event—its location, its cause, and its type—it provides a standardized message format for the AI's perception layer.

This class is a key component of the **Blackboard** architectural pattern used by the Hytale NPC AI. Event producers write EventNotification objects to an NPC's blackboard, which acts as a shared memory space. The NPC's behavior tree or state machine then reads these notifications from the blackboard to make informed decisions about how to react to its environment.

### Lifecycle & Ownership
-   **Creation:** EventNotification instances are created on-demand by various server systems whenever an action occurs that might be relevant to an NPC. It is highly probable that these objects are managed by an object pool to mitigate garbage collection overhead in performance-critical loops.
-   **Scope:** The lifetime of an EventNotification is extremely short, typically confined to a single server tick. It is created, populated, passed to the relevant AI systems, processed, and then becomes immediately eligible for garbage collection or is returned to its object pool.
-   **Destruction:** There is no explicit destruction logic. Ownership is transferred from the event producer to the AI system, which is expected to discard its reference after processing is complete.

## Internal State & Concurrency
-   **State:** This object is entirely mutable. Its fields are intended to be populated by a producer system immediately after instantiation. The internal Vector3d object is also mutable, and its state is modified via the setPosition method.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single, synchronous game loop or world update thread. Concurrent modification from multiple threads will lead to race conditions and unpredictable AI behavior. All access must be externally synchronized if multi-threaded processing is required, though this is strongly discouraged.

## API Surface
The public API is composed entirely of simple getters and setters for populating and reading event data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPosition() | Vector3d | O(1) | Returns a direct, mutable reference to the event's position. |
| setPosition(x, y, z) | void | O(1) | Sets the world-space coordinates of the event. |
| getInitiator() | Ref<EntityStore> | O(1) | Retrieves a reference to the entity that caused the event. |
| setInitiator(initiator) | void | O(1) | Assigns the event's source entity. |
| getSet() | int | O(1) | Retrieves an integer identifier, likely representing a category or type of event. |
| setSet(set) | void | O(1) | Assigns the event's type identifier. |

## Integration Patterns

### Standard Usage
The canonical use case involves a system creating, populating, and dispatching the event to an NPC's blackboard for processing within the same tick.

```java
// Example: A sound system notifies nearby NPCs
// 1. Obtain an EventNotification instance (likely from a pool)
EventNotification notification = new EventNotification();

// 2. Populate the event data
notification.setPosition(soundSource.x, soundSource.y, soundSource.z);
notification.setInitiator(playerRef);
notification.setSet(SoundEvents.FOOTSTEP);

// 3. Dispatch to the relevant NPC's blackboard
npc.getBlackboard().postEvent(notification);
```

### Anti-Patterns (Do NOT do this)
-   **Retaining References:** Do not hold a reference to an EventNotification object beyond the current game tick. They are ephemeral and may be recycled by an object pool, leading to state corruption.
-   **Cross-Thread Sharing:** Never pass an EventNotification instance between threads without explicit and robust synchronization. The class is not designed for concurrent access.
-   **External State Mutation:** The getPosition method returns a direct reference to the internal Vector3d. Modifying this returned vector will alter the state of the EventNotification itself. This can cause severe, difficult-to-debug issues if multiple systems process the same event.

    ```java
    // WARNING: DANGEROUS MODIFICATION
    EventNotification event = blackboard.getLatestEvent();
    Vector3d pos = event.getPosition();

    // This next line modifies the original event object's state,
    // potentially affecting other AI components.
    pos.add(1, 0, 0);
    ```

## Data Pipeline
The EventNotification serves as the data payload in the NPC perception pipeline.

> Flow:
> World Action (e.g., Player fires weapon) -> Game System (e.g., ProjectileSystem) -> **EventNotification** (Instantiation & Population) -> NPC Blackboard -> AI Behavior Tree (Consumption) -> NPC Reaction (e.g., Take cover)

