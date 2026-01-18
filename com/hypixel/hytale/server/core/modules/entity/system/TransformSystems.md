---
description: Architectural reference for TransformSystems
---

# TransformSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Utility / System Grouping

## Definition
```java
// Signature
public class TransformSystems {
    // Contains static nested System classes
}
```

## Architecture & Concepts
The TransformSystems class is not a system itself, but a static container for a group of highly critical Entity Component Systems (ECS) that manage entity transforms. These systems are the definitive authority for processing, synchronizing, and cleaning up entity position, rotation, and scale data on the server.

This grouping encapsulates two distinct but related responsibilities:
1.  **Network Replication:** Detecting changes in an entity's transform and broadcasting those updates to relevant clients.
2.  **Lifecycle Management:** Ensuring that an entity's spatial data is correctly handled when it is removed from the world.

These systems form a bridge between the server's internal game state and the external representation observed by players, making their correct function essential for world consistency.

## System: EntityTrackerUpdate

### Definition
```java
// Signature
public static class EntityTrackerUpdate extends EntityTickingSystem<EntityStore> {
```

### Architecture & Concepts
EntityTrackerUpdate is a high-frequency system responsible for the network replication of entity movement and rotation. It runs within the main server tick loop and operates on all entities that are visible to at least one player and possess a transform.

The core architectural pattern is **state diffing**. The system continuously compares an entity's current transform data (from TransformComponent and HeadRotation) against the last state that was sent over the network (cached within the TransformComponent's *sentTransform* field). This comparison prevents the server from flooding clients with redundant data for entities that have not moved.

A network update is generated under two conditions:
1.  The entity's position, body rotation, or head rotation has changed since the last sent update.
2.  The entity has become newly visible to a player. This is a critical edge case that ensures players immediately receive transform data for entities that enter their view, regardless of whether the entity is currently moving.

This system is fundamental to the player's perception of a living, dynamic world. Its performance and correctness directly impact network bandwidth and visual consistency.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's central System Scheduler during the bootstrap sequence. It is not intended for manual creation.
-   **Scope:** Singleton. A single instance persists for the entire server session.
-   **Destruction:** The instance is discarded and garbage collected during server shutdown.

### Internal State & Concurrency
-   **State:** This system is **stateless**. It holds no mutable fields related to the entities it processes. All state is read from and written to the components of the entities being processed in the current tick. The TransformComponent itself is stateful, caching the last sent transform.
-   **Thread Safety:** The system is designed for parallel execution. The `isParallel` method allows the ECS scheduler to distribute the workload of processing different entity chunks across multiple worker threads. All operations within the `tick` method are confined to the components of a single entity, and the final `queueUpdate` call on the EntityViewer is expected to be a thread-safe operation.

### API Surface
The public API is designed for consumption by the ECS framework, not by developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, commands) | void | O(1) | Called by the scheduler for each entity matching the query. Contains the core state diffing and update queuing logic. |
| getQuery() | Query | O(1) | Defines the component signature for entities this system will process: must have Visible and TransformComponent. |
| getGroup() | SystemGroup | O(1) | Assigns this system to the `QUEUE_UPDATE_GROUP`, ensuring it runs in the correct order relative to other entity tracking systems. |

### Integration Patterns
This system is not invoked directly. It is automatically executed by the server's main tick loop for all entities matching its query. A developer's interaction with this system is indirect: by modifying an entity's TransformComponent, they are flagging it for potential processing by EntityTrackerUpdate in the next tick.

The system is tightly coupled with the EntityTrackerSystems, from which it sources the `Visible` component to determine which clients should receive updates.

### Data Pipeline
The flow of data is from the game simulation, through this system, and into the network output queue.

> Flow:
> Physics/AI System updates TransformComponent -> **EntityTrackerUpdate** detects change -> A ModelTransform protocol message is created -> The message is queued in an EntityViewer (per-player queue) -> Network Flush System sends the update to the client.

---

## System: OnRemove

### Definition
```java
// Signature
public static class OnRemove extends HolderSystem<EntityStore> {
```

### Architecture & Concepts
OnRemove is a lifecycle-event-driven system. Unlike a ticking system, it does not run every frame. Instead, the ECS framework invokes it only when an entity possessing a TransformComponent is removed from the world.

Its sole purpose is to perform critical cleanup of spatial partitioning data. When an entity exists, it is registered within the world's chunk system for efficient spatial queries. If this registration is not cleared upon the entity's removal, it results in a dangling reference, leading to memory leaks and potential crashes from systems attempting to access a destroyed entity. This system prevents such issues by explicitly nullifying the entity's chunk location.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's System Scheduler at startup.
-   **Scope:** Singleton. A single instance persists for the entire server session.
-   **Destruction:** The instance is discarded during server shutdown.

### Internal State & Concurrency
-   **State:** This system is **stateless**.
-   **Thread Safety:** Entity removal is a sensitive operation. The ECS framework guarantees that this system's `onEntityRemoved` method is called in a safe context, preventing race conditions with other systems that might be trying to access the entity during its destruction.

### API Surface
The public API is a set of callbacks for the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityRemoved(holder, reason, store) | void | O(1) | The core callback, invoked by the framework when an entity matching the query is destroyed. Executes the cleanup logic. |
| getQuery() | Query | O(1) | Defines the component signature for entities this system will monitor: any entity with a TransformComponent. |

### Integration Patterns
This system is a prime example of a reactive, event-driven system within the ECS architecture. Developers do not call it. They trigger its execution by deleting an entity that has a TransformComponent. The framework uses the system's query to determine that OnRemove has "subscribed" to the removal event for that type of entity and invokes it accordingly.

**Warning:** Any new component that registers an entity's location in an external data structure **must** have a corresponding `OnRemove` system to ensure proper cleanup. Failure to do so is a common source of severe bugs.

### Data Pipeline
The data flow is a simple, triggered cleanup process.

> Flow:
> Game Logic requests entity deletion -> ECS Framework processes removal -> Framework identifies entity has TransformComponent -> **OnRemove** system is invoked -> System accesses component and calls `setChunkLocation(null, null)` -> Spatial partitioning data is cleared.

