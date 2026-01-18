---
description: Architectural reference for UUIDComponent
---

# UUIDComponent

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Data Component

## Definition
```java
// Signature
public final class UUIDComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The UUIDComponent is a fundamental data component within Hytale's Entity Component System (ECS). Its sole responsibility is to attach a stable, universally unique identifier to a server-side entity. This identifier is the canonical reference for an entity, critical for systems that require persistent and unambiguous lookups, such as database storage, network replication, and inter-system event messaging.

As an implementation of the `Component` interface, it is a passive data container managed by the core entity systems. Its architecture is heavily influenced by the engine's data-driven design, evidenced by the static `CODEC` field. This `BuilderCodec` instance defines the complete serialization and deserialization contract for the component, allowing the engine to transparently save entity state to an `EntityStore` or transmit it over the network without custom logic.

The `afterDecode` hook within the codec definition is a key resilience pattern. It ensures that any entity being loaded from a data source that lacks a UUID will have one automatically generated, preventing state corruption and guaranteeing identity.

### Lifecycle & Ownership
- **Creation:** A UUIDComponent is instantiated and attached to an entity when that entity is first created. This is typically handled by an entity factory or spawner. The static factory methods `generateVersion3UUID` and `randomUUID` provide controlled entry points for this process. It is also created during deserialization by the `CODEC`.
- **Scope:** The lifecycle of a UUIDComponent is strictly bound to its parent entity. It exists for the entire duration that the entity is active in the world.
- **Destruction:** The component is marked for garbage collection when its parent entity is destroyed and removed from the `EntityStore`. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The component's state is immutable. It consists of a single private `UUID` field which is set only at creation time (either via the constructor or the deserialization codec). There are no public setters.
- **Thread Safety:** The component is inherently thread-safe for read operations due to its immutability. All state is established in a single atomic step during construction. The `clone` method's implementation, which returns the same instance, is a critical optimization that relies on this immutability.

**WARNING:** Systems should never attempt to modify a UUIDComponent's state via reflection. The entity's identity is considered permanent and changing it post-creation will lead to severe data desynchronization and lookup failures.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | Static method to retrieve the component's registered type from the EntityModule. |
| getUuid() | UUID | O(1) | Returns the unique identifier for the entity. |
| clone() | Component | O(1) | Returns the same instance. Relies on the component's immutability. |
| generateVersion3UUID() | UUIDComponent | O(1) | Static factory to create a component with a new version 3 (name-based) UUID. |
| randomUUID() | UUIDComponent | O(1) | Static factory to create a component with a new version 4 (random) UUID. |

## Integration Patterns

### Standard Usage
The component is typically added to an entity upon its creation. Other systems then query the entity for this component to retrieve its canonical identifier for cross-referencing.

```java
// Example of a system assigning a UUID to a newly created entity
Entity newEntity = world.createEntity();
UUIDComponent id = UUIDComponent.randomUUID();
newEntity.addComponent(id);

// Later, another system can retrieve the ID
UUID entityId = newEntity.getComponent(UUIDComponent.class).getUuid();
database.savePlayerData(entityId, playerData);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Cloning:** Do not call `clone()` expecting a new, distinct object. The method returns `this` as an optimization. Code that relies on a deep copy will fail.
- **Manual Instantiation via Private Constructor:** The private constructor is reserved for the `CODEC` system. Use the provided static factory methods for all programmatic creation.
- **Identity Reassignment:** Do not remove a UUIDComponent from an entity to replace it with another. An entity's UUID is its permanent identity. Reassigning it will break all existing references in other systems, save files, and network caches.

## Data Pipeline
The UUIDComponent is a critical payload in the entity persistence and replication data pipeline. Its flow is governed by the engine's serialization framework.

> **Serialization Flow:**
> Entity is marked for saving -> `EntityStore` iterates components -> `UUIDComponent.CODEC` is invoked -> The `UUID` is written as a 16-byte binary array -> Data is committed to disk.

> **Deserialization Flow:**
> Raw binary data from disk -> `EntityStore` reads entity data -> `UUIDComponent.CODEC` is invoked -> A new `UUIDComponent` is instantiated -> The `afterDecode` hook runs, generating a UUID if one was missing -> The component is attached to the loaded entity.

