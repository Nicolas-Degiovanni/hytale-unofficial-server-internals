---
description: Architectural reference for PlayerPermissionChangeEvent
---

# PlayerPermissionChangeEvent

**Package:** com.hypixel.hytale.server.core.event.events.permissions
**Type:** Event / Data Transfer Object

## Definition
```java
// Signature
public abstract class PlayerPermissionChangeEvent implements IEvent<Void> {
    // ... nested static classes for specific events
}
```

## Architecture & Concepts

The PlayerPermissionChangeEvent class family serves as the authoritative notification mechanism for all modifications to a player's permissions on the server. It is not a service or manager, but rather an immutable data record representing a state change that has already occurred.

Architecturally, this class acts as an abstract base for a set of more specific, concrete event types implemented as nested static classes (e.g., GroupAdded, PermissionsRemoved). This design pattern allows systems to subscribe to either the generic, high-level PlayerPermissionChangeEvent to catch all permission-related modifications, or to a specific subclass for more granular control.

By implementing the IEvent interface, these objects are designed to be dispatched through the central server EventBus. This decouples the permission management system, which produces these events, from the various consumer systems (e.g., chat, gameplay mechanics, command validation) that need to react to permission changes. The use of the Void generic type in IEvent<Void> signifies that this is a one-way notification; handlers are not expected to return a value or modify the event's outcome.

## Lifecycle & Ownership

-   **Creation:** A concrete subclass, such as PlayerPermissionChangeEvent.GroupAdded, is instantiated by the core permission management service at the exact moment a player's permissions are successfully altered. This typically occurs in response to an administrative command or an API call. The abstract base class is never instantiated directly.
-   **Scope:** These event objects are extremely short-lived and transient. Their lifecycle is confined to the duration of their dispatch through the EventBus. They are fire-and-forget messages.
-   **Destruction:** Once the EventBus has finished broadcasting the event to all registered listeners, the object is no longer referenced by any core system and becomes immediately eligible for garbage collection. No system should ever retain a long-term reference to an event object.

## Internal State & Concurrency

-   **State:** Strictly **immutable**. All internal fields, including the playerUuid and the specific data within each subclass, are declared as final. Collections, such as the set of permissions in PermissionsAdded, are defensively wrapped in an unmodifiable view before being exposed through getters.

-   **Thread Safety:** This class and its subclasses are inherently **thread-safe**. Their immutability guarantees that they can be safely published, read, and passed between different threads without any risk of data corruption or race conditions. No external locking or synchronization is required when handling these events.

    **WARNING:** Any attempt to modify a collection returned by a getter (e.g., getAddedPermissions) will result in an UnsupportedOperationException at runtime. This is an intentional design choice to enforce immutability.

## API Surface

The public contract is focused on providing read-only access to the event data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayerUuid() | UUID | O(1) | Returns the unique identifier of the player whose permissions changed. |
| getGroupName() | String | O(1) | **(Subclasses only)** Returns the name of the group added or removed. |
| getAddedPermissions() | Set<String> | O(1) | **(Subclasses only)** Returns an unmodifiable view of the permissions that were added. |
| getRemovedPermissions() | Set<String> | O(1) | **(Subclasses only)** Returns an unmodifiable view of the permissions that were removed. |

## Integration Patterns

### Standard Usage

The sole intended use of this class is to be received as a parameter in a method subscribed to the server's EventBus. Systems react to the data contained within the event.

```java
// A listener in a hypothetical system that needs to update a player's state
@Subscribe
public void onPlayerGroupAdded(PlayerPermissionChangeEvent.GroupAdded event) {
    UUID playerId = event.getPlayerUuid();
    String group = event.getGroupName();

    // Re-evaluate player capabilities or update UI elements
    Player targetPlayer = world.getPlayer(playerId);
    if (targetPlayer != null) {
        targetPlayer.recalculateStateForNewGroup(group);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation by Consumers:** Systems should only *consume* these events. Never create and post a PlayerPermissionChangeEvent from a non-permission-authority system, as this will lead to a desynchronized and inconsistent server state.
-   **Attempting to Veto the Change:** This event is a report of something that has *already happened*. It is not a cancellable "pre-event". Do not use it to implement logic that attempts to prevent a permission change.
-   **State Modification:** Do not attempt to cast the returned unmodifiable sets to a mutable type. This violates the architectural contract and is not guaranteed to work across future updates.

## Data Pipeline

The flow of data is unidirectional, originating from a permission authority and broadcasting outwards to all interested systems.

> Flow:
> Admin Command or API Call -> Permission Management Service -> **new PlayerPermissionChangeEvent.GroupAdded(...)** -> EventBus.post() -> Subscribed Listeners (Chat System, Player Session Manager, etc.)

