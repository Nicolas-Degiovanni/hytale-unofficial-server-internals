---
description: Architectural reference for PredictedProjectileSystems.EntityTrackerUpdate
---

# PredictedProjectileSystems.EntityTrackerUpdate

**Package:** com.hypixel.hytale.server.core.modules.projectile.system
**Type:** System Component

## Definition
```java
// Signature
public static class EntityTrackerUpdate extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The EntityTrackerUpdate system is a critical server-side component that bridges the Entity Component System (ECS) with the client-side prediction network protocol. Its sole responsibility is to detect when a projectile entity configured for prediction becomes visible to a player for the first time.

When this visibility change occurs, this system constructs and queues a specialized **ComponentUpdate** packet. This packet does not contain state data; instead, it signals to the client that it should begin its own local, client-side prediction for the specified projectile, identified by a unique prediction ID. This mechanism is essential for reducing network latency and providing smooth projectile movement for players.

This system operates exclusively on entities that possess both a **PredictedProjectile** component and an **EntityTrackerSystems.Visible** component. It is part of the larger entity tracking module, which manages entity visibility and network replication.

### Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's ECS System Scheduler during world initialization. This system is discovered and registered as part of the server's core module loading process.
- **Scope:** Session-scoped. An instance of this system persists for the entire lifetime of a game world on the server.
- **Destruction:** The system is destroyed and its resources are released when the game world is unloaded or the server shuts down. Its lifecycle is entirely managed by the Hytale engine.

## Internal State & Concurrency
- **State:** This system is stateless between ticks. It holds immutable references to component types and a pre-compiled ECS query, but it does not retain or accumulate any data from one tick to the next. All operations are based on the current state of the components in the world.
- **Thread Safety:** This system is designed for parallel execution. The `isParallel` method indicates that the ECS scheduler may distribute the workload of processing different entity chunks across multiple worker threads. The underlying ECS framework guarantees that a single entity will not be processed by more than one thread simultaneously. All downstream operations, such as queueing updates on an **EntityViewer**, are expected to be thread-safe.

## API Surface
The public API consists of overrides from the **EntityTickingSystem** base class. These methods constitute the contract with the ECS scheduler and are not intended for direct invocation by developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGroup() | SystemGroup | O(1) | Assigns this system to the **EntityTrackerSystems.QUEUE_UPDATE_GROUP**, ensuring it runs after visibility calculations are complete. |
| getQuery() | Query | O(1) | Returns the pre-compiled ECS query for entities with **Visible** and **PredictedProjectile** components. |
| isParallel(archetypeChunkSize, taskCount) | boolean | O(1) | Determines if the system's tick logic can be executed in parallel across multiple threads. |
| tick(dt, index, archetypeChunk, store, commandBuffer) | void | O(N) | The core logic executed by the scheduler for each matching entity. Checks for new viewers and queues prediction packets. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. To enable client-side prediction for a projectile, a developer must add the **PredictedProjectile** component to the entity's archetype. The engine's systems will handle the rest.

```java
// Example: Spawning a projectile with prediction enabled
// This is the correct way to trigger the EntityTrackerUpdate system's logic.

Entity projectile = world.createEntity(projectileArchetype);

// The PredictedProjectile component is added, often via the archetype.
// If not in the archetype, it can be added manually.
PredictedProjectile predictionData = new PredictedProjectile();
predictionData.setUuid(UUID.randomUUID()); // Set a unique ID for prediction tracking.

commandBuffer.addComponent(projectile.getRef(), predictionData);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new EntityTrackerUpdate()`. The ECS scheduler is solely responsible for the lifecycle of systems. Direct instantiation will result in a non-functional object that is not registered with the engine.
- **Manual Invocation:** Do not call the `tick` method directly. This bypasses the scheduler, breaks parallelism, and can lead to severe race conditions and unpredictable behavior.
- **Stateful Logic:** Do not modify this system to store state across ticks. Systems should be idempotent and rely only on the current state of the ECS world.

## Data Pipeline
This system acts as a translator, converting an ECS state change (an entity becoming visible) into a network event.

> Flow:
> Entity Visibility Calculation -> **EntityTrackerUpdate System** -> ComponentUpdate Packet Creation -> EntityViewer Update Queue -> Network Serialization -> Client Receives Prediction Signal

