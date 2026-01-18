---
description: Architectural reference for Holder
---

# Holder

**Package:** com.hypixel.hytale.component
**Type:** Transient State Container

## Definition
```java
// Signature
public class Holder<ECS_TYPE> {
```

## Architecture & Concepts
The Holder is the canonical data container for a single entity within the Hytale Entity-Component-System (ECS) framework. It does not represent the entity's identity (e.g., a numeric ID), but rather encapsulates the complete, mutable state of that entity by managing its collection of components.

This class is a cornerstone of the ECS engine, providing a high-performance, thread-safe mechanism for systems to interact with entity data. Its design is predicated on two core ECS optimization patterns:

1.  **Archetype-based Management:** Each Holder maintains a reference to an Archetype. The Archetype is an immutable metadata object that represents the unique *set* of component types attached to the entity. Systems can query entities based on their Archetype for extremely fast filtering, as it avoids iterating over every component on every entity.
2.  **Indexed Component Storage:** The actual component instances are stored in a simple array, `Component<ECS_TYPE>[]`. The index for each component in this array is determined by its globally unique ComponentType index. This provides O(1) access time when retrieving a component if its type is known.

The Holder acts as the synchronization point between various game systems—such as physics, AI, and rendering—that may operate on the same entity from different threads. Its internal locking mechanism is critical for maintaining data consistency in a multi-threaded engine.

### Lifecycle & Ownership
-   **Creation:** A Holder is instantiated and managed exclusively by the ECS framework, typically via a `ComponentRegistry` or an `EntityManager` when a new entity is spawned. Direct instantiation by client code is a critical error.
-   **Scope:** The lifecycle of a Holder is strictly bound to the lifecycle of the entity it represents. It is created when the entity is created and persists for as long as the entity exists within the game world.
-   **Destruction:** The Holder is eligible for garbage collection when its associated entity is destroyed and all framework-level references to it are cleared. It does not implement any explicit `close` or `destroy` methods.

## Internal State & Concurrency
-   **State:** The Holder's state is highly mutable. The internal `archetype` and `components` array are frequently reconfigured at runtime as components are added to or removed from the entity. It is the authoritative source of truth for an entity's component data.
-   **Thread Safety:** This class is fundamentally thread-safe. All public methods that access or mutate its internal state are protected by a `java.util.concurrent.locks.StampedLock`.
    -   **Read Operations:** Methods like `getComponent` and `clone` acquire a non-blocking read lock, allowing multiple threads to read an entity's state concurrently without contention.
    -   **Write Operations:** Methods like `addComponent`, `removeComponent`, and `updateData` acquire an exclusive write lock, ensuring that all modifications are atomic and visible to all threads once complete.

    **WARNING:** While the Holder itself is thread-safe, the Component objects it contains may not be. It is the responsibility of the system developer to ensure that modifications to component data are performed in a thread-safe manner.

## API Surface
The public API provides atomic operations for querying and manipulating an entity's component set.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponent(type) | T | O(1) | Retrieves a component instance by its type. Returns null if not present. This is the primary read operation. |
| addComponent(type, component) | void | O(1) | Adds a new component. Throws an exception if a component of that type already exists. |
| removeComponent(type) | void | O(1) | Removes a component by its type. The component instance is nulled in the internal array. |
| replaceComponent(type, component) | void | O(1) | Overwrites an existing component instance with a new one. |
| putComponent(type, component) | void | O(1) | A convenience method that adds a component if absent or replaces it if present. |
| ensureAndGetComponent(type) | T | O(1) | Retrieves a component, creating and adding a default instance via the ComponentRegistry if it does not exist. |
| updateData(oldData, newData) | void | O(N) | A high-overhead operation used for runtime schema migration. Reconciles component state when component types are dynamically registered or unregistered. |
| clone() | Holder | O(M) | Creates a deep copy of the Holder and all its components. M is the number of components. |

## Integration Patterns

### Standard Usage
Systems should always retrieve a Holder from a central entity manager or world context. The Holder is the gateway for all component data access for a given entity.

```java
// A system retrieves a Holder for an entity it needs to process.
Holder<GameEcsType> holder = world.getComponentHolder(entityId);

// Read a component's data
PositionComponent pos = holder.getComponent(POSITION_TYPE);
if (pos != null) {
    // Perform logic based on the position data
    pos.setX(pos.getX() + 1.0f);
}

// Add a new component to change entity behavior
if (shouldApplyEffect) {
    holder.addComponent(FROZEN_EFFECT_TYPE, new FrozenEffect(10.0f));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new Holder()`. Holders are managed by the engine's core ECS framework. Direct instantiation will result in a non-functional object that is not tracked by any systems.
-   **Caching Holders:** Do not store references to Holders in systems across multiple game ticks. The entity could be destroyed, leading to a memory leak or attempts to operate on stale data. Always re-fetch the Holder from the world or entity manager at the start of a processing cycle.
-   **External Locking:** Do not attempt to acquire the Holder's internal lock externally. The public API methods provide all necessary atomicity guarantees.

## Data Pipeline
The Holder is a key participant in the entity serialization and data migration pipelines.

> **Serialization Flow:**
> **Holder** -> `createComponentsMap()` -> `Map<String, Component>` -> BSON Codec -> Serialized BSON Document

> **Deserialization Flow:**
> Serialized BSON Document -> BSON Codec -> `Map<String, Component>` -> `loadComponentsMap()` -> **Holder**

> **Schema Migration Flow:**
> ComponentRegistry Change -> Engine invokes `updateData()` on **Holder** -> Components are remapped or moved to an `UnknownComponents` bucket -> State is reconciled with the new component schema.

