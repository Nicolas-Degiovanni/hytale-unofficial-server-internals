---
description: Architectural reference for WorldEventType
---

# WorldEventType

**Package:** com.hypixel.hytale.component.event
**Type:** Descriptor / Value Object

## Definition
```java
// Signature
public class WorldEventType<ECS_TYPE, Event extends EcsEvent> extends EventSystemType<ECS_TYPE, Event, WorldEventSystem<ECS_TYPE, Event>> {
```

## Architecture & Concepts
The WorldEventType class is a foundational component of the Hytale Entity Component System (ECS) event bus. It is not a service or a manager, but rather a **type-safe descriptor** or **type token**. Its primary function is to create a static, compile-time-checked link between a specific event class (e.g., PlayerDamageEvent) and the system responsible for processing it (e.g., PlayerDamageSystem).

In a highly dynamic system like an ECS, events are often dispatched using integer IDs or string names, which can lead to runtime errors. WorldEventType solves this by encapsulating the event type, the processing system type, and its unique registration index into a single, immutable object. This object then serves as a unique key for retrieving event systems and dispatching events, guaranteeing that the correct system receives the correct event type.

It acts as a piece of static metadata, established at engine initialization, that provides the runtime with the necessary information to route events efficiently and safely.

### Lifecycle & Ownership
- **Creation:** Instances are created once during the engine's bootstrap or component registration phase. They are intended to be defined as static final fields in a central registry or constants class. They are **not** created dynamically during gameplay.
- **Scope:** Application-scoped. A WorldEventType instance persists for the entire lifetime of the client or server process. It represents a permanent definition of an event.
- **Destruction:** Instances are garbage collected only upon final process shutdown. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields, including the component registry, system class, event class, and index, are set via the constructor and cannot be modified thereafter. This object is a pure data container for type information.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a WorldEventType instance can be safely accessed and read by any number of threads simultaneously without requiring locks or other synchronization primitives. This is critical for multi-threaded game logic where various systems may need to query event metadata.

## API Surface
The public contract is exclusively for definition and is centered on the constructor. The object itself is used as a parameter for other systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldEventType(registry, tClass, eClass, index) | Constructor | O(1) | Constructs a new event type definition. This is the sole entry point for creating an instance. |

## Integration Patterns

### Standard Usage
The correct pattern is to define all WorldEventType instances as static final constants. These constants are then used throughout the codebase to refer to the event type in a safe manner.

```java
// In a central constants or registry class
public static final WorldEventType<MyEcs, PlayerJumpEvent> PLAYER_JUMP =
    new WorldEventType<>(
        ComponentRegistries.WORLD,
        PlayerJumpSystem.class,
        PlayerJumpEvent.class,
        EventIndices.NEXT++
    );

// In game logic, to dispatch an event
WorldEventSystem<MyEcs, PlayerJumpEvent> jumpSystem = world.getEventSystem(PLAYER_JUMP);
jumpSystem.fire(new PlayerJumpEvent(entityId, jumpHeight));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new WorldEventType()` within game logic loops or methods. These objects are definitions, not runtime data. Creating them dynamically will pollute the system and bypass the registration process.
- **Index Mismanagement:** Manually providing an index that is already in use will cause catastrophic aliasing issues, where one event type incorrectly maps to another's system. Always use a centralized, atomic index allocator.

## Data Pipeline
WorldEventType does not process data itself. Instead, it acts as a **routing key** that directs data (the event object) to the correct processor (the event system).

> Flow:
> Event Object Created -> **WorldEventType (as Key)** -> WorldEventSystem Lookup -> System Processes Event Object

