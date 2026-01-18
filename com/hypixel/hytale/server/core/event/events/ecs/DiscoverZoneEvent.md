---
description: Architectural reference for DiscoverZoneEvent
---

# DiscoverZoneEvent

**Package:** com.hypixel.hytale.server.core.event.events.ecs
**Type:** Event / Data Transfer Object

## Definition
```java
// Signature
public abstract class DiscoverZoneEvent extends EcsEvent {
    // ...
    public static class Display extends DiscoverZoneEvent implements ICancellableEcsEvent {
        // ...
    }
}
```

## Architecture & Concepts
DiscoverZoneEvent is a message-based event that signals a significant moment in world exploration: when a player entity first enters a named geographical zone. As a subclass of EcsEvent, it is designed to be dispatched through the server-side Entity Component System event bus.

This event acts as a decoupling mechanism. The system responsible for tracking player position and map data, the WorldMapTracker, can broadcast that a discovery has occurred without needing to know which other systems must react. Systems concerned with achievements, UI notifications, or statistical tracking can subscribe to this event and execute their logic independently.

The architecture uses an abstract base class to define the core data payload, while concrete inner classes like **Display** represent specific, actionable stages of the event. The **Display** subclass is cancellable, providing a critical interception point for game logic to suppress the standard UI notification under specific conditions, such as during a cinematic or if the player is in a state where such notifications are undesirable.

### Lifecycle & Ownership
-   **Creation:** An instance of a concrete subclass, typically DiscoverZoneEvent.Display, is created by the WorldMapTracker system. This occurs during the game tick when the system's logic determines a player entity has crossed into a new, previously unvisited zone.
-   **Scope:** Transient and extremely short-lived. The event object exists only for the duration of its propagation through the ECS event bus within a single server tick.
-   **Destruction:** The object becomes eligible for garbage collection immediately after the event bus has finished notifying all subscribers for the current tick. There is no manual memory management. **WARNING:** Holding a reference to this event beyond the scope of an event handler is a memory leak anti-pattern.

## Internal State & Concurrency
-   **State:** Primarily immutable. The core payload, ZoneDiscoveryInfo, is stored in a final field and cannot be changed after construction. The concrete subclass DiscoverZoneEvent.Display introduces a single piece of mutable state: the boolean *cancelled* flag.
-   **Thread Safety:** **Not thread-safe.** This event is designed to be created, dispatched, and handled exclusively on the main server thread. Any attempt to access or modify an instance from a worker thread will result in race conditions and undefined behavior. All interactions must be synchronized with the server's main game loop.

## API Surface
The public contract is minimal, focusing on data retrieval and cancellation control.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDiscoveryInfo() | WorldMapTracker.ZoneDiscoveryInfo | O(1) | Retrieves the immutable data payload containing details about the zone discovery. |
| isCancelled() | boolean | O(1) | (Display subclass only) Returns true if the event has been marked for cancellation. |
| setCancelled(boolean) | void | O(1) | (Display subclass only) Sets the cancellation flag. Consuming systems should respect this flag. |

## Integration Patterns

### Standard Usage
The intended use is to subscribe to this event within an ECS System to react to a zone discovery. The handler should first check if the event has already been cancelled by a higher-priority system.

```java
// Inside an ECS System that handles player notifications
@Subscribe
public void onZoneDiscovered(DiscoverZoneEvent.Display event) {
    if (event.isCancelled()) {
        return;
    }

    // Retrieve player and zone info
    EntityId playerId = event.getDiscoveryInfo().getPlayerId();
    String zoneName = event.getDiscoveryInfo().getZone().getDisplayName();

    // Send a packet to the client to display the "Zone Discovered" UI
    PacketSender.send(playerId, new ZoneDiscoveryPacket(zoneName));
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Instantiation:** Systems should not create and post their own DiscoverZoneEvent instances. Doing so can lead to a desynchronization of the player's actual map discovery state, which is owned and managed by the WorldMapTracker.
-   **Ignoring Cancellation:** Failing to check the isCancelled flag before acting on the event can violate game logic rules imposed by other systems, leading to bugs like UI popups appearing during cutscenes.
-   **Asynchronous Processing:** Do not dispatch this event to a separate thread for processing. It must be handled synchronously within the event callback to maintain a consistent world state for the current tick.

## Data Pipeline
The flow of data begins with player movement and ends with a client-side effect, with DiscoverZoneEvent serving as the central message.

> Flow:
> Player Entity Movement -> WorldMapTracker Logic -> **new DiscoverZoneEvent.Display()** -> ECS Event Bus -> Subscribed Systems (e.g., UINotificationSystem) -> Network Packet Sent to Client -> Client Renders UI Notification

