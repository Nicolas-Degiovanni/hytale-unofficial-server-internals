---
description: Architectural reference for PlayerSpatialSystem
---

# PlayerSpatialSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** System Component

## Definition
```java
// Signature
public class PlayerSpatialSystem extends SpatialSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerSpatialSystem is a specialized processor within the server's Entity Component System (ECS) framework. Its sole responsibility is to maintain the spatial state of player entities within a shared, high-performance spatial partitioning data structure.

This class acts as a bridge between the logical representation of a player (defined by its components) and the physical world simulation. It operates on a specific subset of entities defined by its static **QUERY**, which targets entities possessing both a **Player** component and a **TransformComponent**.

Crucially, this system does not own the spatial data structure (e.g., a grid or octree) itself. Instead, it is provided a reference to a **SpatialResource** during its construction. The core logic for inserting, updating, and removing entities from this resource is handled by its parent class, **SpatialSystem**. The PlayerSpatialSystem provides two critical pieces of information to its parent:
1.  **What to process:** The `getQuery` method returns the filter for player entities.
2.  **How to get position:** The `getPosition` method provides the specific logic for extracting a Vector3d from a player's **TransformComponent**.

This design pattern promotes separation of concerns: the generic **SpatialSystem** handles the complex algorithms for managing a spatial index, while concrete implementations like **PlayerSpatialSystem** provide the domain-specific knowledge about a particular entity type.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS scheduler or a world module loader during world initialization. It is a managed component and must be constructed with a valid reference to a shared **SpatialResource**.
-   **Scope:** The lifecycle of a PlayerSpatialSystem instance is tightly coupled to the server world it manages. It persists for the entire duration of a world session.
-   **Destruction:** The system is marked for garbage collection when its associated world is unloaded or the server shuts down. No explicit cleanup logic is required within this class.

## Internal State & Concurrency
-   **State:** The PlayerSpatialSystem class is effectively stateless. It does not store or cache any entity data. All state is read directly from entity components during the `tick` and written to the external **SpatialResource** managed by the parent **SpatialSystem**. The **SpatialResource** itself is highly mutable.
-   **Thread Safety:** This system is not thread-safe and is designed to be executed exclusively by its owning ECS scheduler on a single thread, typically the main server thread. Invoking its methods from other threads will lead to race conditions and world state corruption. Concurrency control for the underlying **SpatialResource** must be handled externally if it is to be read by asynchronous tasks (e.g., physics or AI jobs).

## API Surface
The public API is minimal, intended for consumption by the ECS framework, not by general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static entity query used by the ECS scheduler to identify all player entities. |
| tick(dt, systemIndex, store) | void | O(N) | Executes one simulation step. Iterates over all matched player entities to update their position in the spatial index. N is the number of players. |
| getPosition(chunk, index) | Vector3d | O(1) | Extracts the world position from a player entity's **TransformComponent**. Called internally by the parent **SpatialSystem** during the tick. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. It is automatically registered and executed by the server's system scheduler. Other game systems that require spatial information about players should query the shared **SpatialResource**, not this system.

```java
// Example of a different system querying the resource populated by PlayerSpatialSystem

// 1. Obtain the shared spatial resource from the world context
SpatialResource playersSpatialIndex = world.getResource(PLAYER_SPATIAL_RESOURCE_TYPE);

// 2. Perform a query (e.g., find entities near a point)
List<Ref<EntityStore>> nearbyPlayers = new ArrayList<>();
playersSpatialIndex.querySphere(explosionCenter, explosionRadius, nearbyPlayers);

// 3. Process the results
for (Ref<EntityStore> playerRef : nearbyPlayers) {
    // ... apply damage or other effects
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new PlayerSpatialSystem()`. The system must be instantiated by the framework that can provide the mandatory **SpatialResource** dependency. Failure to do so will result in a NullPointerException.
-   **Manual Ticking:** Do not call the `tick` method manually. This bypasses the ECS scheduler, which can break system ordering dependencies, cause state corruption, and lead to unpredictable behavior.
-   **State Storage:** Do not add fields to this class to store per-player state. State should be stored in components. This system is a processor, not a data container.

## Data Pipeline
The system's primary function is to read data from one location (Components) and facilitate its writing to another (the SpatialResource). It is a critical link in the server's spatial awareness data flow.

> Flow:
> **TransformComponent** (Source of Position) -> **PlayerSpatialSystem.getPosition()** (Data Access) -> **SpatialSystem.tick()** (Processing Logic) -> **SpatialResource** (Updated Index) -> Other Systems (Physics, AI, Networking Queries)

