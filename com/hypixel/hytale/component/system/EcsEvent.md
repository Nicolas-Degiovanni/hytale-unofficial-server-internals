---
description: Architectural reference for EcsEvent
---

# EcsEvent

**Package:** com.hypixel.hytale.component.system
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class EcsEvent {
```

## Architecture & Concepts
The EcsEvent class is an architectural primitive and the foundational base type for all events within the Entity-Component-System (ECS) framework. It establishes a contract for message-passing, enabling highly decoupled communication between different game systems.

This class is not intended for direct use. Instead, developers must extend it to create concrete event types, such as PlayerJoinEvent or EntityDamagedEvent. These concrete event objects act as immutable data carriers, encapsulating the state associated with a specific occurrence in the game world.

The primary role of EcsEvent is to serve as a marker and a common supertype, allowing the central EcsEventBus to manage, route, and dispatch any event type to its registered listeners without prior knowledge of the event's specific implementation. This pattern is central to achieving a reactive and data-driven architecture, preventing systems from holding direct references to one another.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of EcsEvent are instantiated on-demand by various systems when a notable event occurs. For example, the physics system might create a CollisionEvent when two entities intersect. The abstract EcsEvent class itself is never instantiated.
- **Scope:** Event objects are extremely short-lived and ephemeral. Their lifecycle is typically confined to a single game tick. They are created, dispatched, handled, and then immediately become candidates for garbage collection.
- **Destruction:** There is no manual destruction. Once the EcsEventBus has finished broadcasting an event to all its listeners, no strong references to the event object should remain, and it will be reclaimed by the Java garbage collector.

## Internal State & Concurrency
- **State:** The EcsEvent base class is stateless. Subclasses are designed to hold immutable, final fields representing the event's payload.
    - **WARNING:** It is a critical design principle that event objects be treated as immutable once created. Modifying an event's state after dispatch can lead to unpredictable behavior and severe race conditions, as multiple listeners may be processing it concurrently or sequentially.
- **Thread Safety:** EcsEvent objects are not thread-safe. The ECS event bus is designed to operate within a single, well-defined thread, typically the main game update thread. Passing event objects across threads is a dangerous anti-pattern and must be avoided unless managed by a thread-safe queueing mechanism that marshals the event back to the main thread for processing.

## API Surface
As an abstract class with no defined methods, EcsEvent has no concrete API surface. Its contract is purely structural, enforced through inheritance. The "API" is the pattern of extending the class to create new event types.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (N/A) | (N/A) | (N/A) | This class defines no public methods or fields. |

## Integration Patterns

### Standard Usage
The correct pattern is to define a new, concrete event class that extends EcsEvent and then dispatch it through the appropriate system, typically an event bus or registry.

```java
// 1. Define a concrete event
public class PlayerHealthChangedEvent extends EcsEvent {
    public final int entityId;
    public final float newHealth;

    public PlayerHealthChangedEvent(int entityId, float newHealth) {
        this.entityId = entityId;
        this.newHealth = newHealth;
    }
}

// 2. Dispatch the event from a game system
EcsEventBus eventBus = context.getService(EcsEventBus.class);
eventBus.dispatch(new PlayerHealthChangedEvent(playerId, 85.0f));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to use `new EcsEvent()`. This will result in a compile-time error as the class is abstract.
- **Mutable State:** Do not define events with public, non-final fields. An event is a record of something that *has happened*; its data must not change after creation.
- **Overly Complex Events:** Events should be simple data carriers. Avoid embedding complex logic, service locators, or system references within an event object. This violates the principle of decoupling.

## Data Pipeline
The EcsEvent class is a data payload that moves through the core game loop's event processing pipeline.

> Flow:
> Game System Logic -> `new ConcreteEvent()` -> EcsEventBus Dispatcher -> **EcsEvent (Payload)** -> Registered System Listeners -> Game State Mutation

