---
description: Architectural reference for Archetype
---

# Archetype

**Package:** com.hypixel.hytale.component
**Type:** Value Object / Definition

## Definition
```java
// Signature
public class Archetype<ECS_TYPE> implements Query<ECS_TYPE> {
```

## Architecture & Concepts
The Archetype is a foundational concept within Hytale's Entity Component System (ECS). It is not an object that holds game state, but rather an immutable **schema** or **blueprint** that defines a unique combination of component types. Every entity in the game world that possesses the exact same set of components belongs to the same single Archetype.

This class is the cornerstone of the engine's data-oriented design. By grouping all entities of the same Archetype into contiguous memory blocks, the ECS can perform queries and system updates with extreme efficiency. Iterating over all entities with a `Position` and `Velocity` component becomes a simple, cache-friendly traversal of linear memory, avoiding the pointer-chasing and cache misses common in object-oriented entity models.

An Archetype is essentially a specialized, sparse set of `ComponentType`s, optimized for O(1) lookups. Its internal representation is an array where the index corresponds to the `ComponentType`'s unique ID. This allows for near-instantaneous checks to determine if an Archetype contains a specific component.

The generic parameter `ECS_TYPE` serves as a phantom type, ensuring that Archetypes and `ComponentType`s from different ECS worlds (e.g., client vs. server) are not accidentally mixed, providing compile-time safety.

## Lifecycle & Ownership
- **Creation:** Archetypes are never instantiated directly via their constructor. They are created exclusively through a set of static factory methods: `of`, `add`, and `remove`. The ECS framework's `EntityManager` is the primary consumer of these methods. When an entity has a component added or removed, the manager computes a new Archetype by calling these factories on the entity's previous Archetype. The static `EMPTY` instance serves as a flyweight for entities with no components.

- **Scope:** An Archetype is an immutable, canonical representation of a component set. Once created for a specific combination, it is cached and reused by the ECS framework for all entities matching that signature. Its lifetime is tied to the `ComponentRegistry` from which its `ComponentType`s originate, effectively persisting as long as the parent ECS world is active.

- **Destruction:** There is no explicit destruction method. Archetype instances are managed by the Java garbage collector. They become eligible for collection when the ECS world is shut down and all references from the `EntityManager` and `ComponentRegistry` are released.

## Internal State & Concurrency
- **State:** The Archetype class is **strictly immutable**. All internal fields, including the `componentTypes` array, are considered final upon construction. Methods like `add` and `remove` do not modify the instance they are called on; instead, they return a **new** Archetype instance representing the result of the operation. This design guarantees that an Archetype's definition is stable and predictable throughout its lifetime.

- **Thread Safety:** As a direct result of its immutability, the Archetype class is **inherently thread-safe**. Instances can be freely shared and read across multiple threads without any need for locks or other synchronization primitives. This is critical for the performance of the engine's multi-threaded job system, which may query the ECS structure from various worker threads simultaneously.

## API Surface
The public API is dominated by static factory methods for creation and instance methods for inspection and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(ComponentType...) | static Archetype | O(N) | Creates a new Archetype from a set of ComponentTypes. Throws if types are duplicated or from different registries. |
| add(Archetype, ComponentType) | static Archetype | O(M) | Returns a new Archetype representing the union of the base Archetype and the new ComponentType. M is the max component index. |
| remove(Archetype, ComponentType) | static Archetype | O(M) | Returns a new Archetype representing the base Archetype with the specified ComponentType removed. M is the max component index. |
| contains(ComponentType) | boolean | O(1) | Performs a highly optimized check to see if the Archetype includes the given ComponentType. |
| contains(Archetype) | boolean | O(N) | Checks if this Archetype is a superset of another Archetype. N is the number of components in the parameter. |
| validateComponents(...) | void | O(N) | Validates that an array of Component instances conforms exactly to this Archetype's schema. Throws on mismatch. |
| getSerializableArchetype(...) | Archetype | O(N) | Computes and returns a new, potentially smaller Archetype containing only the serializable components of this one. |
| asExactQuery() | ExactArchetypeQuery | O(1) | Returns a query object that will match entities belonging to *exactly* this Archetype, and no others. |

## Integration Patterns

### Standard Usage
Developers typically do not interact with the Archetype class directly. Its primary role is internal to the ECS framework. The most common developer-facing interaction is implicitly defining an Archetype when creating a `Query`.

```java
// A developer defines a query for entities that have at least Position and Velocity.
// Under the hood, the ECS framework creates an Archetype to represent this query.
Query query = Archetype.of(
    componentRegistry.getPositionType(),
    componentRegistry.getVelocityType()
);

// The engine's QueryProcessor then uses this query Archetype to efficiently
// find all world Archetypes that are a superset of it.
world.query(query).forEach(entity -> {
    // Process entities
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private and must not be accessed. Always use the static factory methods `of`, `add`, or `remove`.
- **Mixing Registries:** Attempting to create an Archetype using `ComponentType`s from two different `ComponentRegistry` instances will result in a runtime exception. All components within an Archetype must belong to the same ECS world.
- **Assuming Mutability:** Never expect an Archetype object to change. Operations that add or remove components always produce a new instance. Caching a reference to an Archetype and expecting it to update is a critical design error.

## Data Pipeline
The Archetype class is not part of a data processing pipeline; rather, it is a structural definition that **enables** the pipeline. It defines the memory layout for chunks of component data.

> **Entity Transformation Flow:**
>
> 1. An operation occurs (e.g., `entityManager.addComponent(entity, newComponent)`).
> 2. The `EntityManager` retrieves the entity's current `Archetype`.
> 3. A new `Archetype` is generated by calling `Archetype.add(currentArchetype, newComponentType)`.
> 4. The `EntityManager` moves the entity's component data from a memory chunk associated with the old Archetype to a chunk associated with the **new Archetype**.
> 5. The entity record is updated to point to its new Archetype and location.

> **System Query Flow:**
>
> 1. A `System` requests to process entities matching a `Query`.
> 2. The `QueryProcessor` translates the `Query` into a query `Archetype`.
> 3. The processor iterates through all known `Archetype`s in the world, performing a fast `contains(queryArchetype)` check.
> 4. For each matching `Archetype`, the processor gets the list of memory chunks containing the component data.
> 5. The `System` receives direct, linear access to the component arrays within those chunks for processing.

