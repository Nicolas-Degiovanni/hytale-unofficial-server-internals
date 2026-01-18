---
description: Architectural reference for GenericVelocityInstructionSystem
---

# GenericVelocityInstructionSystem

**Package:** com.hypixel.hytale.server.core.modules.physics.systems
**Type:** System Component

## Definition
```java
// Signature
public class GenericVelocityInstructionSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The GenericVelocityInstructionSystem is a core component of the server-side physics engine, operating within the Entity Component System (ECS) framework. Its primary architectural role is to act as a centralized processor for deferred velocity modifications.

This system embodies the **Command Pattern**. Instead of various game logic systems (e.g., AI, player input, status effects) directly manipulating an entity's velocity, they enqueue their intent as an **Instruction** into the entity's Velocity component. This system then consumes that queue at a specific, deterministic point in the game tick.

This design decouples the *intent* to change velocity from the *application* of that change. This provides two key benefits:
1.  **Determinism:** By processing all velocity commands in a single, ordered pass, it eliminates race conditions and unpredictable behavior that could arise from multiple systems modifying the same data.
2.  **Order of Operations:** The system's dependencies ensure it runs *after* all other systems that might generate velocity instructions. This guarantees that all commands for a given tick are collected before being processed.

It operates on a query that targets all entities possessing a Velocity component, making it a foundational system for any moving object in the world.

### Lifecycle & Ownership
-   **Creation:** Instantiated automatically by the server's ECS World or Scheduler during the initialization of the Physics module. It is discovered and registered as part of the system graph.
-   **Scope:** The system's instance persists for the entire lifetime of the server's ECS World. It is not created or destroyed on a per-tick basis.
-   **Destruction:** The instance is discarded and eligible for garbage collection only when the server shuts down and the parent ECS World is destroyed.

## Internal State & Concurrency
-   **State:** This class is effectively **stateless**. Its member fields, `dependencies` and `query`, are immutable and defined at compile-time or construction. All state it manipulates is external, residing within the Velocity components of the entities it processes.
-   **Thread Safety:** This system is not designed to be thread-safe in isolation. The `tick` method is expected to be invoked by the ECS scheduler, which guarantees that it will not be called concurrently for the same set of components. The entire architecture of queuing instructions is a concurrency control pattern, ensuring that writes to the core velocity data are serialized through this single system.

## API Surface
The public API is exclusively for consumption by the ECS scheduler. Game logic developers should not interact with these methods directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(I) | Processes all pending instructions for a single entity. I is the number of instructions. Called by the scheduler for each entity matching the query. |
| getDependencies() | Set | O(1) | Returns the scheduling dependencies, ensuring this system runs after velocity-modifying systems. |
| getQuery() | Query | O(1) | Returns the component query used by the scheduler to identify target entities. |

## Integration Patterns

### Standard Usage
A developer **does not** call this system directly. The correct pattern is for other systems to add instructions to an entity's Velocity component. The engine will then automatically invoke this system to process them.

```java
// Example from a hypothetical "JumpSystem"
// This system adds a command; it does not call GenericVelocityInstructionSystem.

Velocity velocity = archetypeChunk.getComponent(index, Velocity.getComponentType());
Vector3f jumpForce = new Vector3f(0, 10, 0);

// Queue an instruction to be processed later in the tick
velocity.getInstructions().add(Velocity.Instruction.add(jumpForce));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new GenericVelocityInstructionSystem()`. The ECS framework is responsible for its creation and lifecycle management.
-   **Manual Invocation:** Never call the `tick` method directly. Doing so bypasses the scheduler, violates dependency ordering, and will lead to unpredictable physics simulation and state corruption.
-   **Direct Velocity Mutation:** While technically possible, other systems should avoid calling `velocity.set()` or `velocity.addForce()` directly. This breaks the command pattern and re-introduces the risk of race conditions that this system is designed to prevent. All velocity changes should be submitted as instructions.

## Data Pipeline
The system acts as a processing stage in the entity data flow for a single game tick. It consumes a list of commands and produces a single, consolidated state change.

> Flow:
> Game Logic System (e.g., AI) -> Creates `Velocity.Instruction` -> Adds Instruction to `Velocity` component list -> **GenericVelocityInstructionSystem** reads list -> Modifies primary vector in `Velocity` component -> Clears instruction list -> Physics Integration System reads final velocity

