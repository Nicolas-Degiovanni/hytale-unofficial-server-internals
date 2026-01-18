---
description: Architectural reference for SteeringSystem
---

# SteeringSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class SteeringSystem extends SteppableTickingSystem {
```

## Architecture & Concepts
The SteeringSystem is a core component of the server-side NPC motion pipeline, operating within the Entity Component System (ECS) framework. Its primary function is not to implement steering logic itself, but to act as a high-level orchestrator that delegates motion control to an NPC's currently active behavioral components.

This system queries for all entities possessing an NPCEntity component and, for each one, invokes the `steer` method on its active MotionController. This design decouples the generic process of "steering all NPCs" from the specific implementation of "how a single NPC should move," which is defined by its Role.

The execution order of this system is critical for stable physics and behavior:
1.  It runs **after** AvoidanceSystem, ensuring that steering calculations are based on collision-avoidance adjustments made in the same tick.
2.  It runs **after** KnockbackSystems, allowing steering to be influenced by or potentially override involuntary movements like knockback.
3.  It runs **before** TransformSystems.EntityTrackerUpdate, guaranteeing that any changes to an entity's position and rotation are finalized before they are broadcast to clients or other tracking systems.

This strategic placement makes it one of the final authorities on an NPC's intended movement within a single server tick.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's central ECS System Registry upon the loading of the NPCPlugin. It is not created dynamically during gameplay.
-   **Scope:** Session-scoped. The SteeringSystem instance persists for the entire lifetime of the server world.
-   **Destruction:** The instance is destroyed and garbage collected when the server world is shut down or the NPCPlugin is unloaded.

## Internal State & Concurrency
-   **State:** The SteeringSystem is **stateless**. Its fields are final and are initialized at construction. It does not cache any data between ticks or across entities. All state is read directly from entity components during the `steppedTick` execution.

-   **Thread Safety:** This system is explicitly **not thread-safe**. The `isParallel` method returns false, which instructs the ECS scheduler to execute its `steppedTick` method serially for all matching entities.

    **WARNING:** Attempting to invoke this system's methods from outside the main server thread will lead to concurrency violations, data corruption, and server instability. The ECS scheduler guarantees safe execution.

## API Surface
The primary API is the `steppedTick` method, which is intended for consumption by the ECS framework, not for direct developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| steppedTick(dt, index, chunk, store, commands) | void | O(N) | Framework-invoked method. Processes a single entity matching the query. Complexity is O(N) for the system, where N is the number of NPCs. The delegated call to `MotionController.steer` has its own unknown complexity. Throws and catches internal exceptions to prevent a single failing NPC from crashing the server tick. |

## Integration Patterns

### Standard Usage
Developers do not interact with the SteeringSystem directly. Instead, they influence its behavior by assigning and configuring the `Role` and `MotionController` components on an NPC entity. The system will automatically discover and process these components.

```java
// Conceptual example of configuring an NPC that the SteeringSystem will process.
// This code would exist within a spawner or behavior-tree node.

NPCEntity npc = entity.get(NPCEntity.class);
Role newRole = new GuardRole(); // GuardRole provides a specific MotionController

// By setting the role, the SteeringSystem will now delegate to the
// GuardRole's MotionController during the next tick.
npc.setRole(newRole);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new SteeringSystem()`. The ECS framework is solely responsible for its lifecycle. Direct instantiation will result in a non-functional object that is not registered with the game loop.
-   **Manual Invocation:** Do not call `steppedTick` manually. The ECS scheduler provides the correct state, context, and timing. Manual invocation will bypass dependency ordering and cause severe desynchronization and physics errors.
-   **Stateful Motion Controllers:** The `MotionController` implementations that this system delegates to should be designed to be stateless or have their state reset each tick. Storing persistent state across ticks within a motion controller can lead to unpredictable behavior, especially when an NPC's `Role` changes.

## Data Pipeline
The SteeringSystem acts as a processing node in the per-tick NPC update pipeline. It reads component data, delegates logic, and facilitates the mutation of transform data.

> Flow:
> ECS Scheduler -> Query for NPCEntity -> **SteeringSystem.steppedTick** -> Read NPCEntity and TransformComponent -> Delegate to `Role.getActiveMotionController().steer()` -> Mutate TransformComponent & Write to CommandBuffer -> TransformSystems.EntityTrackerUpdate

