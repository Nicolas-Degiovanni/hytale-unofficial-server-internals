---
description: Architectural reference for EntityEventType
---

# EntityEventType

**Package:** com.hypixel.hytale.component.event
**Type:** Type Handle

## Definition
```java
// Signature
public class EntityEventType<ECS_TYPE, Event extends EcsEvent> extends EventSystemType<ECS_TYPE, Event, EntityEventSystem<ECS_TYPE, Event>> {
```

## Architecture & Concepts

The EntityEventType class is a specialized, type-safe handle used within the Entity Component System (ECS) to represent a specific category of entity-related events. It is a core component of the engine's event dispatch mechanism, acting as a statically-typed key rather than a service or manager.

Architecturally, it serves as the immutable link between three critical components:
1.  An event data structure (a class extending EcsEvent).
2.  The system responsible for processing that event (a class extending EntityEventSystem).
3.  The central ComponentRegistry that manages all systems.

By encapsulating these type relationships, EntityEventType enables the event bus to be both highly performant and type-safe. Instead of relying on string-based lookups or runtime type-checking, the engine uses instances of EntityEventType as direct, unique identifiers to route a specific event to its designated handler system. This class is a specialization of the more generic EventSystemType, tailored specifically for events that operate within the context of entities.

### Lifecycle & Ownership
-   **Creation:** Instances are created and managed exclusively by the ComponentRegistry during the engine's initialization phase. As the engine scans for and registers all available EntityEventSystem implementations, it creates a corresponding EntityEventType singleton for each.
-   **Scope:** An EntityEventType instance is a static descriptor that persists for the entire application lifetime. It is effectively a constant once created.
-   **Destruction:** The object is garbage collected only when the ComponentRegistry is destroyed during application shutdown. There is no manual destruction logic.

## Internal State & Concurrency
-   **State:** This class is deeply **immutable**. All its internal fields, which hold type metadata and a registry index, are set once in the constructor and never modified. It holds no mutable runtime state, such as event queues or listener lists.
-   **Thread Safety:** EntityEventType is inherently **thread-safe**. Due to its immutability, an instance can be safely shared and used as a key across any number of threads without requiring locks or other synchronization primitives.

## API Surface

The public contract of this class is its constructor, which is invoked internally by the engine. It exposes no public methods of its own; its value is derived from its identity and the type information it represents. All functional behavior is inherited from its parent, EventSystemType, and is not intended for direct invocation by game code.

## Integration Patterns

### Standard Usage
Developers do not instantiate or directly interact with this class. Instead, they interact with the event system *through* the handles that the engine provides. The primary pattern is to define a custom event and system, and then use the engine-provided type to dispatch instances of that event.

```java
// 1. Define a custom event data structure
public class PlayerDamageEvent extends EcsEvent {
    public final int damageAmount;
    // constructor...
}

// 2. Define the system to handle it
public class PlayerDamageSystem extends EntityEventSystem<ClientECS, PlayerDamageEvent> {
    // system logic...
}

// 3. The engine automatically creates an EntityEventType for this pair.

// 4. Elsewhere, dispatch an event using the type as a key.
// Note: The actual API to get EVENT_TYPE may differ.
EntityId targetPlayer = ...;
context.getEventBus().dispatch(targetPlayer, PlayerDamageEvent.EVENT_TYPE, new PlayerDamageEvent(10));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new EntityEventType(...)`. These handles are managed exclusively by the ComponentRegistry. Manual creation will result in an unregistered, non-functional event type that the engine cannot route.
-   **Type Casting:** Avoid casting an EntityEventType to a different generic specialization. This violates the type safety the class is designed to enforce and will lead to runtime ClassCastExceptions.

## Data Pipeline

EntityEventType does not process data itself. It is a metadata object used to **direct the flow of data** within the event bus. It acts as a routing key.

> Flow:
> Event Bus `dispatch` call -> **EntityEventType** (used as lookup key) -> ComponentRegistry -> Target EntityEventSystem -> System processes the EcsEvent payload

