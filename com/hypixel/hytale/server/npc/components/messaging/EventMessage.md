---
description: Architectural reference for EventMessage
---

# EventMessage

**Package:** com.hypixel.hytale.server.npc.components.messaging
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class EventMessage extends NPCMessage {
```

## Architecture & Concepts
The EventMessage class is a specialized, transient data structure used within the server-side NPC messaging system. It represents a spatial event that has occurred at a specific world coordinate, intended to be broadcast to other NPCs within a defined radius.

This class is a fundamental component of NPC group behavior and environmental awareness. It allows one NPC to communicate significant occurrences—such as spotting a player, hearing a sound, or reaching a patrol point—to its peers. The system uses this data to trigger coordinated responses, such as flocking, fleeing, or attacking.

The design heavily implies the use of an object pooling pattern. The existence of an *activate* method, which re-initializes the object's state, is a strong indicator that these objects are recycled to minimize garbage collection overhead in the main game loop.

## Lifecycle & Ownership
- **Creation:** EventMessage instances are not meant to be instantiated directly by gameplay logic. They are typically acquired from a central message pool or factory managed by the NPC messaging service. The *clone* method provides a secondary path for creating a new instance from an existing one.
- **Scope:** The lifetime of an EventMessage is extremely short, typically confined to a single game tick. It is created, populated, dispatched, and processed within one cycle of the NPC AI update loop.
- **Destruction:** Instances are not explicitly destroyed. After processing, they are expected to be released back to the object pool they were acquired from. Failure to do so will result in object leaks and performance degradation.

## Internal State & Concurrency
- **State:** The class is highly **mutable**. Its primary purpose is to be configured at runtime via the *activate* method. The internal *position* Vector3d is mutable, and the *getPosition* method returns a direct reference to this internal object.

- **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access within the server's main update loop or a world-specific AI thread. There are no internal locks or synchronization mechanisms. Concurrent modification from multiple threads will lead to unpredictable behavior and data corruption.

    **WARNING:** The *getPosition* method returns a direct, mutable reference to the internal Vector3d. Modifying this returned vector will alter the state of the EventMessage itself. This is a deliberate performance optimization to avoid object allocation but requires careful handling by the caller.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| activate(x, y, z, target, age) | void | O(1) | Re-initializes the message for reuse, setting its core spatial and target parameters. This is the primary entry point for configuring a recycled message object. |
| clone() | EventMessage | O(1) | Creates a new EventMessage instance with a deep copy of the current state. |
| getPosition() | Vector3d | O(1) | Returns a direct reference to the internal position vector. See warning on thread safety. |
| getMaxRangeSquared() | double | O(1) | Returns the pre-calculated squared maximum range for the event broadcast. |
| setSameFlock(boolean) | void | O(1) | Sets a flag indicating if this message should only be processed by NPCs in the same flock. |

## Integration Patterns

### Standard Usage
An AI behavior component retrieves a message from a pool, activates it with event-specific data, and dispatches it through a central messaging system.

```java
// Assume 'messageBus' and 'npc' are provided by the AI system context
// 1. Acquire a message from a pool (conceptual)
EventMessage msg = messageBus.acquireMessage(EventMessage.class);

// 2. Configure the message with event data
Vector3d npcPosition = npc.getPosition();
Ref<EntityStore> targetRef = npc.getCurrentTarget();
msg.activate(npcPosition.x, npcPosition.y, npcPosition.z, targetRef, world.getAge());
msg.setSameFlock(true);

// 3. Dispatch the message for processing by other NPCs
messageBus.dispatch(msg);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new EventMessage()`. This bypasses the object pooling system, leading to unnecessary garbage collection and performance penalties. Always acquire instances from the appropriate manager or service.
- **State Leak Modification:** Do not modify the object returned by *getPosition* unless you are fully aware of the side effects. This can lead to subtle and hard-to-debug issues in other systems processing the same message instance.
    ```java
    // BAD: This modifies the message's internal state unexpectedly
    EventMessage msg = ...;
    Vector3d pos = msg.getPosition();
    pos.setX(0); // The message's position is now changed
    ```
- **State Retention:** Do not hold references to an EventMessage instance across multiple game ticks. They are designed to be recycled immediately after the tick in which they are dispatched.

## Data Pipeline
The EventMessage serves as a data payload that flows from an event source to multiple NPC listeners.

> Flow:
> NPC Behavior (e.g., OnTargetSpotted) -> Message Pool Request -> **EventMessage** (activated) -> Message Dispatcher -> Spatial Query (finds nearby NPCs) -> Target NPC Behavior (e.g., OnAllyAlert) -> AI State Update

