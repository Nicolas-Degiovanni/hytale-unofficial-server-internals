---
description: Architectural reference for EntityRegistration
---

# EntityRegistration

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** Transient

## Definition
```java
// Signature
public class EntityRegistration extends Registration {
```

## Architecture & Concepts
The EntityRegistration class is a descriptor object, not a service or manager. Its fundamental role is to act as a manifest for a single entity type within the server's core entity system. It encapsulates the link between a concrete Java class (e.g., Zombie, Player, Arrow) and its lifecycle management hooks within the registration framework.

This class is a key component of the server's modularity. Instead of a central, hardcoded list of all possible entities, modules can dynamically declare the entities they provide by creating and submitting an EntityRegistration instance. The server's central entity registry consumes these objects to understand what entities are available.

The design heavily utilizes a callback pattern through the inherited `BooleanSupplier` and `Runnable` fields from its parent, the Registration class. This decouples the registration data (the entity class) from the state management logic (checking if it is enabled, or how to unregister it). This allows a module to retain full control over its own entity's lifecycle rules while exposing a standardized contract to the core system.

## Lifecycle & Ownership
- **Creation:** An EntityRegistration instance is created by a module or system responsible for defining a new entity type. This typically occurs during the server's bootstrap or module-loading phase. For example, a "Hostile Mobs" module would instantiate an `EntityRegistration` for its `Zombie` class.

- **Scope:** The object's lifetime is tied to the registration of the entity type it represents. It persists as long as the entity is known to the server's entity registry. If the owning module is unloaded, the registration is typically destroyed with it.

- **Destruction:** The object is eligible for garbage collection after it has been successfully unregistered from the central registry. The `unregister` `Runnable` passed during construction is invoked to perform any necessary cleanup before the object is discarded.

## Internal State & Concurrency
- **State:** The internal state of an EntityRegistration object is **Immutable**. The `entityClass` field is final and assigned only at construction. The lifecycle callbacks inherited from the parent `Registration` class are also set at construction and cannot be changed. While the object itself is immutable, the state it represents (i.e., whether the entity is enabled) is dynamic and is determined by executing the external `isEnabled` `BooleanSupplier`.

- **Thread Safety:** The object is inherently thread-safe for read operations due to its immutability. Callers can safely access `getEntityClass` from any thread.

    **WARNING:** The thread safety of the provided `BooleanSupplier` and `Runnable` callbacks is the responsibility of the creator. The EntityRegistration class provides no internal synchronization for these external delegates. Callbacks that modify shared state must be implemented with appropriate concurrency controls.

## API Surface
The primary contract is defined by its constructors and the methods inherited from the `Registration` parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEntityClass() | Class<? extends Entity> | O(1) | Returns the concrete Java class for the entity this registration represents. |
| isEnabled() | boolean | Varies | (Inherited) Invokes the `BooleanSupplier` to determine if the entity type is active. Complexity is dependent on the supplied logic. |
| unregister() | void | Varies | (Inherited) Invokes the `Runnable` to execute the unregistration logic for this entity type. |

## Integration Patterns

### Standard Usage
An EntityRegistration should be created by a module and passed to a central registry service. The module provides the logic for enabling and unregistering its own entity.

```java
// In a module's initialization method
EntityModuleRegistry registry = context.getService(EntityModuleRegistry.class);

// The module defines its own lifecycle logic
BooleanSupplier enabledCheck = () -> this.moduleConfig.isZombieEnabled();
Runnable unregisterLogic = () -> this.cleanupZombieAssets();

EntityRegistration zombieRegistration = new EntityRegistration(
    Zombie.class,
    enabledCheck,
    unregisterLogic
);

registry.registerEntity(zombieRegistration);
```

### Anti-Patterns (Do NOT do this)
- **Complex Callback Logic:** The `isEnabled` supplier may be called frequently by the core engine. Avoid placing slow, blocking, or computationally expensive operations (like file I/O or network calls) within it.

- **Orphaned Instances:** Creating an `EntityRegistration` instance without submitting it to a registry is pointless. The object has no behavior on its own and will simply be garbage collected.

- **Stateful Callbacks:** Avoid implementing complex state machines within the `isEnabled` or `unregister` callbacks. This logic belongs in the module that owns the entity, with the callbacks acting as simple delegates.

## Data Pipeline
EntityRegistration is not part of a real-time data processing pipeline. Instead, it is a configuration artifact used during the server's initialization and control plane operations.

> **Flow:**
> Server Bootstrap → Module Initialization → **Module creates EntityRegistration instance** → Instance submitted to Entity Registry → Registry populates internal maps → Entity spawning systems query Registry to validate entity types.

