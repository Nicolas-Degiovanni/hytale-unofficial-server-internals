---
description: Architectural reference for CancellableEcsEvent
---

# CancellableEcsEvent

**Package:** com.hypixel.hytale.component.system
**Type:** Component Model

## Definition
```java
// Signature
public abstract class CancellableEcsEvent extends EcsEvent implements ICancellableEcsEvent {
```

## Architecture & Concepts
CancellableEcsEvent is an abstract base class that forms a foundational component of the engine's event-driven architecture within the Entity-Component-System (ECS) framework. It extends the base EcsEvent with a critical feature: a standardized mechanism for event cancellation.

This class embodies the *Chain of Responsibility* and *Observer* patterns. When an event is dispatched, it travels through a series of listening systems. Any system in this chain can inspect the event and decide to "cancel" it, preventing it from propagating to subsequent systems or from triggering a final, state-changing action.

This design decouples the event's originator from the complex game logic that might invalidate it. For example, a system that fires a PlayerDamageEvent does not need to know about invincibility buffs, protective shields, or special game zones. Instead, dedicated systems for those features can listen for the event and cancel it, ensuring a clean separation of concerns.

The implementation of the cancellation logic is marked as **final**, enforcing a consistent contract across all cancellable events in the engine.

### Lifecycle & Ownership
- **Creation:** Instances of concrete subclasses are created on-demand by game logic or systems when a notable occurrence happens. For example: `new PlayerTakesDamageEvent(entityId, damageAmount)`. They are immediately dispatched to an event bus.
- **Scope:** Transient and extremely short-lived. An event object exists only for the duration of its processing cycle within a single game tick. It is a temporary data carrier, not a long-term state container.
- **Destruction:** The object becomes eligible for garbage collection as soon as the event bus finishes its dispatch cycle for that tick. No references are maintained, preventing memory leaks.

## Internal State & Concurrency
- **State:** Mutable. The class contains a single boolean field, `cancelled`, which defaults to false. Its state is expected to change exactly once during its lifecycle, from false to true.

- **Thread Safety:** **Not thread-safe.** This class is designed for a single-threaded execution model, synchronous with the main game loop. All event creation, dispatch, and modification must occur on the same thread.

    **WARNING:** Accessing or modifying an event instance from multiple threads will introduce severe race conditions, leading to unpredictable and non-deterministic game behavior. The ECS framework guarantees serial, ordered processing of events to prevent this.

## API Surface
The public contract is minimal and final, focused exclusively on managing the cancellation state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isCancelled() | boolean | O(1) | Returns true if any listening system has previously called setCancelled(true). |
| setCancelled(boolean) | void | O(1) | Sets the cancellation state. Typically called with true to halt event processing. |

## Integration Patterns

### Standard Usage
The primary pattern is to extend this class to create a specific, concrete event. Systems then listen for this event and conditionally cancel it. The event dispatcher is responsible for checking the cancelled status after each listener invocation.

```java
// 1. Define a concrete, cancellable event
public class PlayerChatEvent extends CancellableEcsEvent {
    public final String message;
    public PlayerChatEvent(String message) { this.message = message; }
}

// 2. A system listens and applies logic
public class ChatFilterSystem implements IEventListener<PlayerChatEvent> {
    @Override
    public void onEvent(PlayerChatEvent event) {
        if (isProfane(event.message)) {
            // Halt the event. The message will not be broadcast.
            event.setCancelled(true);
        }
    }
}

// 3. The event bus respects the flag
// (Conceptual dispatcher logic)
void dispatch(CancellableEcsEvent event) {
    for (IEventListener listener : getListenersFor(event.class)) {
        listener.onEvent(event);
        if (event.isCancelled()) {
            // Stop propagation immediately
            return;
        }
    }
    // ... proceed with final action if not cancelled
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Flag:** A system that dispatches a cancellable event and immediately proceeds with its logic without waiting for the event bus to complete processing is a critical anti-pattern. This completely bypasses the cancellation mechanism.
- **Asynchronous Modification:** Do not dispatch an event and then modify it from a separate thread or a callback that executes later. The event's state must only be modified synchronously within the event listener's call stack.
- **Resetting Cancellation:** Calling `setCancelled(false)` after an event has already been cancelled by another system is a dangerous practice. It violates the chain of responsibility and can resurrect an event that game logic has already determined should be stopped.

## Data Pipeline
The flow of data and control for a cancellable event is strictly linear and synchronous within a single game tick.

> Flow:
> Game Logic Instantiation (`new ConcreteEvent()`) -> Event Bus Dispatch -> System A (Listener) -> System B (Listener, calls `setCancelled(true)`) -> **Event Bus** (Detects cancelled state) -> Propagation Halts -> Final Action Is Skipped

