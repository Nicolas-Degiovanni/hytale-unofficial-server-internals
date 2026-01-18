---
description: Architectural reference for SpawnMarkerSuppressionSystem
---

# SpawnMarkerSuppressionSystem

**Package:** com.hypixel.hytale.server.spawning.suppression.system
**Type:** Transient

## Definition
```java
// Signature
public class SpawnMarkerSuppressionSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The SpawnMarkerSuppressionSystem is a reactive, stateless system within the server-side Entity-Component-System (ECS) framework. Its sole responsibility is to enforce spawn suppression rules on newly created Spawn Marker entities.

This system operates as a bridge between two distinct concepts:
1.  **Global State:** The SpawnSuppressionController, an ECS Resource that maintains a world-wide map of all active spawn suppression zones (e.g., placed by players or triggered by world events).
2.  **Entity State:** Individual entities that possess a SpawnMarkerEntity component, signifying they are potential locations for mob spawning.

When an entity that qualifies as a spawn marker is added to the world, the ECS engine automatically invokes this system. The system then queries the global SpawnSuppressionController to determine if the new marker's position falls within any active suppression radii. If it does, the system updates the entity's SpawnMarkerEntity component to mark it as suppressed.

It is critical to understand that this system only acts **at the moment of entity creation**. It does not periodically re-evaluate markers if suppression zones are created or moved after the marker already exists. A separate system is responsible for that continuous evaluation.

### Lifecycle & Ownership
- **Creation:** Instantiated once per world by the SpawningPlugin during the server's world-loading sequence. Its dependencies (ComponentType and ResourceType) are injected at construction, indicating it is managed by a framework. It is then registered with the world's central SystemManager.
- **Scope:** The system's lifetime is strictly bound to the EntityStore (the game world) it is registered with. It remains active as long as the world is loaded.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the corresponding game world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This system is fundamentally **stateless**. It holds no mutable state of its own and does not cache data between invocations. All fields are final and are used to define its behavior and query. Each call to onEntityAdded operates exclusively on the data provided by the ECS framework (the entity's components) and the global SpawnSuppressionController resource.

- **Thread Safety:** This system is **not thread-safe** and must only be operated by the single-threaded ECS engine that manages the EntityStore. The methods onEntityAdded and onEntityRemove are designed as callbacks from a synchronous game loop.

    **Warning:** Manually invoking its methods from an external thread will bypass the engine's command buffering and synchronization, leading to race conditions, data corruption, and server instability.

## API Surface
The public contract of this class is defined by its implementation of the RefSystem interface. These methods are intended for invocation by the ECS engine, not by developers directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(...) | void | O(N) | Callback executed when a new entity matching the query is added. N is the number of active suppressors in the world. |
| onEntityRemove(...) | void | O(1) | Callback executed when a matching entity is removed. This implementation is a no-op. |
| getQuery() | Query | O(1) | Returns the query that defines which entities this system processes. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. Its functionality is triggered implicitly by the game engine. The system is registered during server initialization, and its logic is automatically applied to any new entity that has the required components.

The following example illustrates how the system is *registered*, which is its only direct point of interaction.

```java
// During world initialization (conceptual)
EntityStore world = server.getWorld("world1");
SystemManager systemManager = world.getSystemManager();

// The system is created and registered with the world's manager
systemManager.register(
    new SpawnMarkerSuppressionSystem(
        SpawnMarkerEntity.getComponentType(),
        SpawnSuppressionController.getResourceType()
    )
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SpawnMarkerSuppressionSystem()` in game logic. An instance is useless unless it is registered with a SystemManager, which is handled internally by the spawning module.
- **Manual Invocation:** Never call `onEntityAdded` directly. This bypasses the ECS engine's transactional command buffer and query caching, which can corrupt entity state and cause unpredictable behavior. The engine is the sole authority for invoking system callbacks.
- **Stateful Modification:** Do not modify the system to cache results or hold mutable state. This violates the stateless principle of ECS systems and will cause issues in a distributed or multi-threaded environment.

## Data Pipeline
The system operates as a step in a larger data flow managed by the ECS engine. It is purely reactive to entity creation events.

> Flow:
> Entity Creation Event -> ECS Engine Query Match -> **SpawnMarkerSuppressionSystem.onEntityAdded** -> Read SpawnSuppressionController -> Distance Check -> Write to SpawnMarkerEntity Component (via CommandBuffer)

