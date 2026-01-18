---
description: Architectural reference for GroupPermissionChangeEvent
---

# GroupPermissionChangeEvent

**Package:** com.hypixel.hytale.server.core.event.events.permissions
**Type:** Event Object

## Definition
```java
// Signature
public abstract class GroupPermissionChangeEvent implements IEvent<Void> {

    // Nested Concrete Implementations
    public static class Added extends GroupPermissionChangeEvent { /* ... */ }
    public static class Removed extends GroupPermissionChangeEvent { /* ... */ }
}
```

## Architecture & Concepts
The GroupPermissionChangeEvent is a data-carrying object, not a service. It functions as a message within the server's core event bus system. Its primary architectural purpose is to decouple the **Permission Management System** from other server subsystems that must react to changes in permission group configurations.

This class follows a polymorphic event pattern. The abstract base class, GroupPermissionChangeEvent, defines the common contract—identifying the affected group. The two concrete nested classes, **Added** and **Removed**, provide specific data about the nature of the change. This design allows subsystems to listen for the general event type or subscribe to only the specific changes they care about, promoting a clean, loosely-coupled architecture.

When a server administrator modifies a permission group, the PermissionService constructs and dispatches an instance of this event. Subsystems like the command handler, chat system, or gameplay rule engines can listen for these events to dynamically update their behavior without being directly coupled to the PermissionService.

## Lifecycle & Ownership
-   **Creation:** An instance of a concrete subclass (Added or Removed) is created exclusively by the server's internal PermissionService. This occurs immediately after a successful modification to a permission group's rule set.
-   **Scope:** The object's lifetime is exceptionally brief and is tied to the event dispatch cycle. It exists only for the time it takes the event bus to broadcast it to all registered listeners.
-   **Destruction:** Once all listeners have processed the event, it falls out of scope and becomes eligible for garbage collection. There are no manual cleanup or destruction methods.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields (`groupName`, `addedPermissions`, `removedPermissions`) are declared as `final` and are assigned only during construction. The `Set` collections exposed by the getters are defensively wrapped in `Collections.unmodifiableSet`, preventing any post-construction modification.

-   **Thread Safety:** **Inherently thread-safe**. Its strict immutability guarantees that an instance can be safely published and consumed across multiple threads without any need for external synchronization or locks. This is a critical feature for events that may be handled by asynchronous or multi-threaded listeners in a high-performance server environment.

## API Surface
This API is designed for data retrieval only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGroupName() | String | O(1) | Returns the name of the group that was modified. |
| getAddedPermissions() | Set<String> | O(1) | **(On Added)** Returns an unmodifiable set of permissions that were added. |
| getRemovedPermissions() | Set<String> | O(1) | **(On Removed)** Returns an unmodifiable set of permissions that were removed. |

## Integration Patterns

### Standard Usage
The correct way to interact with this event is to create a listener method within a service or manager and register it with the server's event bus. The listener should check the concrete type of the event to handle additions and removals correctly.

```java
// Example of a listener in a hypothetical service
@Subscribe
public void onGroupPermissionsChanged(GroupPermissionChangeEvent event) {
    if (event instanceof GroupPermissionChangeEvent.Added) {
        GroupPermissionChangeEvent.Added addedEvent = (GroupPermissionChangeEvent.Added) event;
        log.info("Permissions added to group " + addedEvent.getGroupName() + ": " + addedEvent.getAddedPermissions());
        // Logic to refresh caches or update player states
    } else if (event instanceof GroupPermissionChangeEvent.Removed) {
        GroupPermissionChangeEvent.Removed removedEvent = (GroupPermissionChangeEvent.Removed) event;
        log.info("Permissions removed from group " + removedEvent.getGroupName() + ": " + removedEvent.getRemovedPermissions());
        // Logic to re-evaluate access for players in this group
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of this event using `new`. These events are a reflection of the true state of the PermissionService. Manually firing a synthetic event will cause other systems to become desynchronized from the authoritative permission state, leading to unpredictable behavior.
-   **Attempting to Modify State:** The collections returned by `getAddedPermissions` and `getRemovedPermissions` are immutable. Attempting to call methods like `add()` or `remove()` on them will result in an `UnsupportedOperationException` at runtime.
-   **Ignoring Event Subtype:** Writing a listener that only consumes the base GroupPermissionChangeEvent without checking its concrete type is a code smell. This will inevitably lead to incomplete or incorrect logic, as the handler will not know whether permissions were added or removed.

## Data Pipeline
This event is a message that flows from the point of authority (the PermissionService) to various observers.

> Flow:
> Administrative Action (e.g., Console Command) -> PermissionService Logic -> **new GroupPermissionChangeEvent.Added(...)** -> Server EventBus -> Registered Listeners (e.g., CommandHandler, PlayerSessionManager)
---

