---
description: Architectural reference for PlayerGroupEvent
---

# PlayerGroupEvent

**Package:** com.hypixel.hytale.server.core.event.events.permissions
**Type:** Transient Data Object

## Definition
```java
// Signature
public class PlayerGroupEvent extends PlayerPermissionChangeEvent {

    // Nested specialization classes
    public static class Added extends PlayerGroupEvent { /* ... */ }
    public static class Removed extends PlayerGroupEvent { /* ... */ }
}
```

## Architecture & Concepts

The PlayerGroupEvent is a foundational data structure within the server's event-driven architecture. It is not a service or manager, but rather an immutable **message** that represents a discrete change to a player's group membership.

This class is a specialization of the more generic PlayerPermissionChangeEvent, creating a clear and logical event hierarchy. This design allows systems to subscribe to events with varying levels of specificity: a system can listen for *any* permission change via the parent type, or it can listen for only group-related changes by subscribing to PlayerGroupEvent.

The most significant architectural pattern employed here is the use of static nested subclasses: **Added** and **Removed**. This powerful technique replaces the need for internal state flags (e.g., an enum for event type). It enables the event bus to perform type-based dispatch, allowing listeners to subscribe directly to the precise event they care about. This results in cleaner, more performant, and more type-safe handler code, eliminating the need for `instanceof` checks in consumer logic.

## Lifecycle & Ownership

- **Creation:** Instantiated exclusively by the server's core permission management system when a player is successfully added to or removed from a group. This is typically triggered by an administrative command or an internal system process.
- **Scope:** Ephemeral. The object's lifecycle is bound to its dispatch cycle within the event bus. It is created, passed to all relevant subscribers, and then becomes eligible for garbage collection.
- **Destruction:** Managed automatically by the Java Garbage Collector. There are no native resources or explicit cleanup methods associated with this object.

## Internal State & Concurrency

- **State:** **Immutable**. All fields, including the inherited playerUuid and the local groupName, are assigned at construction and cannot be modified thereafter. This is a critical design choice for event objects, as it guarantees that the event's data remains consistent as it is passed between different systems and potentially across different threads.
- **Thread Safety:** Inherently **thread-safe**. Due to its immutability, an instance of PlayerGroupEvent can be safely read by multiple threads concurrently without any need for locks or other synchronization primitives. This is essential for the performance and stability of the server's multi-threaded event bus.

## API Surface

The public contract is minimal, focusing solely on data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGroupName() | String | O(1) | Returns the non-null name of the group involved in the event. |
| getPlayerUuid() | UUID | O(1) | *Inherited.* Returns the UUID of the player whose groups have changed. |

## Integration Patterns

### Standard Usage

The canonical use of this class is through subscription within the server's event bus. Systems that need to react to group changes will define a handler method annotated to receive a specific subtype.

```java
// Example: A service that updates a player's display name on group change
public class PlayerDisplayNameService {

    @Subscribe
    public void onPlayerGroupAdded(PlayerGroupEvent.Added event) {
        // This method is ONLY invoked when a player is added to a group
        Player target = Server.getPlayer(event.getPlayerUuid());
        if (target != null) {
            target.updateDisplayName();
        }
    }

    @Subscribe
    public void onPlayerGroupRemoved(PlayerGroupEvent.Removed event) {
        // This method is ONLY invoked when a player is removed from a group
        Player target = Server.getPlayer(event.getPlayerUuid());
        if (target != null) {
            target.updateDisplayName();
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

- **Manual Instantiation:** Never create an instance of PlayerGroupEvent or its subtypes directly in gameplay or plugin code. These events are the authoritative record of a change from the permission system. Manually firing them can lead to a desynchronized state across the server.
- **Listening to the Base Type:** While possible, listening for the base `PlayerGroupEvent` and then using `instanceof` to determine the type is verbose and inefficient. It defeats the purpose of the type-safe dispatch provided by the `Added` and `Removed` subclasses.

    ```java
    // AVOID THIS PATTERN
    @Subscribe
    public void onPlayerGroupChange(PlayerGroupEvent event) {
        if (event instanceof PlayerGroupEvent.Added) {
            // ... logic for added
        } else if (event instanceof PlayerGroupEvent.Removed) {
            // ... logic for removed
        }
    }
    ```

## Data Pipeline

The PlayerGroupEvent is a data packet that flows from a state-changing action to various reactive systems.

> Flow:
> Administrative Command (`/group add ...`) -> PermissionService -> **PlayerGroupEvent.Added** -> Server EventBus -> Subscribed Listeners (e.g., ChatService, ZoneController, PlayerDataPersistence)

