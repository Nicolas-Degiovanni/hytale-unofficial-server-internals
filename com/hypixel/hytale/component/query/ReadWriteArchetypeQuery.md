---
description: Architectural reference for ReadWriteArchetypeQuery
---

# ReadWriteArchetypeQuery

**Package:** com.hypixel.hytale.component.query
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface ReadWriteArchetypeQuery<ECS_TYPE> extends Query<ECS_TYPE> {
```

## Architecture & Concepts
The ReadWriteArchetypeQuery interface is a foundational contract within Hytale's Entity Component System (ECS). It serves as a high-performance filter, defining the precise data access requirements for a given System. Its primary architectural role is to declare which components a System will read from and which it will write to.

This explicit separation of read and write dependencies is the cornerstone of the ECS scheduler's ability to parallelize work. By analyzing the ReadWriteArchetypeQuery contracts from multiple Systems, the engine can build a dependency graph, identifying which Systems can be executed concurrently on different threads without causing data races, and which must be executed serially.

This interface does not hold entity data itself; it is a metadata-driven specification used to select entities that match a specific component layout, or *archetype*.

## Lifecycle & Ownership
As an interface, ReadWriteArchetypeQuery does not have a concrete lifecycle. Its lifecycle is defined by its implementing classes.

- **Creation:** Implementations are typically defined as part of an ECS System's initialization. They are often created once and cached for the lifetime of the System, representing a static declaration of data requirements.
- **Scope:** The query's scope is bound to the System that defines it. It persists as long as the System is registered and active within the ECS World.
- **Destruction:** The query object is eligible for garbage collection when its owning System is destroyed or unregistered from the ECS World.

**WARNING:** The Archetypes returned by this query are definitions. Modifying them at runtime after System registration will lead to undefined behavior and likely destabilize the ECS scheduler.

## Internal State & Concurrency
- **State:** This interface is stateless. It is a pure contract whose default methods operate on the provided arguments or the state of the Archetypes returned by the implementer. The implementing class is expected to hold immutable references to a read Archetype and a write Archetype.
- **Thread Safety:** The interface itself is inherently thread-safe. Its methods are designed to be called by the ECS scheduler, which may operate across multiple threads. The contract's design is what *enables* thread safety at a higher level by providing the scheduler with the necessary information to prevent conflicting data access between Systems.

## API Surface
The public contract is focused on retrieving the component requirements and validating them.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getReadArchetype() | Archetype | O(1) | Returns the Archetype defining all components the System requires read-only access to. |
| getWriteArchetype() | Archetype | O(1) | Returns the Archetype defining all components the System requires write access to. |
| test(archetype) | boolean | O(N+M) | **Default Method.** Returns true if the provided entity Archetype contains all components from both the read and write archetypes. |
| requiresComponentType(type) | boolean | O(N+M) | **Default Method.** Returns true if the specified component type is present in either the read or write archetype. |
| validateRegistry(registry) | void | O(N+M) | **Default Method.** Validates that all components in both archetypes are registered in the master ComponentRegistry. Throws an exception on failure. |
| validate() | void | O(N*M) | **Default Method.** Validates that the read and write archetypes do not contain overlapping component types, which is a critical invariant. |

*Complexity Note: N and M refer to the number of component types in the read and write archetypes, respectively.*

## Integration Patterns

### Standard Usage
A System declares its data dependencies by implementing this interface. The ECS World then uses this query to efficiently iterate over only the relevant entities during the System's update tick.

```java
// A System defines its query to process entities with Position and Velocity,
// while also writing to a PhysicsState component.
public class PhysicsSystem implements System, ReadWriteArchetypeQuery<WorldEntity> {

    private static final Archetype<WorldEntity> READ_ARCHETYPE = Archetype.of(
        Position.class,
        Velocity.class
    );

    private static final Archetype<WorldEntity> WRITE_ARCHETYPE = Archetype.of(
        PhysicsState.class
    );

    @Override
    public Archetype<WorldEntity> getReadArchetype() {
        return READ_ARCHETYPE;
    }

    @Override
    public Archetype<WorldEntity> getWriteArchetype() {
        return WRITE_ARCHETYPE;
    }

    // The ECS scheduler uses the query to provide the system with matching entities.
    public void update(EntityView view) {
        // ... process entities ...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Component Overlap:** Defining the same component type in both the read and write archetypes is a contract violation. The default `validate` method is designed to catch this error during engine startup. This indicates a logical flaw in the System's design.
- **Contract Violation:** Requesting a component for read-only access via `getReadArchetype` but then attempting to modify its data within the System. This subverts the scheduler's safety guarantees and can introduce severe, difficult-to-debug race conditions in a multithreaded environment.
- **Dynamic Queries:** Avoid creating new query instances per-frame. Queries should be static definitions that are created once and reused, allowing the engine to perform significant upfront optimization.

## Data Pipeline
ReadWriteArchetypeQuery acts as a filter or a gate within the ECS data processing pipeline. It does not transform data but rather selects which data flows to a given System.

> Flow:
> ECS World Entity Pool -> **ReadWriteArchetypeQuery.test(entityArchetype)** -> [MATCH] -> System Update Tick -> System processes entity components

