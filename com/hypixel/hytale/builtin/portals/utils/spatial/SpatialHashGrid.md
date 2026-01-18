---
description: Architectural reference for SpatialHashGrid
---

# SpatialHashGrid<T>

**Package:** com.hypixel.hytale.builtin.portals.utils.spatial
**Type:** Transient

## Definition
```java
// Signature
public class SpatialHashGrid<T> {
```

## Architecture & Concepts
The SpatialHashGrid is a fundamental spatial partitioning data structure used to accelerate proximity and range-based queries in a 3D world. It is a performance-critical component for systems that need to efficiently find objects near a specific point or within a given radius, such as AI, physics, networking, and culling.

The core architectural principle is the discretization of continuous 3D space into a grid of fixed-size cubic cells. Each object is stored in a "bucket" corresponding to the cell its position occupies. When a query is performed, only the cells that intersect the query volume need to be inspected, dramatically reducing the number of objects to check compared to a naive linear scan of all objects in the world.

Internally, the system maintains two primary data structures:
1.  **The Grid Map:** An `Object2ObjectOpenHashMap` mapping a cell's integer coordinates (Vector3i) to a list of all objects within that cell. This is the primary structure for spatial lookups.
2.  **The Index Map:** An `Object2ObjectOpenHashMap` mapping an object instance (T) directly to its corresponding internal Entry record. This secondary index provides O(1) average-time access for update and removal operations (e.g., `move`, `remove`), which would otherwise require a costly search.

The choice of `fastutil` collections indicates a design priority for high performance and low memory overhead, at the expense of thread safety.

## Lifecycle & Ownership
- **Creation:** An instance is created via its public constructor: `new SpatialHashGrid(double cellSize)`. It is typically instantiated and owned by a higher-level manager system, such as an EntityManager or a World instance, which is responsible for a discrete collection of spatial objects.
- **Scope:** The lifetime of a SpatialHashGrid is directly coupled to the lifetime of the object collection it manages. For example, a grid might exist for the duration a game zone is loaded and active.
- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit `close` methods. It becomes eligible for collection once its owning manager is destroyed and all references to it are released.

## Internal State & Concurrency
- **State:** The SpatialHashGrid is a stateful and highly mutable component. Its internal maps are continuously modified by `add`, `remove`, and `move` operations. It caches the position and cell location of every object it contains. The accuracy of its queries is entirely dependent on the freshness of this state.

- **Thread Safety:** **This class is not thread-safe.** Its internal data structures (`Object2ObjectOpenHashMap`, `ObjectArrayList`) are not designed for concurrent access. All method calls that modify the grid (`add`, `remove`, `move`, `removeIf`) or read from it (`queryRange`, `findClosest`) must be externally synchronized.

    **WARNING:** Unsynchronized access from multiple threads will lead to `ConcurrentModificationException`, data corruption, or non-deterministic query results. The standard engine pattern is to confine all interactions with a given SpatialHashGrid instance to a single thread, typically the main game loop or world update thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(pos, value) | void | O(1) avg | Adds an object to the grid at the specified position. |
| remove(value) | boolean | O(1) avg | Removes an object from the grid. Fails silently if the object does not exist. |
| move(value, newPos) | void | O(1) avg | Updates the position of an existing object. Handles cell transitions internally. |
| queryRange(center, radius) | Map<T, Vector3d> | O(k) | Returns all objects within the specified radius. Complexity is proportional to the number of entities **k** in the queried cells, not the total number of entities in the grid. |
| findClosest(center, radius) | T | O(k) | Finds the single closest object within the search radius. Returns null if none are found. |
| hasAnyWithin(center, radius) | boolean | O(k) | Performs an early-exit check to determine if any object exists within the radius. More performant than checking if `queryRange` is empty. |

## Integration Patterns

### Standard Usage
The SpatialHashGrid should be owned by a manager class that oversees a collection of entities. The manager is responsible for keeping the grid synchronized with the state of its entities.

```java
// Example: An EntitySystem managing a SpatialHashGrid
public class WorldEntityManager {
    private final SpatialHashGrid<Entity> spatialGrid;
    private final List<Entity> allEntities;

    public WorldEntityManager() {
        // Cell size is a critical tuning parameter.
        double optimalCellSize = 16.0;
        this.spatialGrid = new SpatialHashGrid<>(optimalCellSize);
        this.allEntities = new ArrayList<>();
    }

    public void addEntity(Entity entity) {
        allEntities.add(entity);
        spatialGrid.add(entity.getPosition(), entity);
    }

    // This must be called every tick for entities that have moved.
    public void updateEntityPosition(Entity entity, Vector3d newPosition) {
        entity.setPosition(newPosition);
        spatialGrid.move(entity, newPosition);
    }

    public List<Entity> findNearbyEntities(Vector3d location, double radius) {
        // Use the grid to efficiently query for neighbors.
        Map<Entity, Vector3d> results = spatialGrid.queryRange(location, radius);
        return new ArrayList<>(results.keySet());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Desynchronization:** Do not modify an object's position without immediately calling `move`. Failure to do so will cause the grid to contain stale data, leading to incorrect query results. The grid will believe the object is in its old cell.
- **Concurrent Access:** Do not access a single grid instance from multiple threads without implementing robust external locking. This will corrupt the grid's internal state.
- **Improper Cell Sizing:**
    - **Too small:** A `cellSize` that is too small will create an excessive number of grid cells, increasing memory overhead and iteration time over sparse cells.
    - **Too large:** A `cellSize` that is too large will place too many objects into a single cell, causing query performance to degrade and approach a linear scan (O(N)). The ideal cell size is typically related to the average query radius or average entity size.

## Data Pipeline
The primary data flow involves spatial queries. The grid acts as an indexed data store, not a processing pipeline.

> Flow for a Proximity Query:
> Query Request (Center Point, Radius) -> **SpatialHashGrid.queryRange()** -> Calculate Bounding Box of Cells -> Iterate over affected Grid Cells -> For each Cell, iterate over contained Entries -> Perform precise distance check -> Aggregate results -> Return Filtered Map of Objects

