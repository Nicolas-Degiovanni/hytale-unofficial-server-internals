---
description: Architectural reference for PathSpatialSystem
---

# PathSpatialSystem

**Package:** com.hypixel.hytale.builtin.path
**Type:** System Component

## Definition
```java
// Signature
public class PathSpatialSystem extends SpatialSystem<EntityStore> {
```

## Architecture & Concepts
The PathSpatialSystem is a specialized implementation of the core SpatialSystem. Its sole responsibility is to identify all entities that represent patrol path markers and integrate them into the server's spatial partitioning data structures. It acts as a declarative bridge between the Entity Component System (ECS) and the spatial query engine.

This system does **not** perform pathfinding or AI logic. Instead, it populates a spatial index (managed by the parent SpatialSystem) with the locations of `PatrolPathMarkerEntity` instances. This allows other, higher-level systems—such as an AI behavior system—to perform highly efficient spatial queries, like "find all patrol nodes within a 50-meter radius".

The system's logic is driven by its static `QUERY` field, an Archetype that filters for entities possessing both a `PatrolPathMarkerEntity` component and a `TransformComponent`. The `getPosition` method then serves as an adapter, instructing the generic `SpatialSystem` on how to extract a `Vector3d` position from the `TransformComponent` of a matched entity.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's central `SystemRegistry` during the world initialization phase. The engine's dependency injection mechanism provides the required `ResourceType` to its constructor. Direct instantiation is not supported.
-   **Scope:** This system is long-lived. Its lifecycle is bound to the server's world instance. It persists as long as the world is loaded and active.
-   **Destruction:** The system is destroyed and its resources are released when the world is unloaded or the server shuts down. This process is managed automatically by the engine.

## Internal State & Concurrency
-   **State:** The `PathSpatialSystem` class itself is stateless. It contains no mutable fields and only provides configuration and data access logic. All mutable state, specifically the spatial index data structure (e.g., an octree or grid), is encapsulated and managed within its parent `SpatialSystem` class.
-   **Thread Safety:** This system is designed to be operated by a single thread as part of the main server game loop. The `tick` method is **not thread-safe** and must not be invoked concurrently. Accessing the underlying spatial resource from other threads may require explicit synchronization or deferral to the main thread, depending on the guarantees provided by the `SpatialResource` implementation.

## API Surface
The public API is minimal, as the system is primarily controlled by the engine's scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static Archetype used by the engine to identify relevant entities for processing. |
| tick(dt, systemIndex, store) | void | O(N) | Invoked by the engine each frame to update the spatial index with any changes to path marker positions. N is the number of matched entities. |
| getPosition(chunk, index) | Vector3d | O(1) | A callback method used by the parent `SpatialSystem` to extract the world position from a specific entity's `TransformComponent`. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. The intended pattern is to create entities with the required components. The system will automatically discover and manage them.

```java
// Example: Creating a patrol marker entity that this system will automatically index.
// This code would exist within a world generation or entity creation system.

Entity entity = world.createEntity();
entity.addComponent(new TransformComponent(new Vector3d(100, 64, -50)));
entity.addComponent(new PatrolPathMarkerEntity()); // This component flags the entity

// The PathSpatialSystem will now automatically find and index this entity on its next tick.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PathSpatialSystem()`. The system requires an engine-managed `ResourceType` and must be registered with the server's system scheduler to function.
-   **Manual Ticking:** Never call the `tick` method manually. Doing so will disrupt the server's update order, potentially leading to race conditions, corrupted spatial data, and crashes.
-   **Component Mismatch:** Creating an entity with `PatrolPathMarkerEntity` but no `TransformComponent` will cause the entity to be ignored by this system, as it cannot satisfy the `QUERY` archetype.

## Data Pipeline
The system operates as a processor in the server's data flow, transforming component data into an optimized spatial index.

> Flow:
> EntityStore (Creation of an entity with `PatrolPathMarkerEntity` + `TransformComponent`) -> Server Scheduler (Matches entity against `PathSpatialSystem.getQuery()`) -> **`PathSpatialSystem.tick()`** -> `getPosition()` is called for the entity -> `SpatialSystem` (Parent class updates its internal spatial index) -> `SpatialResource` (The updated index is now available for queries by other systems)

