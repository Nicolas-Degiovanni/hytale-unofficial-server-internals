---
description: Architectural reference for FailedSpawnSystem
---

# FailedSpawnSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Singleton / System

## Definition
```java
// Signature
public class FailedSpawnSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The FailedSpawnSystem is a specialized, reactive cleanup mechanism within the server-side Entity-Component-System (ECS) framework. Its sole responsibility is to identify and immediately remove entities that have been marked as spawn failures.

This system operates as an automated garbage collector for aborted entity creation processes. When a higher-level system, such as an NPC spawner, determines that an entity cannot be placed into the world correctly (e.g., due to an invalid location or missing prerequisites), it does not delete the entity directly. Instead, it attaches a **FailedSpawnComponent** to it.

The FailedSpawnSystem subscribes to all entities possessing this specific component. Upon detection, it wastes no time and issues a command to the world's CommandBuffer to permanently remove the entity. This pattern ensures that entity removal is transactional and synchronized with the main server tick, preventing partial or corrupted entities from ever persisting in the game state.

## Lifecycle & Ownership
- **Creation:** Instantiated and registered by the server's central ECS world orchestrator during server bootstrap. It is part of the core set of systems required for NPC management.
- **Scope:** Session-scoped. The system remains active for the entire lifetime of the server instance.
- **Destruction:** The system is destroyed when the server shuts down and the ECS world is torn down.

## Internal State & Concurrency
- **State:** This system is entirely **stateless**. It maintains no internal data, caches, or configuration between invocations. Its behavior is purely a function of the entity data passed to it by the ECS engine.
- **Thread Safety:** The Hytale ECS engine guarantees that all system callbacks, such as onEntityAdded, are executed serially within the main server game loop. The system is therefore not designed for concurrent access and must not be invoked from external threads. All interactions are mediated by the ECS CommandBuffer, ensuring thread safety at the framework level.

## API Surface
The public methods of this class are framework callbacks and are not intended for direct invocation by user code. The primary interaction is indirect, by adding a FailedSpawnComponent to an entity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(...) | void | O(1) | Framework callback. Triggers when an entity with a FailedSpawnComponent is added to the world. Immediately posts a removeEntity command. |
| getQuery() | Query | O(1) | Framework callback. Defines the component query that activates this system. |

## Integration Patterns

### Standard Usage
A developer does not call this system directly. Instead, they trigger its behavior by tagging an entity with the FailedSpawnComponent. This is the canonical pattern for flagging an entity for immediate, safe removal after a failed creation attempt.

```java
// In a separate system responsible for spawning, e.g., NpcSpawnerSystem

// An attempt to spawn an entity fails a validation check.
if (!isValidSpawnLocation(pos)) {
    // Instead of deleting the entity directly, we mark it as a failed spawn.
    // The FailedSpawnSystem will handle the cleanup on the next tick.
    Ref<EntityStore> failedEntityRef = world.createEntity();
    commandBuffer.addComponent(failedEntityRef, new FailedSpawnComponent());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new FailedSpawnSystem()`. The ECS engine is solely responsible for the lifecycle of all systems.
- **Manual Invocation:** Never call `system.onEntityAdded(...)` from your own code. Doing so bypasses the ECS scheduler and command buffer, which can corrupt the world state or cause concurrency violations.
- **Delayed Cleanup:** Do not add a FailedSpawnComponent with the expectation that the entity will persist for any length of time. The component's presence is a signal for *immediate* removal.

## Data Pipeline
The system acts as a terminal stage in the data flow for a failed entity's lifecycle. It converts a component-based signal into a destructive entity command.

> Flow:
> Spawning Logic Failure -> `commandBuffer.addComponent(FailedSpawnComponent)` -> ECS Engine Dispatch -> **FailedSpawnSystem.onEntityAdded** -> `commandBuffer.removeEntity()` -> World State Update (Entity Removed)

