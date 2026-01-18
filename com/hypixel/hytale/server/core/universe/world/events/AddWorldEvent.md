---
description: Architectural reference for AddWorldEvent
---

# AddWorldEvent

**Package:** com.hypixel.hytale.server.core.universe.world.events
**Type:** Transient

## Definition
```java
// Signature
public class AddWorldEvent extends WorldEvent implements ICancellable {
```

## Architecture & Concepts
The AddWorldEvent is a transient, state-bearing object that functions as a control signal within the server's event-driven architecture. It is dispatched immediately before a new World instance is formally registered with the server's central Universe.

Its primary architectural purpose is to provide a **veto point** for other systems. By implementing the ICancellable interface, this event allows subsystems, such as plugins or custom game rule engines, to intercept and prevent a world from being added without modifying the core world management code. This creates a powerful, decoupled mechanism for enforcing server policies, validating world states, or managing resource limits.

This event is a fundamental component of the World lifecycle management system. It represents the transition of a World from a "pending" state to an "active" state.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the core world management service (e.g., a UniverseManager) when it receives a request to add a pre-configured World object.
- **Scope:** Extremely short-lived. An AddWorldEvent instance exists only for the duration of its dispatch on the server's main event bus. It is a fire-and-forget signal.
- **Destruction:** The object becomes eligible for garbage collection immediately after the event dispatch cycle completes and the originating caller has checked its final cancellation state. No system should retain a long-term reference to an event object.

## Internal State & Concurrency
- **State:** Mutable. The event's core state is the boolean `cancelled` flag. This field is intentionally mutable, as its purpose is to be modified by one or more event listeners during the dispatch process. It also carries an immutable reference to the World instance in question, inherited from its parent.

- **Thread Safety:** **This class is not thread-safe.** Event objects are designed to be created, dispatched, and processed within a single, synchronous execution block, typically the main server thread. Modifying the cancellation state from an asynchronous task or a different thread will result in a race condition and lead to undefined behavior. All listener logic interacting with this event must be executed on the same thread as the event bus.

## API Surface
The public contract is defined by the ICancellable interface and the data accessor inherited from WorldEvent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWorld() | World | O(1) | Retrieves the non-null World instance that is pending addition. Inherited from WorldEvent. |
| isCancelled() | boolean | O(1) | Returns true if any listener has vetoed the addition of the world. |
| setCancelled(boolean) | void | O(1) | Sets the cancellation state. This is the primary interaction point for listeners wishing to veto the operation. |

## Integration Patterns

### Standard Usage
The canonical use case is within an event listener method. The listener inspects the World associated with the event and, based on some logic, calls `setCancelled(true)` to prevent the action.

```java
// Example of a listener within a server-side plugin or system
@Subscribe
public void onPreWorldAdd(AddWorldEvent event) {
    World candidateWorld = event.getWorld();

    // Enforce a server-wide policy, e.g., limit the total number of active worlds.
    if (server.getActiveWorldCount() >= MAX_WORLDS) {
        log.warn("Blocking addition of world '{}' because the server limit has been reached.", candidateWorld.getName());
        event.setCancelled(true);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of AddWorldEvent using `new`. These events are meaningful only when dispatched by the core engine. Manually creating and posting one will have no effect on the server's actual world list and can desynchronize state.
- **Asynchronous Cancellation:** Do not dispatch the event to a separate thread pool and attempt to modify its state later. The calling code in the UniverseManager proceeds immediately after the synchronous event dispatch and will read an incorrect cancellation state.
- **State Reversal:** While possible, it is a dangerous practice for one listener to reverse a cancellation set by another (calling `setCancelled(false)` after another listener called `setCancelled(true)`). This can break assumptions made by other plugins and lead to unpredictable load orders. The first system to cancel should generally have final authority.

## Data Pipeline
The AddWorldEvent acts as a control gate in the data flow of world registration.

> Flow:
> World Creation Request -> UniverseManager -> **AddWorldEvent** (Instantiation) -> EventBus (Dispatch to Listeners) -> Listeners (Conditional State Mutation) -> UniverseManager (Checks `isCancelled`) -> World Registered in Universe **OR** World Addition Aborted

