---
description: Architectural reference for ComponentAccessor
---

# ComponentAccessor

**Package:** com.hypixel.hytale.component
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface ComponentAccessor<ECS_TYPE> {
```

## Architecture & Concepts
The ComponentAccessor interface defines the primary contract for interacting with an Entity-Component-System (ECS) world. It serves as a high-level facade, abstracting the underlying data structures and memory layout of entities and their associated components. This is the central API through which all game logic, encapsulated within Systems, reads, modifies, and creates ECS data.

This interface is not a concrete implementation but a blueprint. Systems are designed to operate against this contract, making them agnostic to the specific ECS storage strategy (e.g., Archetype-based, sparse-set). The generic parameter, ECS_TYPE, acts as a type-safe marker, allowing the engine to manage multiple, distinct ECS worlds (for instance, a client-side world and a server-side world) without ambiguity.

Any object that provides access to entities, components, and resources for a given ECS instance must implement this interface. It is the sole entry point for mutations and queries, ensuring that all interactions with the ECS state are managed and potentially tracked.

## Lifecycle & Ownership
As an interface, ComponentAccessor itself has no lifecycle. The lifecycle and ownership semantics apply to the **concrete class that implements it**, which is typically the main ECS World or a context object.

- **Creation:** An implementation of ComponentAccessor is instantiated alongside its corresponding ECS World. This typically occurs during major engine initialization phases, such as server startup or client world loading.
- **Scope:** The accessor's lifetime is bound to the lifetime of the ECS World it represents. It persists as long as that world is active and loaded in memory. Systems are granted temporary, often frame-scoped, access to this object.
- **Destruction:** The object is eligible for garbage collection when the ECS World it services is unloaded or destroyed.

**WARNING:** Systems should never cache a ComponentAccessor instance. The provided instance may be a temporary, transactional proxy. Always use the instance passed into the system's execution method for the current tick.

## Internal State & Concurrency
The ComponentAccessor interface is stateless by definition. All state is managed by the underlying ECS World implementation.

- **State:** The implementing class is highly stateful, managing the memory for all entities, components, and resources. The accessor is merely a window into this state.
- **Thread Safety:** Implementations of this interface are **not guaranteed to be thread-safe**. In a multi-threaded engine, access is typically synchronized externally by the System scheduler. For example, systems that modify the same component data are scheduled sequentially, while read-only systems may run in parallel. Direct, unsynchronized calls to a ComponentAccessor from multiple threads will lead to race conditions, data corruption, and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponent(ref, type) | T | O(1) | Retrieves a component of a given type for an entity. Returns null if the entity does not have the component. |
| ensureAndGetComponent(ref, type) | T | O(1) | Retrieves a component; if it does not exist, it is created with default values and attached to the entity before being returned. |
| getArchetype(ref) | Archetype | O(1) | Returns the structural archetype of the entity, defining its collection of component types. |
| getResource(type) | T | O(1) | Retrieves a world-global singleton object, known as a Resource. Throws an exception if the resource does not exist. |
| addEntity(holder, reason) | Ref | O(N) | Creates a new entity from a Holder definition. Complexity depends on archetype creation and memory allocation. |
| removeEntity(ref, holder, reason) | Holder | O(N) | Destroys an entity and returns its data in a Holder. Complexity depends on memory deallocation and archetype shifts. |
| addComponent(ref, type, instance) | void | O(N) | Adds a pre-constructed component to an entity. This may trigger an entity archetype change, which is a costly operation. |
| removeComponent(ref, type) | void | O(N) | Removes a component from an entity. This may trigger an entity archetype change. |
| invoke(target, event) | void | O(M) | Dispatches an event targeted at a specific entity or the entire world. Complexity depends on the number of subscribed listeners. |

## Integration Patterns

### Standard Usage
The ComponentAccessor is provided to a System by the engine's scheduler during its execution phase. The system uses the accessor to query for entities with specific components, read their data, perform logic, and write back results.

```java
// Inside a System's execute method
public void execute(ComponentAccessor<Client> accessor, Query<Client> query) {
    // Get a global resource like the game timer
    GameTime time = accessor.getResource(GameTime.class);

    // Iterate through entities matched by the query
    for (Ref<Client> entityRef : query) {
        // Read data from one component
        Position pos = accessor.getComponent(entityRef, Position.class);

        // Ensure another component exists, then modify it
        Velocity vel = accessor.ensureAndGetComponent(entityRef, Velocity.class);
        vel.setX(vel.getX() + 1.0f * time.getDelta());

        // Dispatch an event for other systems to react to
        accessor.invoke(new EntityMovedEvent(entityRef));
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Caching the Accessor:** Do not store the ComponentAccessor in a field of your System. The engine may provide a different or decorated instance on each tick.
- **Cross-Thread Access:** Do not share the accessor across threads you manage yourself. All interactions must be performed on the thread managed by the ECS scheduler to prevent data corruption.
- **Ignoring Archetype Changes:** Repeatedly adding and removing components to the same entity within a single frame is extremely inefficient. Each such operation can cause the entity's data to be moved in memory. Batch structural changes where possible.

## Data Pipeline
The ComponentAccessor is the central gateway for all data flowing into and out of the ECS world state. It translates high-level semantic operations into low-level memory manipulations.

> **Read Flow:**
> System Logic -> **ComponentAccessor.getComponent()** -> ECS World (Archetype Lookup) -> Raw Component Memory -> Deserialized Component

> **Write Flow:**
> System Logic -> **ComponentAccessor.addComponent()** -> ECS World (Archetype Change) -> Memory Reallocation & Copy -> New World State

> **Event Flow:**
> System Logic -> **ComponentAccessor.invoke()** -> ECS Event Bus -> Subscribed Systems -> Further State Changes via ComponentAccessor

