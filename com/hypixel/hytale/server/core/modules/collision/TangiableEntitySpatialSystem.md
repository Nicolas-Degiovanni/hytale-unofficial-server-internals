---
description: Architectural reference for TangiableEntitySpatialSystem
---

# TangiableEntitySpatialSystem

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** System Component

## Definition
```java
// Signature
public class TangiableEntitySpatialSystem extends SpatialSystem<EntityStore> {
```

## Architecture & Concepts
The TangiableEntitySpatialSystem is a specialized system within the server's Entity Component System (ECS) framework. Its primary function is to act as a filter and data provider for the engine's core spatial partitioning system. It is responsible for identifying all *physical*, or *tangiable*, entities and feeding their positional data into the parent SpatialSystem.

This system operates on a specific subset of entities defined by its internal QUERY. It selects entities that possess both a TransformComponent and a BoundingBox, while explicitly excluding any entity marked with the Intangible component. This precise filtering ensures that only entities capable of physical interaction are indexed in the spatial database, optimizing collision detection and proximity queries by ignoring non-physical entities like visual effects or logical triggers.

Architecturally, it serves as a bridge between the raw component data stored in the EntityStore and the highly optimized data structures (like Octrees or Grids) managed by the parent SpatialSystem. It does not perform collision checks itself; rather, it prepares and maintains the dataset that other physics and AI systems rely on for efficient spatial queries.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's core module loader or system graph manager during world initialization. It is not intended for manual creation by developers.
-   **Scope:** The lifecycle of a TangiableEntitySpatialSystem instance is bound to a server world instance. It persists as long as the world is loaded and running.
-   **Destruction:** The instance is marked for garbage collection when the corresponding server world is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless. Its behavior is dictated by its static, immutable QUERY. All stateful data, such as the spatial index of entities, is managed by its parent class, SpatialSystem. It acts as a stateless processor that operates on the mutable state of the global EntityStore.
-   **Thread Safety:** The class itself is thread-safe. However, it is designed to be operated by a single-threaded system scheduler as part of the main server game loop. The tick method, which triggers the update of the parent SpatialSystem, is not re-entrant and must not be called concurrently. The underlying component data, such as ArchetypeChunk, is assumed to be accessed only from the designated scheduler thread during the system's update phase to prevent data races.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static query used to identify all tangible entities. |
| tick(dt, systemIndex, store) | void | O(N log N) | Executes one update cycle. Delegates to the parent SpatialSystem to process and index all entities matching the query. Complexity depends on the parent's algorithm. |
| getPosition(archetypeChunk, index) | Vector3d | O(1) | Callback method used by the parent system to retrieve the world position of a specific entity from its TransformComponent. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. Instead, they control its behavior by manipulating entity components. To make an entity eligible for processing by this system, ensure it has the required components.

```java
// An entity with these components will be processed by this system
// and indexed for collision and physics queries.
Entity entity = world.createEntity();
entity.add(new TransformComponent());
entity.add(new BoundingBox());

// To make the entity intangible and thus IGNORED by this system:
entity.add(new Intangible());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new TangiableEntitySpatialSystem()`. The engine's system manager is responsible for its lifecycle. Direct creation will result in a disconnected system that is not part of the main update loop.
-   **Manual Ticking:** Do not call the tick method manually. The server's system scheduler invokes this method in a specific order to ensure data consistency. Manual invocation will corrupt the spatial state and lead to unpredictable physics behavior.
-   **Modifying Components Mid-Tick:** Modifying TransformComponent or BoundingBox on an entity from another thread while the collision systems are ticking can lead to severe race conditions and server instability. All component modifications should be queued or performed during designated safe update phases.

## Data Pipeline
The system processes data by filtering the main entity store and feeding the results into the spatial index.

> Flow:
> EntityStore (All Entities) -> **TangiableEntitySpatialSystem.QUERY** -> Filtered Set of Tangible Entities -> **getPosition()** -> Position Data -> Parent SpatialSystem (Octree) -> Other Systems (Collision, AI)

