---
description: Architectural reference for SpawnMarkerSpatialSystem
---

# SpawnMarkerSpatialSystem

**Package:** com.hypixel.hytale.server.spawning.systems
**Type:** Transient

## Definition
```java
// Signature
public class SpawnMarkerSpatialSystem extends SpatialSystem<EntityStore> {
```

## Architecture & Concepts
The SpawnMarkerSpatialSystem is a specialized component within the server's Entity-Component-System (ECS) framework. It does not implement unique logic; instead, it serves as a configuration layer for the generic, high-performance SpatialSystem it inherits from.

Its primary architectural role is to register a specific type of entity—the spawn marker—with the world's spatial partitioning data structure. This allows other game systems, particularly the entity spawning logic, to perform highly efficient proximity and area queries, such as "find all spawn markers within this chunk" or "find the nearest spawn marker to this player".

The system's behavior is defined by two key overrides:
1.  **Entity Filtering:** It provides a static Archetype, named QUERY, which instructs the ECS engine to only process entities that possess both a SpawnMarkerEntity component and a TransformComponent.
2.  **Position Extraction:** It implements the getPosition method, providing the concrete logic for reading the world-space coordinates from an entity's TransformComponent.

The parent SpatialSystem consumes this configuration to manage the lifecycle of spawn markers within the underlying spatial resource, which is typically an octree, spatial hash, or grid.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's world or system manager during the world initialization phase. Its required SpatialResource dependency is injected by the engine, forbidding direct manual creation.
-   **Scope:** The instance is scoped to a single game world. Its lifetime is directly coupled to the lifetime of the world it services.
-   **Destruction:** The object is marked for garbage collection when the corresponding game world is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is stateless. The QUERY field is a static constant, and the class itself holds no mutable instance fields. All stateful data, such as the spatial index of spawn markers, is managed externally within the injected SpatialResource and the parent SpatialSystem.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, it is designed to be operated exclusively by the main server thread as part of the synchronous game tick. The underlying SpatialResource it interacts with is not guaranteed to be thread-safe, and concurrent modification from other threads will lead to world state corruption.

**WARNING:** Do not invoke methods on this system from asynchronous tasks or parallel threads. All interactions must be scheduled on the main server game loop.

## API Surface
The public API is intended for consumption by the ECS engine, not for general developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static entity archetype query. Used by the engine to identify relevant entities. |
| tick(dt, systemIndex, store) | void | O(N) | Engine entry point. Defers to the parent SpatialSystem to process all matching entities. N is the number of entities matching the query. |
| getPosition(archetypeChunk, index) | Vector3d | O(1) | Callback used by the parent system to extract the position from a TransformComponent. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. It is automatically registered and executed by the server's system scheduler. To make an entity trackable by this system, simply add the required components to it.

```java
// An entity created with these components will be automatically
// processed by SpawnMarkerSpatialSystem on the next server tick.
Entity entity = world.createEntity();
entity.addComponent(new SpawnMarkerEntity());
entity.addComponent(new TransformComponent(new Vector3d(100, 64, -50)));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SpawnMarkerSpatialSystem()`. The system requires an engine-managed SpatialResource to function and must be instantiated by the world's dependency injection container.
-   **Manual Invocation:** Calling the tick method manually will bypass the engine's scheduler and can cause severe state desynchronization, especially if called out of order relative to other systems.

## Data Pipeline
This system acts as a data bridge, feeding positional information from the ECS component store into a specialized spatial data structure for rapid lookups.

> Flow:
> 1. An entity is created with **SpawnMarkerEntity** and **TransformComponent**.
> 2. The ECS engine identifies this entity as matching the system's **QUERY**.
> 3. On the next server tick, the engine invokes the **tick** method.
> 4. The parent **SpatialSystem** logic iterates through matching entities.
> 5. For each entity, it calls the overridden **getPosition** method to retrieve its **Vector3d**.
> 6. The position is used to update the entity's record in the shared **SpatialResource**.
> 7. Other systems can now query the **SpatialResource** to find spawn markers efficiently.

