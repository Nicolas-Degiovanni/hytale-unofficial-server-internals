---
description: Architectural reference for ReadWriteQuery
---

# ReadWriteQuery

**Package:** com.hypixel.hytale.component
**Type:** Transient Value Object

## Definition
```java
// Signature
public class ReadWriteQuery<ECS_TYPE> implements ReadWriteArchetypeQuery<ECS_TYPE> {
```

## Architecture & Concepts
The ReadWriteQuery class is a fundamental data structure within the Entity Component System (ECS) framework. It does not represent a service or a manager, but rather a **data access contract** used by Systems to declare their component requirements to the ECS scheduler.

Its primary role is to encapsulate the result of an ECS query that partitions entities into two distinct groups:
1.  **Read Archetype:** A collection of entities whose components will only be read during a System's execution.
2.  **Write Archetype:** A collection of entities whose components may be modified.

This explicit declaration of read and write intent is critical for the engine's performance and stability. It allows the ECS scheduler to build a dependency graph of all active Systems, enabling it to safely parallelize System updates without introducing data races or read-after-write hazards. A ReadWriteQuery is effectively a formal request for specific, segregated views of the ECS world state.

## Lifecycle & Ownership
-   **Creation:** A ReadWriteQuery is instantiated by a System, typically during its initialization phase, to define its data dependencies. It is not managed by a central registry or service locator.
-   **Scope:** The object's lifetime is tied to the System that created it. It is a lightweight descriptor that persists as long as the owning System is registered and active.
-   **Destruction:** The object is eligible for garbage collection once the owning System is destroyed and all references to the query are released. It has no explicit cleanup or disposal method.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal references to the read and write Archetype objects are marked as final and are assigned exclusively at construction time. The state of a ReadWriteQuery instance cannot be altered after it is created.
-   **Thread Safety:** **Inherently Thread-Safe**. Due to its immutable nature, an instance of ReadWriteQuery can be safely passed between and read by multiple threads without any external synchronization. The thread safety of operations on the underlying Archetype data is a separate concern managed by the ECS scheduler, which uses the query's contract to enforce safe access patterns.

## API Surface
The public API is minimal, serving only to expose the two archetypes defined at construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getReadArchetype() | Archetype | O(1) | Retrieves the archetype containing components designated for read-only access. |
| getWriteArchetype() | Archetype | O(1) | Retrieves the archetype containing components designated for read-write access. |

## Integration Patterns

### Standard Usage
A System constructs a ReadWriteQuery to declare its data dependencies. The ECS scheduler then uses this query object to provide the System with safe, appropriately permissioned access to component data during the update tick.

```java
// Within a System's setup or initialization method
Archetype readOnlyEntities = world.query(Position.class, Velocity.class);
Archetype mutableEntities = world.query(Position.class, PhysicsBody.class);

// The query is created once and stored by the System
this.physicsQuery = new ReadWriteQuery<>(readOnlyEntities, mutableEntities);

// The scheduler later uses this query to execute the system
scheduler.run(system, system.physicsQuery);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Queries:** Do not attempt to modify a ReadWriteQuery after creation. Its immutability is essential for the scheduler to make reliable and predictable execution guarantees.
-   **Direct Iteration:** Avoid using the Archetypes returned by this query for direct iteration within game logic. This object's purpose is to communicate intent to the scheduler. The scheduler is responsible for providing the final, safe-to-iterate data views to the System's update method. Bypassing the scheduler can lead to severe concurrency bugs.

## Data Pipeline
The ReadWriteQuery acts as a static descriptor in the data processing pipeline, defining the shape of the data a System will receive.

> Flow:
> System Initialization -> **ReadWriteQuery** (Instantiation as a data contract) -> ECS Scheduler (Builds dependency graph) -> System Execution (Receives safe Archetype views based on the original query) -> Component Data Modification

