---
description: Architectural reference for PrefabPlaceEntityEvent
---

# PrefabPlaceEntityEvent

**Package:** com.hypixel.hytale.server.core.prefab.event
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPlaceEntityEvent extends EcsEvent {
```

## Architecture & Concepts
The PrefabPlaceEntityEvent is a command-style event within the server-side Entity Component System (ECS). It represents a deferred request to instantiate a new entity from a predefined template, known as a prefab.

This class acts as a message, decoupling the system that requests an entity's creation from the system responsible for the complex logic of prefab instantiation. By encapsulating the *what* (the prefabId) and the *where* (the target EntityStore), it provides a clean, transactional mechanism for world population and dynamic entity spawning.

Its inheritance from EcsEvent signals that it is intended to be processed by one or more registered ECS systems that listen on the main event bus. The use of a Holder for the EntityStore is a critical design choice, ensuring that the target storage context remains valid and accessible throughout the event's lifecycle, even in highly concurrent or asynchronous scenarios.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by high-level game logic systems. Common sources include world generation routines, monster spawners, player actions (e.g., placing a block that contains an entity), or scripted quest triggers.
- **Scope:** Extremely short-lived. The event object exists only for the duration of a single processing tick. Its scope is confined to its transit through the event bus and its consumption by a handler system.
- **Destruction:** The object becomes eligible for garbage collection immediately after all subscribed systems have processed it. It holds no persistent state or external resources requiring manual cleanup.

**WARNING:** Systems must not retain references to this event object across ticks. Doing so constitutes a memory leak and may hold stale references to world state.

## Internal State & Concurrency
- **State:** Immutable. All internal fields are declared as final and are initialized exclusively during construction. The state of a PrefabPlaceEntityEvent instance cannot be altered after it is created.
- **Thread Safety:** Inherently thread-safe. Due to its immutability, an instance of this event can be safely read by multiple threads without any need for external synchronization or locks. However, the systems that *process* this event are responsible for ensuring their own thread safety when modifying the shared world state (e.g., the EntityStore).

## API Surface
The public API is minimal, providing read-only access to the event's payload.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPrefabId() | int | O(1) | Returns the unique identifier for the prefab to be instantiated. |
| getHolder() | Holder<EntityStore> | O(1) | Returns the holder for the target EntityStore where the new entity will be created. |

## Integration Patterns

### Standard Usage
This event is designed to be created and immediately dispatched to an event bus or context, which then routes it to the appropriate processing system.

```java
// Within a game logic system (e.g., a spawner)
// Assume 'context' provides access to the event bus and world data

void spawnCreature(WorldContext context, Holder<EntityStore> targetLocation) {
    int monsterPrefabId = PrefabRegistry.getId("hytale:boar");

    // Create the event with the necessary data
    PrefabPlaceEntityEvent spawnEvent = new PrefabPlaceEntityEvent(monsterPrefabId, targetLocation);

    // Dispatch the event for processing by other systems
    context.getEventBus().post(spawnEvent);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not add this event to a collection or cache it for later use. It represents a point-in-time request and should be processed immediately. Storing it can lead to operations being performed on a stale or invalid EntityStore.
- **State Modification:** Do not attempt to modify the event's internal state using reflection. The immutability contract is critical for system stability.
- **Manual Processing:** Do not call a processing system directly with this event. Always dispatch it through the central event bus to ensure all relevant listeners and interceptors are triggered in the correct order.

## Data Pipeline
The PrefabPlaceEntityEvent is a data-transfer object that initiates a clear sequence of operations within the server's ECS.

> Flow:
> Game Logic System -> **new PrefabPlaceEntityEvent(id, holder)** -> Event Bus -> PrefabInstantiationSystem -> EntityFactory -> New Entity added to EntityStore

