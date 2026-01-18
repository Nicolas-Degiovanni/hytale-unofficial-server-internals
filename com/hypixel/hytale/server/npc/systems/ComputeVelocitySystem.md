---
description: Architectural reference for ComputeVelocitySystem
---

# ComputeVelocitySystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class ComputeVelocitySystem extends SteppableTickingSystem {
```

## Architecture & Concepts
The ComputeVelocitySystem is a data-processing node within the server-side Entity Component System (ECS) framework. Its sole responsibility is to derive an entity's velocity by observing the change in its position between game ticks.

This system acts as a translator, converting positional data into kinetic data. It subscribes to all entities possessing an **NPCEntity**, **TransformComponent**, and **Velocity** component, as defined by its internal **Query**. During each step of the server's simulation loop, the system calculates the displacement of each matched entity since the last tick and divides by the delta time (*dt*) to produce a velocity vector. This result is then written back into the entity's **Velocity** component.

By isolating this calculation, the engine decouples the cause of movement (e.g., AI, player input, physics) from the representation of that movement. Downstream systems, such as collision detection or network replication, can then consume the standardized **Velocity** component without needing to know how an entity's position was updated.

The class extends **SteppableTickingSystem**, indicating it is designed to be executed by a scheduler at a fixed rate, ensuring deterministic physics calculations.

## Lifecycle & Ownership
-   **Creation:** This system is not instantiated directly by developers. It is constructed and managed by a higher-level system scheduler or world builder during the server's initialization phase. Its dependencies, such as required **ComponentType** handles, are injected via its constructor, a pattern typical of dependency-injected frameworks.
-   **Scope:** An instance of ComputeVelocitySystem persists for the entire lifetime of the server world it is registered with. It is part of the core simulation machinery.
-   **Destruction:** The system is destroyed and garbage collected when the server world is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** The ComputeVelocitySystem instance itself is effectively stateless. Its member fields are final references to component types and queries, which are configuration set at construction. It does not store or cache any per-entity data between invocations. All state it operates on is passed into the **steppedTick** method via the **ArchetypeChunk**.
-   **Thread Safety:** This system is designed for parallel execution and is considered thread-safe. The **isParallel** method provides a hint to the ECS scheduler, which may choose to execute the **steppedTick** method for different chunks of entities across multiple threads. Safety is achieved because each invocation of **steppedTick** operates on a single, distinct entity within a chunk. The system contains no shared mutable state, preventing data races between threads.

**WARNING:** Adding mutable member variables to this class will break thread safety and lead to severe, unpredictable concurrency bugs when the scheduler runs it in parallel.

## API Surface
The primary contract is defined by its parent class, **SteppableTickingSystem**. Direct invocation is not intended.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| steppedTick(dt, index, chunk, store, buffer) | void | O(1) | Executes the velocity calculation for a single entity. Called exclusively by the ECS scheduler. |
| getQuery() | Query | O(1) | Returns the entity query that determines which entities this system will process. |
| getDependencies() | Set | O(1) | Declares component dependencies to the scheduler for ordering system execution. |
| isParallel(chunkSize, taskCount) | boolean | O(1) | Informs the scheduler if this system's workload can be parallelized. |

## Integration Patterns

### Standard Usage
A developer does not call methods on this system directly. Instead, it is registered with the world's system scheduler during server setup. The framework handles its lifecycle and execution.

```java
// Example: Registering the system during world initialization
SystemScheduler scheduler = world.getSystemScheduler();

// The framework constructs and registers the system.
// Component types would be retrieved from a central registry.
scheduler.addSystem(new ComputeVelocitySystem(
    NPCEntity.getComponentType(),
    Velocity.getComponentType(),
    /* dependencies */
));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Invocation:** Never manually create an instance and call **steppedTick**. This bypasses the ECS scheduler, breaking dependency ordering, concurrency guarantees, and state management.
-   **Stateful Implementation:** Do not add mutable member fields to this class to track state across ticks or entities. This will fail catastrophically in a parallel execution environment. All state must be stored in components.

## Data Pipeline
This system is a transformation stage in the entity update pipeline. It consumes positional data and produces velocity data.

> Flow:
> AI or Physics System updates **TransformComponent** -> **ComputeVelocitySystem** reads new **TransformComponent.position** and old **NPCEntity.oldPosition** -> **ComputeVelocitySystem** writes to **Velocity** component -> Downstream Physics or Replication System reads **Velocity** component

