---
description: Architectural reference for LegacyBeaconSpatialSystem
---

# LegacyBeaconSpatialSystem

**Package:** com.hypixel.hytale.server.spawning.systems
**Type:** System Component

## Definition
```java
// Signature
public class LegacyBeaconSpatialSystem extends SpatialSystem<EntityStore> {
```

## Architecture & Concepts
The LegacyBeaconSpatialSystem is a specialized component within the server-side Entity Component System (ECS) framework. Its primary function is to act as a bridge between the logical representation of spawn beacon entities and the engine's physical spatial partitioning system.

By extending SpatialSystem, this class registers itself as the authority for providing positional data for a specific subset of entities. The `QUERY` constant defines this subset: any entity possessing both a LegacySpawnBeaconEntity component and a TransformComponent.

This system is a critical performance and organizational primitive. It enables other high-level systems, such as the player spawning manager, to perform efficient spatial queries like "find all spawn beacons within a 50-meter radius" without iterating through every entity in the world. The underlying SpatialResource, which this system populates, typically uses an optimized data structure like an octree or a spatial hash grid for these lookups.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level authority, such as a World or SystemManager, during server world initialization. The constructor's requirement for a SpatialResource indicates that it is configured and owned by the framework that manages the world's spatial data.
-   **Scope:** The lifecycle of a LegacyBeaconSpatialSystem instance is tightly coupled to the server world it operates within. It persists for the entire duration of the world session.
-   **Destruction:** The object is eligible for garbage collection when the server world is unloaded and all references from the managing ECS framework are released. It does not implement any explicit cleanup or disposal logic.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless. Its only member field, QUERY, is a static final constant. All data it operates on is passed as arguments to its methods (e.g., the Store in the tick method). The true state it helps manage resides within the SpatialResource provided during construction and the ECS Store.
-   **Thread Safety:** This system is designed to be safe for concurrent execution by the ECS scheduler. The getPosition method is a pure read operation on an ArchetypeChunk, which is safe. The heavy lifting of concurrent state modification is delegated to the parent `SpatialSystem.tick` method, which is expected to use appropriate synchronization to update the shared SpatialResource.

    **WARNING:** Developers extending this pattern must not introduce mutable instance fields. Doing so would violate the stateless contract and create severe race conditions when the ECS scheduler runs systems in parallel.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static Archetype query used to identify relevant beacon entities. |
| tick(dt, systemIndex, store) | void | O(N) | Main entry point for the engine's update loop. Delegates to the parent SpatialSystem to update the spatial index for all matching entities. N is the number of entities matching the query. |
| getPosition(archetypeChunk, index) | Vector3d | O(1) | Callback used by the parent system to retrieve the world position of a specific entity from its TransformComponent. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. It is automatically discovered and registered with the server's ECS SystemManager during world setup. The engine is solely responsible for invoking its `tick` method at the correct time.

```java
// Example of engine-level system registration (conceptual)
World world = createNewWorld();
SpatialResource<Ref<EntityStore>, EntityStore> spatialIndex = world.getSpatialIndex();

// The system is instantiated and registered with the engine
world.getSystemManager().register(new LegacyBeaconSpatialSystem(spatialIndex));

// The engine's main loop will then manage the system automatically
world.tick(deltaTime);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new LegacyBeaconSpatialSystem()` in game logic. The system must be created and managed by the world's core ECS framework to ensure it receives the correct SpatialResource and is ticked properly.
-   **Manual Ticking:** Never call the `tick` method manually. This would bypass the engine's scheduler, break frame synchronization, and likely cause concurrency exceptions or corrupted spatial data.
-   **Stateful Implementation:** Do not add mutable instance variables to this class. The system is expected to be stateless to guarantee thread safety.

## Data Pipeline
This system acts as a data adapter, feeding entity position information into a more complex spatial data structure. It does not process a pipeline in a traditional sense but rather participates as a data source during the world update tick.

> **Flow during a world tick:**
>
> ECS Scheduler -> **LegacyBeaconSpatialSystem.tick()** -> `super.tick()` -> Iterates matched entities -> Calls `getPosition()` for each entity -> Updates the shared `SpatialResource` (e.g., an octree) with fresh position data.
>
> Other game systems can then safely query the `SpatialResource` for beacon locations.

