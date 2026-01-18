---
description: Architectural reference for DiscoverInstanceEvent
---

# DiscoverInstanceEvent

**Package:** com.hypixel.hytale.builtin.instances.event
**Type:** Event

## Definition
```java
// Signature
public abstract class DiscoverInstanceEvent extends EcsEvent {
    // Nested concrete implementation
    public static class Display extends DiscoverInstanceEvent implements ICancellableEcsEvent {
        // ...
    }
}
```

## Architecture & Concepts
DiscoverInstanceEvent is a message-oriented data structure used within the Entity Component System (ECS) to signal the initiation of a game instance discovery process. It acts as a command, instructing various engine systems to locate, process, and potentially display information about a specific game world, identified by its UUID.

This class follows the Command pattern, where the event object itself encapsulates all the information needed to perform the action. It is designed to be dispatched onto an event bus, decoupling the event producer (e.g., a UI server browser) from the event consumers (e.g., networking and UI rendering systems).

The primary implementation is the nested static class, **Display**. This variant is cancellable, allowing intermediary systems to intercept and halt the discovery process. For example, a system might cancel the event if the user's network is offline or if the instance is on a blocklist.

## Lifecycle & Ownership
- **Creation:** An instance of DiscoverInstanceEvent.Display is created and dispatched by a system that needs to query for a game instance. Common producers include the server browser UI controller upon a user action or an automated system handling friend invites.
- **Scope:** The event's lifecycle is extremely brief and ephemeral. It exists only for the duration of its propagation through the ECS event bus during a single engine tick.
- **Destruction:** Once all registered listeners have processed the event, it falls out of scope and becomes eligible for garbage collection. There is no manual memory management required.

## Internal State & Concurrency
- **State:** The base DiscoverInstanceEvent is immutable after construction, as its core data fields are final. However, the concrete **Display** subclass is **mutable**. Its state, specifically the *cancelled* and *display* flags, is intentionally designed to be modified by event listeners during its processing lifecycle.
- **Thread Safety:** This class is **not thread-safe**. EcsEvent objects are designed to be created, dispatched, and handled within a single, well-defined thread, typically the main game loop thread. Modifying an event's state from a worker thread while it is being processed on the main thread will lead to race conditions and undefined behavior.

**WARNING:** Never dispatch or modify a DiscoverInstanceEvent from a thread other than the one that owns the target ECS event bus. All event handling logic must be confined to the main engine update loop.

## API Surface
The public contract is focused on data retrieval and state modification for the cancellable flow.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInstanceWorldUuid() | UUID | O(1) | Returns the unique identifier for the target game world instance. |
| getDiscoveryConfig() | InstanceDiscoveryConfig | O(1) | Returns the configuration object detailing how discovery should be performed. |
| isCancelled() | boolean | O(1) | (Display only) Checks if any system has cancelled this event. |
| setCancelled(boolean) | void | O(1) | (Display only) Marks the event as cancelled, signaling subsequent systems to ignore it. |
| shouldDisplay() | boolean | O(1) | (Display only) Checks if the instance should be rendered in a UI list. |
| setDisplay(boolean) | void | O(1) | (Display only) Allows a system to override the initial display intention from the config. |

## Integration Patterns

### Standard Usage
The correct pattern is to create a Display event, dispatch it to the appropriate event bus, and let registered systems handle the logic. Systems listen for the event, inspect its properties, and potentially modify its mutable state.

```java
// A UI system wants to find and show a server from a friend's invite.
UUID serverId = ...;
InstanceDiscoveryConfig config = new InstanceDiscoveryConfig(true); // true = display by default

// Create and dispatch the event
DiscoverInstanceEvent.Display event = new DiscoverInstanceEvent.Display(serverId, config);
ecsEventBus.dispatch(event);

// The event is now processed by other systems. If event.isCancelled() is true
// after dispatch, the discovery was halted.
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not hold a reference to an event object after it has been dispatched. Its state is only valid for the single tick in which it is processed.
- **Asynchronous Modification:** Do not dispatch an event and then attempt to modify it from a separate thread. The event bus is not designed for cross-thread communication.
- **Ignoring Cancellation:** Systems that perform high-cost operations (like network requests) based on this event **must** check the isCancelled flag before proceeding. Failure to do so violates the contract and can lead to wasted resources.

## Data Pipeline
The DiscoverInstanceEvent serves as the initial data packet in a multi-stage discovery pipeline.

> Flow:
> User Action (e.g., Click "Join") -> UI System creates **DiscoverInstanceEvent.Display** -> ECS Event Bus -> Network System receives event, checks `isCancelled`, initiates connection -> UI List System receives event, checks `isCancelled` and `shouldDisplay`, adds a "pending" entry to the server list.

