---
description: Architectural reference for MovementStatesSystem
---

# MovementStatesSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class MovementStatesSystem extends SteppableTickingSystem {
```

## Architecture & Concepts
The MovementStatesSystem is a core component of the server-side Entity Component System (ECS) responsible for NPC behavior. Its primary function is to translate the quantitative physical motion of an NPC into a qualitative, descriptive state. It acts as a bridge between the physics simulation and higher-level AI or animation systems.

This system operates on all entities possessing an NPCEntity, a Velocity, and a MovementStatesComponent. For each qualifying entity, it reads the current Velocity component—which represents the entity's physical movement in the world—and delegates the logic for updating the MovementStatesComponent to the NPC's assigned Role object. This decouples the generic system from specific NPC type logic, allowing different roles (e.g., a passive villager versus an aggressive monster) to interpret the same velocity data in unique ways.

A critical architectural constraint is its position in the execution order. The system explicitly declares that it must run *after* the ComputeVelocitySystem. This guarantees that it always processes the final, authoritative velocity for the current game tick, preventing state inconsistencies and one-frame delays in NPC reactions.

## Lifecycle & Ownership
- **Creation:** Instantiated once per world instance by the server's central ECS scheduler or System Registry during the server bootstrap sequence. Component type dependencies are provided via constructor injection at this time.
- **Scope:** Session-scoped. A single instance persists for the entire lifetime of a game world.
- **Destruction:** The instance is marked for garbage collection when the corresponding game world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This system is stateless. Its internal fields are immutable references to component types and dependency configurations, established at construction. It does not cache any entity data between ticks. All state modifications are performed directly on the components of the entities it processes within the scope of the steppedTick method.
- **Thread Safety:** **Not thread-safe.** The system explicitly signals to the ECS scheduler that it cannot be run in parallel by returning false from the isParallel method. This is a deliberate design choice to ensure deterministic behavior and prevent race conditions, as the delegated Role logic may not be designed for concurrent execution, and modifications via the CommandBuffer often require serial processing.

## API Surface
The public contract is defined by its implementation of the SteppableTickingSystem interface. Direct invocation is not intended; the ECS scheduler is the sole caller.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| steppedTick(dt, index, chunk, store, commands) | void | O(N) | The primary entry point, executed by the ECS scheduler for each entity matching the query. N is the number of entities. Delegates state update logic to the entity's Role. |
| getDependencies() | Set | O(1) | Returns the execution order dependencies, primarily ensuring it runs after ComputeVelocitySystem. |
| getQuery() | Query | O(1) | Returns the ECS query used to select entities for processing. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. Its operation is automatic. To have an NPC's movement state managed by this system, a developer must ensure the entity is spawned with the required components:
1.  NPCEntity
2.  Velocity
3.  MovementStatesComponent

The system will then automatically discover and process the entity on each server tick.

```java
// Example: An entity factory ensuring an entity is processed by this system
// This code would exist within a spawner or entity creation utility.

Entity entity = world.createEntity();
CommandBuffer commands = world.getCommandBuffer();

// Adding the required components makes the entity eligible for processing
commands.addComponent(entity, new NPCEntity(...));
commands.addComponent(entity, new Velocity());
commands.addComponent(entity, new MovementStatesComponent());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new MovementStatesSystem()`. The system must be instantiated and managed by the server's ECS framework to guarantee correct dependency injection and inclusion in the tick loop.
- **Manual Invocation:** Calling the steppedTick method manually will bypass the ECS scheduler. This breaks the critical execution order guarantee, likely causing the system to process stale velocity data from the previous tick and leading to desynchronized or incorrect NPC behavior.
- **Modifying Velocity After Execution:** Any system that modifies the Velocity component and runs *after* MovementStatesSystem will cause a one-frame lag in the NPC's movement state update, as the change will not be reflected until the next tick.

## Data Pipeline
The system is a key stage in the NPC motion-to-behavior data pipeline. It consumes physical data and produces logical state data that drives subsequent systems.

> Flow:
> Physics Update -> ComputeVelocitySystem -> **Velocity Component** -> **MovementStatesSystem** -> NPCEntity.getRole().updateMovementState() -> **MovementStatesComponent** -> AI Behavior System / Animation System

