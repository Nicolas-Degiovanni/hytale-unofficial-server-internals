---
description: Architectural reference for TargetMemorySystems.Ticking
---

# TargetMemorySystems.Ticking

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.memory
**Type:** System Component

## Definition
```java
// Signature
public static class Ticking extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The Ticking system is a fundamental component of the Non-Player Character (NPC) AI framework, operating within the server-side Entity Component System (ECS). Its sole responsibility is to manage the lifecycle of an entity's target memory, ensuring that AI decision-making is based on current and valid data.

This system functions as a high-frequency janitor for the **TargetMemory** component. It does not actively find new targets; rather, it prunes the existing lists of known hostiles and friendlies based on a set of validation rules and a time-to-live (TTL) mechanism.

Its execution order is critical. By declaring a dependency to run *before* the **RoleSystems.BehaviourTickSystem**, it guarantees that all memory cleanup for a given tick is completed before the primary AI behavior tree is evaluated. This prevents NPCs from attempting to interact with targets that are dead, disconnected, or otherwise invalid, thus avoiding entire classes of state-consistency bugs.

## Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's central ECS System Manager during world initialization. This is an engine-level process; developers do not create instances of this system manually.
-   **Scope:** The singleton instance persists for the entire lifetime of the game world or server session. It is a long-lived, stateless processor.
-   **Destruction:** The instance is destroyed and garbage collected during the server shutdown sequence when the ECS System Manager is decommissioned.

## Internal State & Concurrency
-   **State:** The Ticking system is **stateless**. Its internal fields are final and configured at construction. All state it manipulates is external, residing within the **TargetMemory** components attached to individual entities. This design is crucial for enabling safe parallel execution.

-   **Thread Safety:** This system is designed to be highly parallelizable and is thread-safe. The `isParallel` method delegates to a utility that determines if the workload is large enough to benefit from multi-threading. The ECS scheduler guarantees that parallel execution will only occur across different, independent **ArchetypeChunks**. Because the system is stateless and only operates on data within the chunk provided to its `tick` method, there are no risks of data races between threads. All modifications are queued into a thread-local **CommandBuffer**, which the ECS framework later merges safely.

## API Surface
The public API is exclusively for consumption by the ECS framework. Direct invocation by user code is a critical anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(N) | The primary entry point called by the ECS scheduler each tick. N is the number of known targets in an entity's memory. Processes a single entity's target memory. |
| getQuery() | Query | O(1) | Defines the component signature for entities this system will operate on. Returns all entities with a **TargetMemory** component. |
| getDependencies() | Set | O(1) | Declares scheduling constraints. Critically, it ensures this system runs before the main AI behavior system. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. To make an NPC's target memory subject to the decay and validation logic, simply attach a **TargetMemory** component to the entity's archetype. The ECS framework will automatically discover the entity and schedule this system to process it on every server tick.

```java
// Example: Adding the component that makes an entity eligible for this system
// This is typically done during entity creation.

Entity entity = world.createEntity();
CommandBuffer buffer = world.getCommandBuffer();

// By adding this component, TargetMemorySystems.Ticking will now process this entity
buffer.addComponent(entity, new TargetMemory());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new TargetMemorySystems.Ticking()`. The ECS framework manages the lifecycle of all systems. Manual instantiation will result in a non-functional object that is not registered with the engine's scheduler.
-   **Direct Invocation:** Never call the `tick` method directly. Doing so bypasses the dependency scheduler, thread management, and command buffer synchronization provided by the ECS framework. This will lead to race conditions, state corruption, and unpredictable AI behavior.

## Data Pipeline
This system processes data in a linear, per-entity flow orchestrated by the ECS scheduler. For each entity matching its query, it performs a validation and decay pass on its target memory.

> Flow:
> ECS Scheduler -> **Ticking.tick()** -> Reads **TargetMemory** Component -> Iterates Hostile/Friendly Lists -> Validates each target (Is Alive? Is Valid Ref? Is Correct GameMode?) -> Decrements Memory Timer -> Queues Removal in **CommandBuffer** if Invalid/Expired -> Modified **TargetMemory** Component

