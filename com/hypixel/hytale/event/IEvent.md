---
description: Architectural reference for the IEvent interface, the core contract for all synchronous, key-based events.
---

# IEvent

**Package:** com.hypixel.hytale.event
**Type:** Contract Interface

## Definition
```java
// Signature
public interface IEvent<KeyType> extends IBaseEvent<KeyType> {
    // This interface defines no methods of its own.
    // It serves as a generic marker and contract, inheriting its
    // core behavior from IBaseEvent.
}
```

## Architecture & Concepts
The IEvent interface is a fundamental contract within Hytale's event-driven architecture. It serves as the primary marker interface for all synchronous, key-based events that flow through the central EventBus. It does not define any methods itself; its purpose is to establish a common, generic type for event objects and to inherit the core contract from its parent, IBaseEvent.

This interface is the cornerstone of decoupling in the engine. Systems do not need direct references to each other; instead, one system can publish an implementation of IEvent, and other interested systems can subscribe to that specific event type without any knowledge of the publisher.

The generic parameter, KeyType, is a critical architectural element. It dictates the type of key used by the EventBus to identify and dispatch the event. This allows for a highly flexible and type-safe subscription model, where listeners can register for events based on a specific key (e.g., a player UUID, an entity ID, or a specific event class).

## Lifecycle & Ownership
As an interface, IEvent itself has no lifecycle. The following applies to all concrete classes that **implement** this interface.

- **Creation:** Event objects are instantiated by a system at the moment a notable occurrence happens. For example, the networking layer might create a PlayerLoginEvent when a new connection is established. They are designed to be lightweight data carriers.

- **Scope:** The scope of an event object is extremely brief and transient. It exists only for the duration of its dispatch cycle within the EventBus. It is created, posted, processed by all relevant listeners, and then becomes eligible for garbage collection.

- **Destruction:** Event objects are intended to be fire-and-forget. Once the EventBus has finished dispatching an event to all subscribers, no system should retain a reference to it. The Java Garbage Collector is responsible for its final destruction.

**WARNING:** Holding a reference to an event object after its dispatch cycle is complete is a severe anti-pattern and a common source of memory leaks.

## Internal State & Concurrency
The contract defined by IEvent implies specific constraints on the state and thread safety of its implementations.

- **State:** Implementations of IEvent should be considered **immutable**. An event object is a snapshot of state at a specific moment in time. Modifying an event's data while it is being dispatched can lead to non-deterministic behavior and severe bugs. All data should be provided via the constructor and exposed through read-only methods.

- **Thread Safety:** Event objects must be thread-safe for reading. Since multiple listeners on different threads could theoretically access the same event instance (depending on the EventBus implementation), the internal data must be safely published. However, the standard engine pattern is to confine event processing to a single, main game thread to avoid concurrency issues altogether. Developers should assume event handlers are executed on a specific, designated thread.

## API Surface
The IEvent interface adds no new methods to its parent, IBaseEvent. The public contract is entirely inherited. The primary role of IEvent is to act as a generic type constraint for the synchronous event system.

## Integration Patterns

### Standard Usage
The primary pattern is to create a new, concrete class that implements IEvent. This class encapsulates the data relevant to the specific event.

```java
// 1. Define a new event class implementing IEvent.
//    The key type is often the class itself for simple broadcast events.
public class PlayerHealthChangedEvent implements IEvent<PlayerHealthChangedEvent> {
    private final Player player;
    private final int newHealth;

    public PlayerHealthChangedEvent(Player player, int newHealth) {
        this.player = player;
        this.newHealth = newHealth;
    }

    // Public getters for the immutable data...
}

// 2. A system creates and posts the event to the bus.
Player player = ...;
int newHealth = player.getHealth();
eventBus.post(new PlayerHealthChangedEvent(player, newHealth));
```

### Anti-Patterns (Do NOT do this)
- **Mutable State:** Do not add setters or other methods that modify the event's data after construction. This breaks the snapshot-in-time contract.
- **Reusing Instances:** Do not cache and reuse event objects. This is extremely dangerous and will cause unpredictable behavior, as listeners may receive stale data. Always create a `new` instance for each event.
- **Complex Logic:** Do not embed complex business logic within an event object. They are data carriers, not service objects.

## Data Pipeline
The data flow for any object implementing IEvent follows a clear, unidirectional path through the engine's core systems.

> Flow:
> Game State Change -> System Logic Creates `new MyEvent()` -> EventBus.post(event) -> **IEvent Dispatch Mechanism** -> Registered Listeners Execute -> Event Object is Discarded

