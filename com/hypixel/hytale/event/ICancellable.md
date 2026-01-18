---
description: Architectural reference for ICancellable
---

# ICancellable

**Package:** com.hypixel.hytale.event
**Type:** Contract Interface

## Definition
```java
// Signature
public interface ICancellable {
```

## Architecture & Concepts
The ICancellable interface defines the fundamental contract for events that can be vetoed or halted during their propagation through the event bus. It is not a concrete class but a behavioral mixin that event objects implement to signal their participation in a cooperative cancellation system.

When an event implements ICancellable, it grants any listener in the event pipeline the authority to prevent subsequent listeners from receiving and processing that same event instance. This is a critical control flow mechanism for managing game logic, preventing default actions, and establishing a priority-based processing chain. For example, a high-priority UI listener might consume a key press event and cancel it to prevent the underlying game world from also processing the same input.

This interface is the cornerstone of the engine's observer pattern, transforming it from a simple broadcast system into a sophisticated, stateful dispatch chain.

## Lifecycle & Ownership
As an interface, ICancellable itself has no lifecycle. It is a static contract. The lifecycle and ownership concerns apply to the concrete event classes that *implement* this interface.

- **Creation:** The implementing event object is typically created by a system that detects a state change (e.g., InputManager, NetworkSystem).
- **Scope:** The event object's lifetime is extremely short, usually confined to the duration of a single dispatch cycle within the EventBus. It is created, passed to all relevant listeners, and then becomes eligible for garbage collection.
- **Destruction:** The implementing event object is destroyed by the Java Garbage Collector once all references to it are dropped, which typically occurs immediately after the event dispatch concludes.

## Internal State & Concurrency
- **State:** The interface defines a contract for a single piece of mutable state: a boolean `cancelled` flag. The implementing class is responsible for storing and managing this flag. By design, this state is write-once, read-many; once an event is cancelled, it should never be un-cancelled.

- **Thread Safety:** **Not intrinsically thread-safe.** The contract involves mutable state via the `setCancelled` method. The Hytale event bus is designed to operate on a single thread (typically the main game thread). Therefore, concurrent modification of the cancelled flag is not an anticipated scenario.

    **WARNING:** Dispatching an event that implements ICancellable across multiple threads without external synchronization will lead to race conditions and non-deterministic behavior. Listeners on different threads could read a stale `isCancelled` value while another thread is in the process of calling `setCancelled`. All event processing must be marshaled to a single, designated thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isCancelled() | boolean | O(1) | Returns the current cancellation state of the event. |
| setCancelled(boolean) | void | O(1) | Sets the cancellation state. Passing true marks the event for cancellation. |

## Integration Patterns

### Standard Usage
The primary pattern is for an event listener to conditionally cancel an event to terminate the dispatch chain. Listeners should check `isCancelled` at the beginning of their execution to respect a prior listener's decision.

```java
// A listener method within a service
public void onPlayerInteract(PlayerInteractEvent event) {
    // Respect prior cancellations
    if (event.isCancelled()) {
        return;
    }

    // Conditional logic to consume the event
    if (player.isHoldingSpecialItem()) {
        // Perform special action...
        
        // Cancel the event to prevent default behavior (e.g., opening a door)
        event.setCancelled(true);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Un-cancelling Events:** Calling `setCancelled(false)` on an already-cancelled event is undefined behavior and violates the contract's intent. The cancellation state should be treated as a one-way flag.
- **Ignoring Cancellation State:** Failing to check `isCancelled()` at the start of a listener method. This breaks the chain of responsibility and can cause redundant or conflicting logic to execute.
- **Assuming Cancellability:** Attempting to cast any arbitrary event to ICancellable without a prior `instanceof` check. This will result in a ClassCastException.

## Data Pipeline
ICancellable acts as a gate within the event data pipeline. It does not transform data but rather terminates the flow for a specific event instance.

> Flow:
> Event Source -> EventBus.post(event) -> Listener 1 -> Listener 2 (calls `setCancelled(true)`) -> **[PIPELINE HALTED]** -> ~~Listener 3~~ -> ~~Default Action~~

