---
description: Architectural reference for LocalSpawnSetupSystem
---

# LocalSpawnSetupSystem

**Package:** com.hypixel.hytale.server.spawning.local
**Type:** System Component

## Definition
```java
// Signature
public class LocalSpawnSetupSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The LocalSpawnSetupSystem is a reactive component within the server-side Entity Component System (ECS) framework. Its sole responsibility is to bootstrap the local spawning logic for newly created player entities. It operates on the principle of "separation of concerns"; rather than embedding spawning setup logic directly into player creation code, this system observes the world for new players and attaches the necessary components post-facto.

This system acts as an automated initializer. When an entity with a Player component is added to the world, this system is triggered by the ECS scheduler. It then issues a command to ensure a LocalSpawnController component is attached to that same player entity. This decouples the core Player entity definition from the specific implementation of its spawning behavior, allowing for greater modularity.

## Lifecycle & Ownership
- **Creation:** Instantiated once during the server's primary bootstrap sequence. It is registered with the central ECS System Manager, which is responsible for invoking its methods at the correct time. The specific ComponentType it queries for (Player) is injected via its constructor, indicating it is configured by a higher-level authority.
- **Scope:** Session-scoped. It persists for the entire lifetime of the server or world instance it is registered with.
- **Destruction:** The system is destroyed and garbage collected when the server or world instance is shut down and the ECS System Manager is cleared.

## Internal State & Concurrency
- **State:** This system is effectively stateless. It holds a single immutable reference to the Player ComponentType, which is set at construction. It does not maintain or modify any state between invocations.
- **Thread Safety:** This class is thread-safe. Its reactive methods, onEntityAdded and onEntityRemove, are invoked by the ECS scheduler. All world-state mutations are deferred via a CommandBuffer. This is a critical concurrency pattern that prevents direct modification of the component store during system iteration. By queueing the addition of the LocalSpawnController component, the system ensures that all changes are applied deterministically at a synchronized point in the game tick, eliminating race conditions.

## API Surface
The public methods of this class are framework callbacks and are not intended for direct invocation by application code. They form the contract with the parent RefSystem and the ECS scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(...) | void | O(1) | Framework callback. Invoked when an entity matching getQuery is added. Queues a command to attach a LocalSpawnController. |
| onEntityRemove(...) | void | O(1) | Framework callback. Invoked on entity removal. Currently a no-op. |
| getQuery() | Query | O(1) | Framework callback. Returns the query that defines which entities this system operates on, specifically entities with a Player component. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. Its operation is automatic and transparent, triggered by the creation of a Player entity. The system is configured and registered once at server startup.

The following example demonstrates the *effect* of the system, not direct usage of it.

```java
// 1. A new Player entity is created somewhere in the game logic.
Ref<EntityStore> playerEntity = world.createEntity();
commandBuffer.addComponent(playerEntity, new Player(...));

// 2. During the next system update phase, LocalSpawnSetupSystem is automatically triggered.
// 3. The system adds a LocalSpawnController to the entity via a command buffer.
// 4. Later, other systems can query for and use the controller.
LocalSpawnController controller = playerEntity.get(LocalSpawnController.class);
controller.setSpawnPoint(newSpawn);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new LocalSpawnSetupSystem()`. The system will not be registered with the ECS scheduler and will have no effect. It must be instantiated and managed by the server's core bootstrap logic.
- **Manual Invocation:** Never call `onEntityAdded` or `onEntityRemove` directly. Doing so bypasses the ECS scheduler and, critically, the CommandBuffer mechanism. This can lead to severe concurrency bugs, such as modifying a collection while it is being iterated over, and will corrupt world state.

## Data Pipeline
The system acts as a simple, event-driven processor in the entity lifecycle pipeline. It listens for a specific event (Player creation) and injects a new component into the entity's data structure.

> Flow:
> Player Component Added to Entity -> ECS Scheduler -> **LocalSpawnSetupSystem.onEntityAdded** -> CommandBuffer.ensureComponent -> Command Flush -> LocalSpawnController Component Added to Entity

