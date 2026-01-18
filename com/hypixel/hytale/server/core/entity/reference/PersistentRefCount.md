---
description: Architectural reference for PersistentRefCount
---

# PersistentRefCount

**Package:** com.hypixel.hytale.server.core.entity.reference
**Type:** Component Data Model

## Definition
```java
// Signature
public class PersistentRefCount implements Component<EntityStore> {
```

## Architecture & Concepts
The PersistentRefCount is a fundamental component within the server-side Entity-Component-System (ECS) architecture. Its sole purpose is to attach a simple, serializable reference counter to an entity. This mechanism is critical for managing the lifecycle of persistent entities that might be referenced by multiple other systems or entities.

Conceptually, this component acts as a distributed garbage collection trigger. When an entity's PersistentRefCount reaches zero, it signals to the engine that no strong references remain, and the entity can be safely unloaded or destroyed. This prevents premature cleanup of entities that are still logically "in use" by other parts of the game world, such as a quest objective or an item spawner's template.

Its integration with the `EntityStore` via the `Component<EntityStore>` interface and the presence of a `CODEC` field explicitly mark it as a component whose state is intended to be saved to and loaded from the world database.

## Lifecycle & Ownership
- **Creation:** A PersistentRefCount instance is never created directly. It is added to an entity by the ECS framework, typically when an entity is first created and needs its lifecycle managed by reference counting. For example: `serverEntity.addComponent(PersistentRefCount.class)`.
- **Scope:** The lifecycle of a PersistentRefCount is strictly bound to the entity it is attached to. It exists only as long as its parent entity exists.
- **Destruction:** The component is destroyed automatically when its parent entity is removed from the world. The `EntityManager` or an equivalent system manages this cleanup process.

## Internal State & Concurrency
- **State:** The component's state is mutable, consisting of a single integer field, `refCount`. It holds no caches or complex data structures.
- **Thread Safety:** This component is **not thread-safe**. The internal `refCount` is not atomic, and the `increment` method is not synchronized. All reads and writes to this component must be performed on the main server thread to prevent race conditions and data corruption.

**WARNING:** Accessing this component from asynchronous tasks or worker threads without external locking will lead to unpredictable behavior and state desynchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | int | O(1) | Returns the current reference count. |
| increment() | void | O(1) | Increments the reference count. Wraps to 0 on overflow. |
| clone() | Component | O(1) | Creates a new instance with the same reference count. |
| getComponentType() | ComponentType | O(1) | Static method to retrieve the registered type for this component. |

## Integration Patterns

### Standard Usage
A system that creates or holds a reference to an entity should retrieve this component and increment its counter. When the reference is released, the counter should be decremented (though a `decrement` method is not present in this snippet, it is the logical counterpart).

```java
// Assume 'entity' is a valid ServerEntity
if (entity.hasComponent(PersistentRefCount.class)) {
    PersistentRefCount counter = entity.getComponent(PersistentRefCount.class);
    counter.increment();
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new PersistentRefCount()`. Components must be managed by the ECS framework to ensure they are correctly registered and attached to an entity.
- **Multi-threaded Modification:** Do not call `increment` from any thread other than the main server tick thread. This will corrupt the reference count.
- **Ignoring Overflow:** The `increment` method wraps from `Integer.MAX_VALUE` back to 0. Systems should not assume the counter will remain at max value; this behavior could lead to premature entity deletion if not handled.

## Data Pipeline
The PersistentRefCount's primary data flow occurs during world persistence operations (saving and loading).

> **Serialization (World Save):**
> Entity in World -> EntityStore Save Process -> **PersistentRefCount.refCount** -> CODEC Serialization -> World Database

> **Deserialization (World Load):**
> World Database -> CODEC Deserialization -> **PersistentRefCount** Instance -> EntityStore Load Process -> Entity in World

