---
description: Architectural reference for ICancellableEcsEvent
---

# ICancellableEcsEvent

**Package:** com.hypixel.hytale.component.system
**Type:** Contract Interface

## Definition
```java
// Signature
public interface ICancellableEcsEvent {
```

## Architecture & Concepts
The ICancellableEcsEvent interface defines a fundamental contract for events within the Entity Component System (ECS) that can be intercepted and halted during their propagation. It is a critical component of the engine's event-driven architecture, enabling systems to control the flow of information and prevent subsequent systems from processing an event that has already been handled or should be ignored.

This interface embodies the "Chain of Responsibility" pattern within the ECS event bus. When an event implementing this interface is dispatched, each listening system is given an opportunity to process it. A listener can choose to "cancel" the event, which signals to the dispatcher that the event chain should be terminated immediately, and no further listeners should be notified. This is essential for scenarios like input handling, where the first system that consumes an input (e.g., a UI element) should prevent the game world from also processing it.

## Lifecycle & Ownership
As an interface, ICancellableEcsEvent does not have a lifecycle of its own. Instead, it defines the lifecycle contract for the concrete event objects that implement it.

- **Creation:** Concrete event objects are instantiated by event producers, typically other systems or core engine services, in response to a specific occurrence (e.g., player interaction, network packet arrival).
- **Scope:** The lifetime of an event object is extremely short and transient. It exists only for the duration of a single dispatch cycle within a single engine tick.
- **Destruction:** Once the event has been fully propagated through all relevant listeners (or has been cancelled), it falls out of scope and is eligible for garbage collection. There is no persistent state associated with these event objects.

## Internal State & Concurrency
- **State:** The interface defines a single piece of mutable state: the boolean *cancelled* flag. Concrete implementations are, by definition, mutable.
- **Thread Safety:** This interface is **not inherently thread-safe**. The responsibility for safe publication and modification lies with the event bus implementation. In the standard engine architecture, ECS events are processed sequentially on a single thread (e.g., the main game loop thread). Accessing or modifying an event from multiple threads simultaneously will lead to race conditions and unpredictable behavior.

**WARNING:** Never dispatch an event instance across multiple threads without external synchronization. The event bus must guarantee that all listeners for a single event instance are invoked serially.

## API Surface
The public contract is minimal, focusing exclusively on managing the cancellation state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isCancelled() | boolean | O(1) | Returns the current cancellation state of the event. The event dispatcher must call this after each listener invocation. |
| setCancelled(boolean) | void | O(1) | Sets the cancellation state. Typically called with *true* to halt event propagation. |

## Integration Patterns

### Standard Usage
The primary pattern involves a listener system conditionally cancelling an event to terminate the dispatch chain. The event dispatcher is responsible for respecting the flag.

```java
// In an ECS System (Listener)
public void onPlayerInteract(PlayerInteractEvent event) {
    // If a UI window is open and handles the click
    if (uiManager.isHandledByUI(event.getClickLocation())) {
        // Consume the event and stop it from reaching game world systems
        event.setCancelled(true);
    }
}

// In the EventBus (Dispatcher) - Conceptual
for (Listener listener : getListenersFor(event.class)) {
    listener.process(event);
    if (event instanceof ICancellableEcsEvent && ((ICancellableEcsEvent) event).isCancelled()) {
        break; // Halt propagation
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Flag:** An event dispatcher that does not check isCancelled after invoking a listener completely defeats the purpose of this interface, potentially causing severe bugs like duplicate action processing.
- **Un-cancelling Events:** A listener should never call setCancelled(false) on an event that may have already been cancelled by a previous listener in the chain. This violates the chain of responsibility and leads to unpredictable, order-dependent logic.

## Data Pipeline
This interface primarily defines a control flow mechanism rather than a data pipeline. It acts as a gate in the flow of event objects.

> Flow:
> Event Source (e.g., Input System) -> Creates Concrete Event -> Event Bus Dispatch -> Listener A (Processes) -> Listener B (Processes and calls **setCancelled(true)**) -> Event Bus (Detects cancellation) -> Propagation Halts; Listener C is never called.

