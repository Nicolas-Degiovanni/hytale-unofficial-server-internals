---
description: Architectural reference for DrainPlayerFromWorldEvent
---

# DrainPlayerFromWorldEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Transient

## Definition
```java
// Signature
public class DrainPlayerFromWorldEvent implements IEvent<String> {
```

## Architecture & Concepts
The DrainPlayerFromWorldEvent is a server-side Data Transfer Object (DTO) that signals the initiation of a player's removal from a specific World instance. It is a critical component in the server's world transition pipeline, used for processes like moving a player between zones, transferring to a minigame server, or handling disconnections.

The term **Drain** is intentional and significant. This event does not signify player data deletion; rather, it instructs listening systems to extract or "drain" the player's state from the active simulation of the source World. The event encapsulates all necessary context for this operation: the player's entity data (via a Holder to an EntityStore), the source World, and the player's final Transform (position and rotation) within that world.

This event acts as a decoupling mechanism. The system that triggers the player's departure (e.g., a portal interaction handler) does not need to know the low-level details of how a player's entity is persisted or detached from the world's entity manager. It simply fires this event, and dedicated systems handle the complex state management.

## Lifecycle & Ownership
- **Creation:** Instantiated by high-level server logic when a player is scheduled to leave a world. Common triggers include portal activation, server shutdown sequences, or administrative commands.
- **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch through the server's event bus. It is designed to be a fire-and-forget message.
- **Destruction:** The object becomes eligible for garbage collection immediately after the event bus has finished notifying all subscribers. No system should retain a reference to this event post-dispatch.

## Internal State & Concurrency
- **State:** **Mutable**. This is a critical design characteristic. The event's target World and Transform can be modified after instantiation via the `setWorld` and `setTransform` methods. This allows event listeners to intercept and potentially redirect a player's destination mid-process. While flexible, this introduces complexity and potential for race conditions if not handled carefully.

- **Thread Safety:** **Not thread-safe**. The internal state is mutable and not protected by any synchronization mechanisms. It is assumed that the event bus dispatches events on a single, controlled thread (e.g., the main server tick thread). Concurrent modification from multiple threads will lead to unpredictable behavior.

## API Surface
The public API is designed for data transport and in-flight modification.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHolder() | Holder<EntityStore> | O(1) | Retrieves the reference holder for the player's entity data. |
| getWorld() | World | O(1) | Retrieves the source World from which the player is being drained. |
| setWorld(world) | void | O(1) | **Warning:** Mutates the event. Overwrites the world context. |
| getTransform() | Transform | O(1) | Retrieves the player's last known position, rotation, and scale. |
| setTransform(transform) | void | O(1) | **Warning:** Mutates the event. Overwrites the transform context. |

## Integration Patterns

### Standard Usage
This event is created and immediately posted to the server's event bus. A listener, such as a `WorldManagerService`, will then handle the logic of detaching the player's entity from the world's simulation loop.

```java
// Executed by a high-level service (e.g., PortalService)

// 1. Gather required context for the player being moved
Holder<EntityStore> playerEntityStore = player.getEntityStoreHolder();
World currentWorld = player.getWorld();
Transform finalTransform = player.getTransform();

// 2. Create the event DTO
DrainPlayerFromWorldEvent event = new DrainPlayerFromWorldEvent(
    playerEntityStore,
    currentWorld,
    finalTransform
);

// 3. Dispatch the event for processing by other systems
server.getEventBus().post(event);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not cache or reuse an instance of this event. It is a transient DTO representing a unique, point-in-time operation.
- **Unmanaged Mutation:** Do not have multiple, independent systems listening for and calling `setWorld` or `setTransform` on the same event. This creates a "last write wins" scenario that is difficult to debug. If redirection logic is needed, it should be handled by a single, authoritative system with a high listener priority.

## Data Pipeline
This event is the first stage in the data flow for transferring a player entity out of an active world simulation.

> Flow:
> Game Logic Trigger (e.g., Portal Use) -> **DrainPlayerFromWorldEvent** (Creation) -> Server Event Bus (Dispatch) -> WorldManager (Listener) -> Player Entity detached from World Tick -> Next Stage (e.g., PlayerSaveEvent or InjectPlayerIntoWorldEvent)

