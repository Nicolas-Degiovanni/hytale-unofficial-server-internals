---
description: Architectural reference for EntitySpatialSystem
---

# EntitySpatialSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** System Component

## Definition
```java
// Signature
public class EntitySpatialSystem extends SpatialSystem<EntityStore> {
```

## Architecture & Concepts
The EntitySpatialSystem is a core component of the server-side Entity Component System (ECS) responsible for maintaining an accelerated spatial index of game entities. It is a specialized implementation of the generic SpatialSystem, tailored specifically for standard, physical entities within a world.

Its primary function is to bridge the gap between an entity's raw position data, stored in a TransformComponent, and the high-level spatial partitioning grid managed by its parent class. This allows other game systems, such as AI, physics, and network culling, to perform highly efficient spatial queries (e.g., "find all entities within this radius") without iterating through every entity in the world.

The system's behavior is precisely defined by its static **QUERY**. This query selects all entities that:
1.  **Have** a TransformComponent, meaning they exist physically in the world.
2.  Do **NOT** have an Intangible component, excluding ghosts or decorative entities.
3.  Do **NOT** have a Player component, as players are managed by a separate, more complex spatial system.

By delegating the core indexing logic to the parent SpatialSystem, this class adheres to the template method pattern. Its sole responsibilities are to define *which* entities to track and *how* to extract their position.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's central System Executor or World Manager during the world bootstrap sequence. It is provided a managed SpatialResource dependency during construction.
- **Scope:** Session-scoped. An instance of EntitySpatialSystem persists for the entire lifetime of the server world it belongs to.
- **Destruction:** The system is marked for garbage collection when its parent world is unloaded or the server shuts down. There is no explicit public destruction method.

## Internal State & Concurrency
- **State:** This class itself is stateless. However, it operates upon and directs the mutation of a highly stateful SpatialResource, which is managed by its parent, SpatialSystem. This underlying resource contains the complete spatial index (e.g., a hash grid or octree) for all matching entities. The state of this index is modified every tick.
- **Thread Safety:** **Not thread-safe.** This system is designed to be executed exclusively on the main server game loop thread. Calling its methods, particularly *tick*, from any other thread will lead to state corruption, data races, and server instability. All interaction with the spatial index must be synchronized with the main tick cycle.

## API Surface
The public API is minimal and intended for framework integration, not direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(N) | Processes all entities matching the query. Invoked by the ECS executor each game tick. |
| getQuery() | Query | O(1) | Returns the static query defining which entities this system manages. |
| getPosition(chunk, index) | Vector3d | O(1) | Extracts the world position from an entity's TransformComponent. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. Its operation is implicit and managed by the ECS framework. Interaction occurs by modifying entity components. Adding a TransformComponent to an entity will cause it to be processed by this system on the next tick. Adding an Intangible component will cause it to be removed.

The system is registered with the main world executor during server startup.

```java
// Conceptual example of system registration
World world = server.getWorld("world1");
ResourceType<EntityStore, SpatialResource> spatialResource = ...;

// The engine creates and registers the system
world.getSystemExecutor().register(new EntitySpatialSystem(spatialResource));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new EntitySpatialSystem()`. The system requires a managed SpatialResource from the engine's resource manager. Failure to provide this will result in a NullPointerException.
- **Manual Invocation:** Do not call the *tick* method directly. This will bypass the engine's update order and synchronization, corrupting the spatial index and causing unpredictable behavior in dependent systems like physics and AI.
- **State Assumption:** Do not assume the spatial index is up-to-date outside of the system execution phase. An entity's position in the index is only guaranteed to be correct *after* the EntitySpatialSystem's tick has completed for a given frame.

## Data Pipeline
The system transforms component data into an optimized spatial index that can be queried by other systems.

> Flow:
> ECS Executor -> Identifies entities matching **QUERY** -> `tick()` method invoked -> **EntitySpatialSystem** -> Calls `getPosition()` for each entity -> Retrieves `Vector3d` from TransformComponent -> Parent `SpatialSystem` updates internal grid -> Updated `SpatialResource` is available for other systems (e.g., CollisionSystem, NetworkSystem) to query.

