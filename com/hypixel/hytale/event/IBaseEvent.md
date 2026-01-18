---
description: Architectural reference for IBaseEvent
---

# IBaseEvent

**Package:** com.hypixel.hytale.event
**Type:** Contract

## Definition
```java
// Signature
public interface IBaseEvent<KeyType> {
```

## Architecture & Concepts
The IBaseEvent interface is the foundational contract for all event objects within the Hytale engine's event bus system. It serves as a generic marker interface, establishing a common type for any data structure intended to be broadcast through the EventManager. Its primary architectural role is to enforce a decoupled communication pattern between disparate engine subsystems.

By having systems publish and subscribe to concrete implementations of IBaseEvent, direct dependencies are eliminated. For example, the Physics system does not need a direct reference to the Audio system to notify it of a collision; it simply posts a CollisionEvent.

The generic parameter, KeyType, is a critical design element. It provides a compile-time mechanism for associating an event with a specific listener key, enabling the EventManager to implement a highly efficient and type-safe dispatch mechanism. This prevents runtime errors and simplifies the process of registering listeners for specific event types.

## Lifecycle & Ownership
As an interface, IBaseEvent itself has no runtime lifecycle. It is a compile-time construct that defines a contract, not an object that is instantiated or destroyed.

The lifecycle and ownership concerns apply to the **concrete classes** that implement this interface.

- **Creation:** Concrete event objects are typically created as transient, short-lived data carriers. They are instantiated by a system that has detected a state change or a significant occurrence (e.g., NetworkSystem creates a PacketReceivedEvent).
- **Scope:** The scope of a concrete event object is typically limited to the duration of its dispatch through the EventManager. Once all registered listeners have processed it, it is eligible for garbage collection.
- **Destruction:** Event objects are managed by the Java Virtual Machine's garbage collector. There is no manual destruction or cleanup required.

**WARNING:** Event objects should not hold references to long-lived services or resources, as this can inadvertently extend their lifecycle and lead to memory leaks if the event is unintentionally retained.

## Internal State & Concurrency
The IBaseEvent interface is stateless.

- **State:** By definition, an interface has no state. Concrete implementations are expected to be simple Plain Old Java Objects (POJOs) or records, acting as immutable data containers. Modifying an event object after its creation is a severe anti-pattern.
- **Thread Safety:** The interface itself is inherently thread-safe. The thread safety of a concrete event implementation is the responsibility of its author. Best practice dictates that event objects should be **deeply immutable**. If an event is posted from one thread and consumed by listeners on another, its immutability is critical to prevent race conditions and ensure predictable behavior.

## API Surface
The IBaseEvent interface defines no methods. Its public contract is entirely expressed through its generic type parameter, which dictates the keying strategy for listener registration within the EventManager.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| <KeyType> | Generic | N/A | A type parameter that defines the key used to subscribe to this event. |

## Integration Patterns

### Standard Usage
The primary pattern is to create a specific, immutable event class that implements IBaseEvent. This class serves as a data transfer object for a particular occurrence.

```java
// 1. Define a concrete, immutable event class
public final class PlayerConnectEvent implements IBaseEvent<PlayerConnectEvent.Key> {
    public static final Key KEY = new Key();
    public static final class Key {}

    private final UUID playerUUID;
    private final String playerName;

    public PlayerConnectEvent(UUID uuid, String name) {
        this.playerUUID = uuid;
        this.playerName = name;
    }

    // Public, final getters for data access
    public UUID getPlayerUUID() { return playerUUID; }
    public String getPlayerName() { return playerName; }
}

// 2. A system creates and posts the event
PlayerConnectEvent event = new PlayerConnectEvent(uuid, name);
eventManager.post(PlayerConnectEvent.KEY, event);
```

### Anti-Patterns (Do NOT do this)
- **Mutable Events:** Do not design event classes with public setters or mutable fields. An event represents a fact that has already occurred; it must not be changed after creation.
- **Generic KeyType:** Avoid using overly broad types like `Object.class` or `String` for the KeyType. This undermines the type safety of the event system and can lead to listener collisions. Always define a unique, static key class within the event itself.
- **Logic in Events:** Do not place business logic, calculations, or service calls within an event object's constructor or methods. Events are data carriers, not active components.

## Data Pipeline
IBaseEvent is the data payload that moves through the engine's central communication bus.

> Flow:
> Engine Subsystem (e.g., Network, Physics) -> Instantiates Concrete Event (implements **IBaseEvent**) -> EventManager.post() -> Listeners (other subsystems) -> State Update

