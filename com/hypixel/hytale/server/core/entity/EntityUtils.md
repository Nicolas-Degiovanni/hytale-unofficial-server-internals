---
description: Architectural reference for EntityUtils
---

# EntityUtils

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Utility

## Definition
```java
// Signature
public class EntityUtils {
```

## Architecture & Concepts
EntityUtils is a stateless, static utility class that serves as a critical adapter between the low-level, performance-oriented Entity Component System (ECS) data structures and higher-level, object-oriented representations.

Its primary architectural function is to perform **entity materialization**. The Hytale server stores entity data in packed, contiguous memory blocks called ArchetypeChunks for cache efficiency. EntityUtils provides the canonical mechanism to take a reference to an entity within one of these chunks (represented by an index) and rehydrate it into a temporary, fully-formed Holder object. This Holder can then be more easily manipulated by game logic systems that are not designed to operate directly on raw component arrays.

The presence of numerous deprecated methods, such as getEntity and getPhysicsValues, signals a significant architectural evolution. These methods are remnants of a hybrid object-component model. The modern design, centered around the toHolder method, favors a purer ECS approach where an "entity" is simply an identity with a collection of components, rather than a concrete base class instance.

## Lifecycle & Ownership
- **Creation:** Not applicable. As a static utility class, EntityUtils is never instantiated. The Java ClassLoader loads it on first access.
- **Scope:** Application-level. The class and its static methods are available for the entire lifetime of the server process.
- **Destruction:** The class is unloaded when the server's Java Virtual Machine shuts down.

## Internal State & Concurrency
- **State:** **Stateless and Immutable.** EntityUtils contains no member fields and does not manage any internal state. Each method call operates exclusively on the arguments provided, producing a new output without side effects on the class itself.

- **Thread Safety:** **Thread-safe.** Due to its stateless nature, all methods in EntityUtils can be safely invoked from multiple threads simultaneously.
    - **WARNING:** While the class itself is thread-safe, the data structures it operates on, such as ArchetypeChunk and ComponentAccessor, are not guaranteed to be. The caller is solely responsible for ensuring that access to these underlying ECS structures is properly synchronized if they are being mutated by other threads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toHolder(index, archetypeChunk) | Holder | O(C) | **Primary API.** Materializes an entity from a raw ArchetypeChunk into a new Holder. Complexity is linear to C, the number of components in the entity's archetype. |
| getEntity(ref, accessor) | Entity | O(C) | **Deprecated.** Retrieves the legacy base Entity component. Avoid in new systems. |
| getPhysicsValues(ref, accessor) | PhysicsValues | O(1) | **Deprecated.** Provides fallback logic to find physics values, first from a direct component, then from a model definition. This logic should be moved into dedicated systems. |
| hasLivingEntity(archetype) | boolean | O(C) | **Deprecated.** Checks for the existence of a LivingEntity component in an archetype. |

## Integration Patterns

### Standard Usage
The sole intended use of this class in modern systems is to convert a low-level entity reference, often obtained from a spatial query or world iterator, into a temporary Holder object for processing.

```java
// System receives a raw entity reference from a query
ArchetypeChunk<EntityStore> targetChunk = ...;
int entityIndexInChunk = ...;

// Materialize the entity's components into a temporary object
Holder<EntityStore> entityHolder = EntityUtils.toHolder(entityIndexInChunk, targetChunk);

// Pass the holder to other systems or process its components
if (entityHolder.hasComponent(HealthComponent.class)) {
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Frequent Materialization:** Do not call toHolder repeatedly for the same entity within a tight loop. The process involves object allocation and data copying, which carries a significant performance cost. For high-performance systems, operate directly on the ArchetypeChunk if possible.
- **Use of Deprecated Methods:** Do not use getEntity, getPhysicsValues, or other deprecated methods in new code. They are maintained for backward compatibility and will be removed. Their existence encourages coupling to an obsolete object model.
- **Attempted Instantiation:** The class has no public constructor and provides only static methods. Do not attempt to create an instance of EntityUtils.

## Data Pipeline
EntityUtils functions as a transformation point in the data flow, converting data from a cache-optimized layout to an object-oriented layout.

> Flow:
> Low-Level System (e.g., Physics Engine) -> Raw Pointer (ArchetypeChunk + index) -> **EntityUtils.toHolder()** -> Materialized Object (Holder) -> High-Level System (e.g., AI, Gameplay Logic)

