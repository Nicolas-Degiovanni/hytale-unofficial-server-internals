---
description: Architectural reference for PlaceBlockEvent
---

# PlaceBlockEvent

**Package:** com.hypixel.hytale.server.core.event.events.ecs
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class PlaceBlockEvent extends CancellableEcsEvent {
```

## Architecture & Concepts
The PlaceBlockEvent is a fundamental message within the server's Entity Component System (ECS) event bus. It represents a player's *intent* to place a block in the world, not the completed action itself. This distinction is critical for creating a modifiable and secure game environment.

By extending CancellableEcsEvent, this class becomes a key interception point for game logic. Systems can listen for this event and choose to veto the action entirely (e.g., for land protection), modify its parameters (e.g., snap the block to a grid), or log the activity.

This event-driven approach decouples the source of the action (player input) from the systems that process and execute it (world modification, inventory management).

### Lifecycle & Ownership
- **Creation:** Instantiated by a high-level server system, typically one responsible for processing player interaction packets. It is created in direct response to a client request to place a block.
- **Scope:** Extremely short-lived. An instance of PlaceBlockEvent exists only for the duration of the event dispatch cycle within a single server tick. It is a fire-and-forget message.
- **Destruction:** The object is eligible for garbage collection immediately after the ECS event bus has finished notifying all subscribed listeners. No system should maintain a long-term reference to an event object.

## Internal State & Concurrency
- **State:** This object is **mutable**. Core properties like the target block position and rotation can be altered by any listener in the event chain. This allows for powerful, emergent game mechanics where multiple systems can influence the final outcome of a single action.

- **Thread Safety:** **Not thread-safe.** This event is designed to be created, dispatched, and handled exclusively on the main server thread within a single game tick. Accessing or modifying an instance of this event from an asynchronous task or a different thread will result in race conditions, data corruption, and severe server instability.

## API Surface
The primary interaction surface includes getters for initial state and setters for in-flight modification. The most critical API, `cancel()`, is inherited from CancellableEcsEvent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItemInHand() | ItemStack | O(1) | Retrieves the item stack used for placement. May be null. |
| getTargetBlock() | Vector3i | O(1) | Retrieves the target world coordinate for the new block. |
| setTargetBlock(pos) | void | O(1) | **Warning:** Modifies the event. Allows a listener to redirect the placement. |
| getRotation() | RotationTuple | O(1) | Retrieves the desired orientation of the new block. |
| setRotation(rot) | void | O(1) | **Warning:** Modifies the event. Allows a listener to change the final block rotation. |

## Integration Patterns

### Standard Usage
The standard pattern is to create an ECS system that subscribes to the event. The system can then inspect the event's properties and either cancel it or allow it to proceed, potentially after modifying it.

```java
// In an ECS System responsible for world protection
@Subscribe
public void onBlockPlaceAttempt(PlaceBlockEvent event) {
    Player placingPlayer = event.getPlayer(); // Assumes a method to get the actor
    Vector3i target = event.getTargetBlock();

    if (!WorldProtectionManager.canBuild(placingPlayer, target)) {
        // Veto the action completely
        event.cancel();
        
        // Optionally send a feedback message to the player
        placingPlayer.sendMessage("You do not have permission to build here.");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Asynchronous Handling:** Do not store a reference to the event to process it later in an asynchronous task. The event object is invalid after the initial tick's dispatch.
- **Manual Instantiation:** Do not create instances of PlaceBlockEvent using `new`. This bypasses the standard player input pipeline and can lead to desynchronization. Events should only be fired by the core engine systems responsible for them.
- **Assuming Finality:** When handling this event, do not assume its properties are final. A listener with a different priority may have already modified the target position or rotation. Always read the state when you receive the event.

## Data Pipeline
The PlaceBlockEvent serves as a structured data container that flows from network input to world state modification.

> Flow:
> Client Input -> Network Packet -> Server Packet Handler -> **PlaceBlockEvent** (Fired on ECS Bus) -> Listener Systems (Validation, Modification, Logging) -> World State System (Consumes non-cancelled event) -> Block placed in world data structure

