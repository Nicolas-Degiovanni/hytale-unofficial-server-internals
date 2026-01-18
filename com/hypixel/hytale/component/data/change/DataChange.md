---
description: Architectural reference for the DataChange marker interface.
---

# DataChange

**Package:** com.hypixel.hytale.component.data.change
**Type:** Contract / Marker Interface

## Definition
```java
// Signature
public interface DataChange {
}
```

## Architecture & Concepts
The DataChange interface is a **marker interface** that forms the fundamental contract for the engine's state synchronization and event system. It has no methods and serves solely to tag classes that represent a discrete modification to game data.

This design pattern is critical for creating a decoupled and extensible architecture. Systems that produce state changes (e.g., a physics system updating an entity's position) can create objects implementing DataChange without needing to know which other systems will consume them. Conversely, consumer systems (e.g., the network synchronizer, the renderer, or the audio engine) can subscribe to any object that fulfills the DataChange contract, allowing them to react to game state modifications generically.

It acts as the common language for the engine's reactive subsystems, ensuring that all state modifications can be queued, serialized, and processed through a unified pipeline.

## Lifecycle & Ownership
As an interface, DataChange itself has no lifecycle. The following lifecycle applies to any **object that implements** this interface.

- **Creation:** Instances are typically created by a component or system when its internal state is modified. For example, the HealthComponent would instantiate a HealthDataChange object when an entity takes damage.
- **Scope:** The scope is almost always transient. A DataChange object exists only long enough to be processed by the relevant systems, after which it is eligible for garbage collection. They are short-lived data transfer objects.
- **Destruction:** Managed by the Java Garbage Collector. Once a DataChange object has been processed by all subscribed listeners (e.g., after being sent over the network or applied to the game state), all references to it are dropped.

## Internal State & Concurrency
The DataChange interface defines no state.

- **State:** Any class implementing this interface is expected to be a simple data container, often immutable or effectively immutable after creation. Its fields should represent the "before" and "after" states of a change, or simply the new state.
- **Thread Safety:** Implementations of DataChange are **not required** to be thread-safe. They are intended to be created on one thread (typically the main game loop) and passed to other systems through thread-safe queues or event buses. Modifying a DataChange object after it has been published is a severe anti-pattern and will lead to race conditions.

## API Surface
This is a marker interface and has no methods. The API contract is fulfilled simply by implementing it.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (None) | - | - | This interface defines no public symbols. |

## Integration Patterns

### Standard Usage
The primary pattern is to implement this interface on a Plain Old Java Object (POJO) or a record that encapsulates a specific state change. This object is then published to a central change management system or event bus.

```java
// 1. Define a class representing a specific change
public final class EntityPositionChange implements DataChange {
    public final int entityId;
    public final Vector3 newPosition;

    public EntityPositionChange(int entityId, Vector3 newPosition) {
        this.entityId = entityId;
        this.newPosition = newPosition;
    }
}

// 2. A system creates and publishes the change
DataChange change = new EntityPositionChange(123, new Vector3(10, 20, 30));
changeTracker.publish(change);
```

### Anti-Patterns (Do NOT do this)
- **Adding Logic:** Do not add business logic or complex behavior to classes that implement DataChange. They should be simple, passive data carriers.
- **Mutable State:** Avoid making the fields of a DataChange implementation mutable. Once created and published, the object should be treated as immutable to prevent race conditions between consumer systems.
- **Implementing on Services:** Do not implement this interface on long-lived services or components. It is strictly for transient, event-like data objects.

## Data Pipeline
Objects implementing DataChange are the payload that flows through the engine's state management pipeline. They are the atomic unit of change.

> Flow:
> Game System (e.g., Physics) -> Creates **EntityPositionChange** -> ChangeTrackerService -> Serializer -> Network Layer -> Client Decoder -> Client Game State<ctrl63>

