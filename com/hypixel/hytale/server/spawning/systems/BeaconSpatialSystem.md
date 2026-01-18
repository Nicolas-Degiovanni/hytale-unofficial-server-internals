---
description: Architectural reference for BeaconSpatialSystem
---

# BeaconSpatialSystem

**Package:** com.hypixel.hytale.server.spawning.systems
**Type:** System Component

## Definition
```java
// Signature
public class BeaconSpatialSystem extends SpatialSystem<EntityStore> {
```

## Architecture & Concepts
The BeaconSpatialSystem is a specialized component within the server's Entity-Component-System (ECS) architecture. Its sole responsibility is to register and track the physical world location of entities that function as spawn beacons.

This class acts as a bridge between the logical definition of a spawn beacon entity and the server's core spatial partitioning system. It achieves this by implementing the abstract SpatialSystem and providing a specific ECS Query. This query targets entities that possess a combination of three components:
1.  **SpawnBeacon:** A marker component identifying the entity as a beacon.
2.  **FloodFillPositionSelector:** A component defining the logic for selecting spawn points around the beacon.
3.  **TransformComponent:** The standard component that stores an entity's position, rotation, and scale in the world.

By defining this precise filter, the BeaconSpatialSystem ensures that only valid spawn beacon entities are indexed by the spatial engine. The heavy lifting of managing the spatial data structure (such as a grid or octree) is handled by the parent SpatialSystem. This class provides the necessary configuration and data-access callbacks, specifically the getPosition method, to allow the parent system to operate on beacon entities.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's central ECS scheduler during world initialization. This system is part of the core engine bootstrap process and is not intended for manual creation.
-   **Scope:** The lifecycle of a BeaconSpatialSystem instance is bound to the lifecycle of the server world it manages. It persists for the entire duration of a game session.
-   **Destruction:** The instance is marked for garbage collection when the corresponding game world is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless. It does not store any per-entity data or maintain its own cache. All state is read directly from the entity components (via the ArchetypeChunk) or managed by the injected SpatialResource and the parent SpatialSystem.
-   **Thread Safety:** This system is not thread-safe and is designed to be operated exclusively by the main server tick thread. The tick method is not re-entrant. Accessing the underlying EntityStore or its components from other threads while this system is executing will result in race conditions, data corruption, and unpredictable server behavior. All interactions with entities managed by this system must be synchronized with the main game loop.

## API Surface
The public API is minimal, as the class is primarily controlled by the ECS framework through its lifecycle methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static ECS query that defines which entities this system processes. |
| tick(dt, systemIndex, store) | void | O(N) | Entry point for the per-frame update, invoked by the ECS scheduler. Delegates to the parent SpatialSystem to update the spatial index for all matching entities. |
| getPosition(archetypeChunk, index) | Vector3d | O(1) | A callback method used by the parent system to retrieve the world position of a specific beacon entity from its TransformComponent. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Interaction is implicit and declarative. To have an entity managed by this system, a developer must create an entity and attach the required set of components.

```java
// Example of creating a compatible entity
// This entity will be automatically discovered and processed by BeaconSpatialSystem.

Entity entity = world.createEntity();
entity.add(new SpawnBeacon());
entity.add(new FloodFillPositionSelector(...));
entity.add(new TransformComponent(new Vector3d(100, 64, 200)));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BeaconSpatialSystem()`. The system requires a `SpatialResource` dependency that is managed and injected by the core engine. Manual instantiation will result in a non-functional system that is disconnected from the game world.
-   **Manual Invocation:** Do not call the `tick` method manually. This bypasses the ECS scheduler's ordering and synchronization guarantees, which can corrupt the spatial index and cause severe desynchronization issues.
-   **Component Omission:** Creating an entity with only a SpawnBeacon component but without a TransformComponent will cause the entity to be ignored by this system, rendering it non-functional as a spatial object.

## Data Pipeline
The system's primary function is to feed entity position data into the server's main spatial index. It does not transform data but rather acts as a selector and accessor for the parent system.

> **Flow: Entity Registration**
>
> 1.  An entity is created with **SpawnBeacon**, **FloodFillPositionSelector**, and **TransformComponent**.
> 2.  The ECS framework evaluates its queries and marks the entity as a target for the **BeaconSpatialSystem**.
> 3.  On the next server tick, the ECS scheduler invokes `BeaconSpatialSystem.tick()`.
> 4.  The parent `SpatialSystem` logic iterates through all matched entities.
> 5.  For each entity, it invokes the local `getPosition()` override to read the `Vector3d` from the entity's **TransformComponent**.
> 6.  The parent `SpatialSystem` uses this position to update its internal spatial data structure (e.g., a sparse grid).
> 7.  The entity is now spatially indexed, allowing other systems to perform efficient location-based queries for spawn beacons.

