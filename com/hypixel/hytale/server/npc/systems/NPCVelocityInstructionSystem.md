---
description: Architectural reference for NPCVelocityInstructionSystem
---

# NPCVelocityInstructionSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class NPCVelocityInstructionSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts

The NPCVelocityInstructionSystem is a specialized processor within the server-side Entity Component System (ECS) framework. Its primary function is to intercept and delegate physics instructions intended for Non-Player Characters (NPCs). It acts as a crucial bridge between generic physics commands and the nuanced behavioral logic of an NPC's assigned **Role**.

This system operates on any entity possessing both an NPCEntity component and a Velocity component. Instead of directly applying velocity changes, it translates each Velocity.Instruction into a high-level command that is passed to the NPC's current Role object (e.g., a GuardRole, VillagerRole). This Role then determines the appropriate behavioral response, such as playing a knockback animation, changing its AI state, or ignoring the instruction entirely if stunned.

This delegation pattern decouples the cause of a physical force from its effect on an NPC's behavior. It allows for complex, state-dependent reactions to physics without cluttering the core physics engine.

The system's execution order is critical for engine stability. It is configured to run **before** the GenericVelocityInstructionSystem. This ensures that NPC-specific instructions are consumed and handled by this system, preventing the generic system from applying a redundant, simplified physical change.

## Lifecycle & Ownership

-   **Creation:** Instantiated by the server's central ECS SystemManager during world initialization. The engine discovers and registers this system automatically; it is not created manually.
-   **Scope:** A single instance of this system exists for the entire duration of a world simulation. It is a global processor that operates on all entities matching its query.
-   **Destruction:** The instance is destroyed and garbage collected when the server world is unloaded or during server shutdown.

## Internal State & Concurrency

-   **State:** This system is **stateless**. Its internal fields, such as the dependency set and entity query, are immutable after construction. All operations are performed exclusively on the component data passed into the tick method for the current frame. It holds no entity state between ticks.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless design. The Hytale ECS scheduler guarantees that the tick method is executed in a thread-safe context for the components it accesses. Direct, multi-threaded invocation of its methods from outside the ECS scheduler is unsupported and will lead to race conditions.

## API Surface

The public API is defined by its role as an EntityTickingSystem. These methods are intended for consumption by the ECS scheduler, not by game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(N) | Processes all pending Velocity.Instruction objects for a single entity. N is the number of instructions, which is typically a small constant. |
| getDependencies() | Set | O(1) | Returns the predefined set of execution order dependencies. |
| getQuery() | Query | O(1) | Returns the query that selects entities with both NPCEntity and Velocity components. |

## Integration Patterns

### Standard Usage

Developers do not interact with this system directly. To influence an NPC's velocity, another system (e.g., combat, AI, ability systems) should add a Velocity.Instruction to the target entity's Velocity component. This system will automatically process the instruction on the subsequent server tick.

```java
// Correct Usage: Another system adds an instruction to an entity's component.
// The NPCVelocityInstructionSystem will process this automatically.

Velocity velocityComponent = entity.getComponent(Velocity.class);
Vector3d knockback = new Vector3d(0, 5, 10);
VelocityConfig config = VelocityConfig.DEFAULT;

velocityComponent.addInstruction(Velocity.Instruction.add(knockback, config));
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new NPCVelocityInstructionSystem()`. The ECS framework is solely responsible for the lifecycle of systems. Direct instantiation will result in a non-functional object that is not registered with the engine.
-   **Manual Invocation:** Do not call the `tick` method directly. This bypasses the dependency sorting and scheduling managed by the engine, which will break the execution order and cause unpredictable physics behavior, such as applying velocity changes twice or not at all.
-   **Post-Tick Inspection:** Do not attempt to read Velocity.Instruction objects in a system that is scheduled to run *after* this one. This system consumes and clears the instruction list as its final operation. Any subsequent system will find an empty list.

## Data Pipeline

The flow of data through this system is a classic "consume and delegate" pattern within a single engine tick.

> Flow:
> Other Game System -> Adds `Velocity.Instruction` to `Velocity` component -> ECS Scheduler invokes **NPCVelocityInstructionSystem** -> System reads instruction -> Delegates to `NPCEntity.getRole()` -> Role updates NPC state -> **NPCVelocityInstructionSystem** clears instruction list from `Velocity` component.

