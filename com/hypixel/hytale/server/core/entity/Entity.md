---
description: Architectural reference for the Entity class, the foundational object for all in-world actors.
---

# Entity

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Base Class / Component

## Definition
```java
// Signature
public abstract class Entity implements Component<EntityStore> {
```

## Architecture & Concepts

The Entity class is the abstract base for all dynamic objects within a Hytale world, such as players, creatures, and items. It serves as the central identity and container for a collection of data-oriented Components within the server's Entity-Component-System (ECS) architecture.

Architecturally, an Entity is not a monolithic object that contains all its own logic and data. Instead, it acts as a lightweight identifier that logically groups a set of disparate Components. The actual data, such as position (TransformComponent) or health, resides within these components, which are managed externally by the World's EntityStore.

This class is in a state of significant architectural transition. It contains numerous deprecated fields and methods (e.g., legacyUuid, transformComponent) that represent a bridge from a more traditional object-oriented model. The modern, canonical approach is to treat the Entity itself as a specialized Component and interact with its data through the ECS registry, not through direct method calls on the Entity object. Its primary modern responsibilities are lifecycle management (creation, removal) and identity (networkId, reference).

## Lifecycle & Ownership

The lifecycle of an Entity is strictly managed by the World it inhabits. Direct manipulation of an Entity's lifecycle outside of the World's control will lead to state corruption and system instability.

- **Creation:** Entities are never instantiated directly via their constructor. They are created by specialized factories or systems, such as the EntityModule, which handles registration and initial component attachment. The `clone` method provides a deep-copy mechanism based on serialization, primarily for internal system use.

- **Scope:** An Entity's lifetime is coupled to its presence in a World. It is loaded via `loadIntoWorld` and persists until it is explicitly removed. The `reference` field, a `Ref<EntityStore>`, is the canonical pointer to this Entity within the ECS, and its validity is tied to the Entity's presence in the world.

- **Destruction:** The `remove` method is the sole entry point for destroying an Entity. This operation is idempotent and thread-safe regarding the removal flag itself. It triggers the `EntityRemoveEvent`, notifies the `EntityStore` to deallocate all associated components, and marks the Entity as removed. Any attempt to interact with an Entity after `remove` has been called is an error. The `removedBy` field captures a stack trace for debugging invalid or unexpected removals.

## Internal State & Concurrency

- **State:** The Entity class maintains highly mutable state critical to its identity and lifecycle, including its `world`, `networkId`, and `reference`. The most critical state variable is the `wasRemoved` AtomicBoolean, which governs its active status.

- **Thread Safety:** This class is **not thread-safe** and is designed to be accessed exclusively from its owning World's main ticking thread. The codebase contains explicit assertions (`world.debugAssertInTickingThread()`) to enforce this contract. The single exception is the `wasRemoved` flag, which is an AtomicBoolean to prevent race conditions during the removal process itself. Asynchronous access to any other state or method, particularly `getTransformComponent`, will result in warnings and potential state corruption. Operations that must be performed from other threads should be scheduled to run on the world thread.

## API Surface

The public API is focused on lifecycle, identity, and integration with the ECS. Most data-access methods are deprecated in favor of direct component lookups.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| remove() | boolean | O(1) | Initiates the entity removal process. Dispatches events and notifies the EntityStore. Returns false if already removed. **Must be called on the world thread.** |
| loadIntoWorld(World) | void | O(1) | Associates the entity with a World and assigns a network ID. Throws IllegalArgumentException if already in a world. |
| unloadFromWorld() | void | O(1) | Disassociates the entity from its World. Throws IllegalArgumentException if not in a world. |
| getWorld() | World | O(1) | Returns the World this entity currently exists in, or null if it is not loaded. |
| getReference() | Ref<EntityStore> | O(1) | Returns the canonical, strong reference to this entity within the ECS. May be null if not yet registered. |
| wasRemoved() | boolean | O(1) | Safely checks if the entity has been marked for removal. This is one of the few methods safe to call from any thread. |
| toHolder() | Holder<EntityStore> | O(N) | Serializes the entity and all its components into a generic Holder object. Complexity is proportional to the number of components. |

## Integration Patterns

### Standard Usage

Entities should never be managed directly. Always interact with them through the World or a ComponentAccessor, treating the Entity object primarily as an identifier to look up its associated data Components.

```java
// Correctly finding an entity and accessing its data
World world = ...;
Ref<EntityStore> entityRef = findEntityReference(); // Assume this method exists

// The ComponentAccessor is the gateway to data
ComponentAccessor<EntityStore> accessor = world.getEntityStore().getStore();
TransformComponent transform = accessor.getComponent(entityRef, TransformComponent.getComponentType());

if (transform != null) {
    // Mutate data on the component, not the entity
    transform.getPosition().add(0, 1, 0);
}
```

### Anti-Patterns (Do NOT do this)

- **Direct Instantiation:** Never call `new Player()` or `new Monster()`. This bypasses the World's entity management system, resulting in an object that is not tracked, ticked, or networked.

- **Asynchronous Modification:** Do not get an Entity object on the main thread and then pass it to another thread to modify its state or components. This will violate threading contracts and cause crashes or data corruption.

- **State Caching:** Do not cache the return value of methods like `getTransformComponent()`. The underlying component can be moved or deallocated by the ECS. Always re-fetch components from a `ComponentAccessor` when needed.

- **Post-Removal Interaction:** Do not hold a reference to an Entity and interact with it after `remove()` has been called. Always check `wasRemoved()` if the validity of the entity is uncertain.

## Data Pipeline

The Entity serves as a logical nexus in several data flows, most notably serialization and networking.

> **Serialization (Saving/Cloning) Flow:**
> `Entity.toHolder()` -> `BuilderCodec` -> Serializes all attached Components -> BsonDocument

> **Lifecycle (Removal) Flow:**
> `Entity.remove()` -> `EntityRemoveEvent` -> Event Bus -> `EntityStore.removeEntity()` -> Component Deallocation

