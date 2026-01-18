---
description: Architectural reference for NonSerialized
---

# NonSerialized

**Package:** com.hypixel.hytale.component
**Type:** Singleton

## Definition
```java
// Signature
public class NonSerialized<ECS_TYPE> implements Component<ECS_TYPE> {
```

## Architecture & Concepts
The NonSerialized component is a specialized, stateless *marker component* within the Entity-Component-System (ECS) framework. Its sole purpose is to signal to other systems, primarily the serialization and persistence layer, that an entity to which it is attached should be ignored.

This component does not hold any data. Its presence on an entity is a boolean flag indicating that the entity's state is transient and should not be saved to disk or transmitted over the network during world state synchronization. This is a critical optimization for performance and data integrity, preventing temporary or client-side-only entities (e.g., particle effects, UI state trackers, temporary physics objects) from polluting persistent storage.

The implementation employs the Singleton pattern to ensure that only one instance of this marker ever exists in memory, regardless of how many entities are marked as non-serializable. This is highly efficient, as it avoids the overhead of allocating and managing numerous identical, stateless objects.

## Lifecycle & Ownership
- **Creation:** The single static instance is created and initialized by the Java Virtual Machine during class loading. It is not instantiated by any game system or dependency injection container.
- **Scope:** As a static singleton, the NonSerialized instance persists for the entire lifetime of the application. Its lifecycle is tied directly to the JVM process.
- **Destruction:** The instance is eligible for garbage collection only when the application terminates and its ClassLoader is unloaded.

## Internal State & Concurrency
- **State:** The NonSerialized component is **immutable and stateless**. It contains no instance fields and its behavior never changes.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its stateless and immutable nature, it can be safely accessed, retrieved via the get method, and attached to entities from any thread without requiring locks or other synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | NonSerialized | O(1) | Statically retrieves the global singleton instance. This is the only valid way to obtain a reference. |
| clone() | Component | O(1) | Returns the singleton instance itself, rather than creating a new object. This reinforces the singleton pattern. |

## Integration Patterns

### Standard Usage
The component is intended to be added to an entity to prevent it from being saved. Systems that manage entity persistence will query for the existence of this component before processing an entity.

```java
// Example: Creating a transient visual effect entity
Entity vfxEntity = world.createEntity();

// Mark the entity to be excluded from world saves
vfxEntity.addComponent(NonSerialized.get());
```

### Anti-Patterns (Do NOT do this)
- **Data Association:** Do not attempt to extend this class to add data. Its design contract is to be a stateless marker. Systems are built to check for its *presence*, not its *state*.
- **Direct Instantiation:** The constructor is private (implicitly), so direct instantiation with *new* is impossible and will result in a compile-time error. Always use the static get method.
- **Type Parameter Misconception:** The generic parameter ECS_TYPE is a requirement to conform to the Component interface. It has no functional impact on the NonSerialized component itself, which is type-agnostic. Do not rely on this type parameter for any logic.

## Data Pipeline
The NonSerialized component does not process data itself; instead, it acts as a control gate within a larger data pipeline, such as world serialization.

> Flow:
> World Save Trigger -> Entity Iterator -> **Component Check: has(NonSerialized.class)?** -> [If True] Skip Entity / [If False] -> Serialize Entity -> Write to Disk

