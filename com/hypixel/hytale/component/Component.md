---
description: Architectural reference for the Component interface, the core data contract in Hytale's ECS framework.
---

# Component

**Package:** com.hypixel.hytale.component
**Type:** Core Interface

## Definition
```java
// Signature
public interface Component<ECS_TYPE> extends Cloneable {
```

## Architecture & Concepts
The Component interface is the fundamental data contract for Hytale's Entity-Component-System (ECS) architecture. It serves as the "C" in ECS, representing a single, atomic piece of data or state that can be attached to an Entity.

Unlike traditional object-oriented models where objects contain both data and behavior, a Component is designed to be a pure data container. All logic and behavior that operate on this data are encapsulated within Systems. This separation of data from logic is a core tenet of ECS, promoting high performance, data locality, and ease of parallelization.

An Entity is little more than an identifier; its identity and behavior are defined by the collection of Components attached to it. For example, an Entity with a PositionComponent and a VelocityComponent is a movable object, while an Entity with a HealthComponent and an ArmorComponent is a damageable character.

The generic parameter, ECS_TYPE, suggests a mechanism for associating components with a specific ECS implementation or context, potentially for compile-time type safety or framework-specific optimizations.

The static EMPTY_ARRAY field is a performance optimization, providing a shared, immutable instance to avoid repeated memory allocation for empty component arrays during entity queries.

## Lifecycle & Ownership
As an interface, Component itself does not have a lifecycle. The following describes the lifecycle of a concrete *implementation* of this interface.

-   **Creation:** Component instances are typically created and attached to an Entity by a factory, a spawner, or an EntityManager. They are rarely instantiated directly in game logic code. The lifecycle of a Component is strictly bound to the Entity it is attached to.
-   **Scope:** A Component instance exists as long as it remains attached to a living Entity. Its scope is identical to its parent Entity's scope within the game world.
-   **Destruction:** A Component is destroyed under two conditions:
    1.  When its parent Entity is destroyed.
    2.  When the Component is explicitly removed from the Entity by a System.
    Garbage collection reclaims the memory once all references are cleared.

## Internal State & Concurrency
-   **State:** Implementations of Component are expected to be mutable Plain Old Data Objects (PODs). Their primary purpose is to hold the state that Systems will read from and write to during the game loop.
-   **Thread Safety:** Component implementations are **not thread-safe** by design. The ECS architecture mandates that synchronization is managed at a higher level, typically by the scheduler that executes the Systems. Systems are scheduled in a way that prevents concurrent writes to the same Component types. Direct, unsynchronized access to a Component from multiple threads will lead to race conditions and data corruption.

**WARNING:** Never store a reference to a Component instance. Always re-query the Component from its parent Entity at the start of a processing tick, as the Component or the Entity itself may have been removed by a preceding System.

## API Surface
The public contract is minimal, focusing on the requirement for components to be duplicable.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| clone() | Component | O(N) | Creates a shallow or deep copy of the component. The complexity depends on the implementation. This is fundamental for entity duplication and state management. |
| cloneSerializable() | Component | O(N) | Creates a copy of the component suitable for serialization to disk or network. By default, this delegates to clone(), but can be overridden to strip transient or non-serializable fields. |

## Integration Patterns

### Standard Usage
A developer's primary interaction is to implement this interface to define new types of data. Systems then query for entities possessing these component types to perform game logic.

```java
// 1. Define a new component type
public class PositionComponent implements Component<Void> {
    public float x, y, z;

    @Override
    public PositionComponent clone() {
        // Implementation...
    }
}

// 2. A System queries for and processes entities with this component
public class MovementSystem extends System {
    public void update(World world) {
        for (Entity entity : world.query(PositionComponent.class, VelocityComponent.class)) {
            PositionComponent pos = entity.getComponent(PositionComponent.class);
            VelocityComponent vel = entity.getComponent(VelocityComponent.class);
            pos.x += vel.dx;
            // ...
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Embedding Logic:** Do not add methods with business logic to a Component implementation. All behavior belongs in a System. A Component should only contain data and, at most, simple helper methods for accessing that data.
-   **Storing Entity References:** Avoid storing direct references to other Entities or complex engine objects within a Component. This creates tight coupling, complicates serialization and cloning, and can lead to dangling references if the referenced entity is destroyed. Use entity IDs instead.
-   **Implementing Complex Interfaces:** Components should not implement complex behavioral interfaces. Their contract is to hold data and be cloneable.

## Data Pipeline
The Component is a passive data packet within the main game loop pipeline. It does not actively process data; it is the data that gets processed.

> Flow:
> System Scheduler -> Executes a System -> System queries for Entities with a **Component** set -> System reads/writes **Component** data -> Game state is altered

