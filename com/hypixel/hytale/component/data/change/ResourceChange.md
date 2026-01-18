---
description: Architectural reference for ResourceChange
---

# ResourceChange

**Package:** com.hypixel.hytale.component.data.change
**Type:** Transient Value Object

## Definition
```java
// Signature
public class ResourceChange<ECS_TYPE, T extends Resource<ECS_TYPE>> implements DataChange {
```

## Architecture & Concepts
The ResourceChange class is a fundamental, immutable data transfer object within the Entity Component System's data layer. It is not a service or a manager; it is a **message** that represents a single, atomic change to a world-level *Resource*.

In Hytale's ECS, a Resource is a singleton data structure associated with an entire world, distinct from Components which are attached to individual entities. Examples include global game state like *WorldTime* or *WeatherState*.

By implementing the DataChange interface, ResourceChange objects are designated as formal units of change. This allows them to be consumed by higher-level systems responsible for state replication, persistence, or reacting to game state modifications. The object's design is intentionally lightweight; it signals *that* a change occurred and to *what type* of Resource, but it does not carry the new data payload itself. This separation of concerns prevents data duplication and keeps the change notification system efficient.

The use of generics ensures complete type safety throughout the change processing pipeline, preventing runtime errors when systems attempt to query for the changed Resource.

### Lifecycle & Ownership
- **Creation:** An instance of ResourceChange is created by the system that is authoritative for a given Resource. For example, a `TimeOfDaySystem` would instantiate a `new ResourceChange(ChangeType.ADD, WorldTime.TYPE)` when it first initializes the world's time.
- **Scope:** Extremely short-lived and transactional. A ResourceChange object is designed to exist only for the duration of its processing within a single engine tick. It is created, published to a change tracking system, and immediately becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or `close` methods. Its short scope ensures it does not contribute to long-term memory pressure.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields (`type`, `resourceType`) are declared `final` and are assigned exclusively in the constructor. This is a critical design feature that guarantees the integrity of the change event after its creation.

- **Thread Safety:** **Inherently thread-safe**. Due to its complete immutability, a ResourceChange instance can be safely passed between and read by multiple threads without any external synchronization or locking mechanisms. This is essential for engine components that may process game state changes in parallel, such as networking or logging systems.

## API Surface
The public contract is minimal, focusing on data access. The primary interaction is object creation via its constructor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ResourceChange(type, resourceType) | constructor | O(1) | Creates a new, immutable change event. |
| getType() | ChangeType | O(1) | Returns the nature of the change (e.g., ADD, REMOVE). |
| getResourceType() | ResourceType | O(1) | Returns the type token for the Resource that was affected. |

## Integration Patterns

### Standard Usage
ResourceChange objects are not meant to be used in isolation. They are created and immediately passed to a change management system, such as a `ChangeSet` or a world-level event bus, for processing by other systems.

```java
// A system responsible for the Weather Resource adds it to the world.
// It then creates a ResourceChange to notify other systems.
ChangeSet changeSet = world.getChangeSet();
world.addResource(WeatherState.TYPE, new WeatherState());

// Create and publish the change event.
ResourceChange<MyWorld, WeatherState> change = new ResourceChange<>(
    ChangeType.ADD,
    WeatherState.TYPE
);
changeSet.addChange(change);
```

### Anti-Patterns (Do NOT do this)
- **State Storage:** Do not maintain long-term references to ResourceChange objects. They represent a point-in-time event, not a state. Storing them can lead to memory leaks and logic errors based on stale information.
- **Orphaned Creation:** Do not create a ResourceChange without a corresponding, actual modification to the world's Resource state. The event must always reflect a real change.
- **Modification via Reflection:** Attempting to modify the `final` fields of this object is a severe violation of its design contract. This will break immutability guarantees and lead to unpredictable and unstable engine behavior.

## Data Pipeline
A ResourceChange serves as a message that flows from a state-mutating system to various consumer systems.

> Flow:
> Authoritative System (e.g., WeatherSystem) → **new ResourceChange(...)** → ChangeSet / Event Bus → Consumer System (e.g., ReplicationSystem) → Network Packet Generation

