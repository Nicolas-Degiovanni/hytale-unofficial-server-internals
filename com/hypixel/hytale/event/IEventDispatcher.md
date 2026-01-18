---
description: Architectural reference for IEventDispatcher
---

# IEventDispatcher

**Package:** com.hypixel.hytale.event
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IEventDispatcher<EventType extends IBaseEvent, ReturnType> {
   default boolean hasListener() {
      return true;
   }

   ReturnType dispatch(@Nullable EventType var1);
}
```

## Architecture & Concepts
The IEventDispatcher interface is the central contract for the engine's event-driven communication system. It formalizes the "dispatch" side of a publisher-subscriber pattern, defining the mechanism by which a system can broadcast an event to an unknown number of listeners.

This interface is a cornerstone of decoupling within the Hytale architecture. It allows high-level systems (like the UI or Network managers) to announce state changes or occurrences without being directly coupled to the numerous low-level systems (like individual render components or game logic handlers) that need to react to them.

Any class that implements IEventDispatcher acts as a specialized "event bus" or "channel" for a specific category of events, defined by the generic parameter EventType. The dispatcher is responsible for managing a collection of listeners and invoking them when an event is dispatched.

## Lifecycle & Ownership
As an interface, IEventDispatcher itself has no lifecycle. The following pertains to its concrete implementations.

- **Creation:** Implementations are typically instantiated and owned by major engine services or managers. For example, a global ClientEventBus would be created at application startup, while a more scoped WorldEventDispatcher might be created when a world is loaded.
- **Scope:** The lifetime of a dispatcher implementation is tied directly to its owner. A global dispatcher is a session-scoped singleton, whereas a dispatcher for a specific UI screen is transient and exists only as long as that screen is active.
- **Destruction:** Implementations are eligible for garbage collection when their owning system is destroyed and all references are released. There is no explicit destruction method in the contract.

## Internal State & Concurrency
- **State:** The interface is stateless. However, any non-trivial implementation is inherently stateful, as it must maintain an internal collection of registered listeners. This collection is mutable, with listeners being added and removed at runtime.
- **Thread Safety:** **CRITICAL:** The IEventDispatcher contract makes no guarantees about thread safety. It is the responsibility of the concrete implementation to ensure safe concurrent access. Implementations intended for multi-threaded dispatch (e.g., from network and main threads simultaneously) **must** use thread-safe collections for listeners and appropriate synchronization on the dispatch method to prevent concurrent modification exceptions and ensure memory visibility. Consumers should never assume an implementation is thread-safe without explicit documentation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasListener() | boolean | O(1) | A default method providing a simple check. Implementations should override this for an accurate report on whether any listeners are currently subscribed. |
| dispatch(EventType event) | ReturnType | O(N) | Broadcasts the given event to all N registered listeners. The return type is generic and implementation-dependent, often used to aggregate results or handle cancellation. |

## Integration Patterns

### Standard Usage
The primary pattern is to retrieve a specific dispatcher from a service locator or context and use it to broadcast a new event instance.

```java
// Obtain the appropriate dispatcher for player-related events
IEventDispatcher<PlayerJoinEvent, Void> dispatcher = ServiceRegistry.get(PlayerEventBus.class);

// Create and populate the event object
PlayerJoinEvent joinEvent = new PlayerJoinEvent(playerEntity);

// Dispatch the event to all listeners
dispatcher.dispatch(joinEvent);
```

### Anti-Patterns (Do NOT do this)
- **Assuming Implementation:** Do not cast an IEventDispatcher to a concrete class to access internal state, such as the listener list. This violates encapsulation and creates brittle, implementation-dependent code.
- **Dispatching from Constructors:** Dispatching an event from within the constructor of an object can lead to race conditions where listeners receive the event before the object is fully initialized.
- **Ignoring Threading Context:** Dispatching a computationally expensive event from a performance-sensitive thread (e.g., the render thread) can cause significant frame drops. Events should be dispatched on the appropriate thread, typically the main game logic thread.

## Data Pipeline
The IEventDispatcher is a key component in the flow of information between decoupled systems. It acts as the central hub for a given event type.

> Flow:
> Event Source (e.g., Input System) -> Creates `KeyEvent` object -> Calls `InputDispatcher.dispatch(event)` -> **IEventDispatcher Implementation** -> Invokes `onKeyEvent` on all registered listeners -> Listeners (e.g., UI, Player Controller) consume the event.

