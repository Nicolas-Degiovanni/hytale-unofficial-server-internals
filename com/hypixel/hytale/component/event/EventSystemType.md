---
description: Architectural reference for EventSystemType
---

# EventSystemType

**Package:** com.hypixel.hytale.component.event
**Type:** Abstract Type Definition

## Definition
```java
// Signature
public abstract class EventSystemType<ECS_TYPE, Event extends EcsEvent, SYSTEM_TYPE extends EventSystem<Event> & ISystem<ECS_TYPE>>
   extends SystemType<ECS_TYPE, SYSTEM_TYPE> {
```

## Architecture & Concepts
EventSystemType is a foundational metadata class within the Entity Component System (ECS) framework. It extends the generic SystemType to create a specialized, type-safe descriptor for systems that are designed to process a single, specific type of event.

Its primary architectural role is to act as a "type token" or a key within the ComponentRegistry. It establishes an immutable, compile-time link between three distinct concepts:
1.  An ECS entity context (ECS_TYPE).
2.  A specific event class that inherits from EcsEvent (Event).
3.  A specific system class that implements EventSystem (SYSTEM_TYPE).

The engine's event dispatcher uses instances of EventSystemType to look up the correct system responsible for handling a given event. The overridden `isType` method provides a stricter contract than its parent, ensuring not only that a system is of the correct class, but also that it is explicitly configured to handle the exact event type defined by this EventSystemType. This design prevents ambiguity and runtime errors when multiple systems might otherwise match a less specific query.

## Lifecycle & Ownership
-   **Creation:** Instances are not created dynamically during gameplay. Concrete subclasses of EventSystemType are expected to be defined as static final constants in a central registry (e.g., a hypothetical `SystemTypes` class) and instantiated once during engine bootstrap. The constructor is protected to enforce this pattern.
-   **Scope:** An EventSystemType instance is a static descriptor. Once created, it persists for the entire lifetime of the application. It carries no per-session or per-world state.
-   **Destruction:** Instances are reclaimed by the JVM during application shutdown. There is no manual destruction or cleanup process.

## Internal State & Concurrency
-   **State:** EventSystemType is **immutable**. Its internal state, including the reference to the event class (`eClass`), is set once in the constructor and cannot be changed. This immutability is critical for its role as a reliable key in a potentially concurrent registry system.
-   **Thread Safety:** This class is **inherently thread-safe**. Due to its immutable nature, a single instance can be safely accessed and read by multiple threads simultaneously without any need for external locking or synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEventClass() | Class<Event> | O(1) | Returns the concrete event class this type is associated with. |
| isType(ISystem system) | boolean | O(1) | Performs a strict type check. Returns true only if the provided system is of the correct class *and* handles the exact event type represented by this instance. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay logic developers. It is a core component of the engine's internal registry and event dispatching machinery. The engine uses it as a key to retrieve the correct system for an event.

```java
// Conceptual: How the engine's ComponentRegistry might use a static EventSystemType instance
// Assume PlayerJoinEventType is a concrete, static final instance of EventSystemType.

public EventSystem findSystemForEvent(EcsEvent event) {
    // 1. A static EventSystemType instance is retrieved, perhaps from a map.
    EventSystemType type = SystemTypes.forEvent(event.getClass());

    // 2. The type is used to query the registry.
    // The registry iterates its systems, calling type.isType(system) on each one.
    return componentRegistry.findSystem(type);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to create instances of EventSystemType or its subclasses dynamically. They are meant to be static, singleton-like constants. The protected constructor enforces this.
-   **Stateful Subclasses:** Do not create subclasses that add mutable state. This violates the core design principle of EventSystemType as an immutable descriptor and would break thread safety.
-   **Incorrect Type Association:** Using an EventSystemType to look up a system that does not implement EventSystem will always fail the `isType` check and return no results.

## Data Pipeline
EventSystemType does not process data itself; it is a critical piece of metadata that directs the flow of event data within the ECS. It acts as a routing instruction for the event bus.

> Flow:
> EcsEvent Fired -> Event Bus -> **EventSystemType** (Used as a lookup key) -> ComponentRegistry -> Target EventSystem -> Event Processed by System Logic

