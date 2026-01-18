---
description: Architectural reference for the SpatialStructure interface, the core contract for spatial partitioning and querying in the engine.
---

# SpatialStructure

**Package:** com.hypixel.hytale.component.spatial
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface SpatialStructure<T> {
```

## Architecture & Concepts
The SpatialStructure interface defines a formal contract for spatial partitioning data structures. Its primary role is to abstract the underlying implementation (e.g., Octree, k-d tree, Grid) from the systems that need to perform efficient spatial queries. This is a fundamental component for performance-critical systems that operate on object proximity.

This interface is the architectural lynchpin for systems such as:
- **Physics:** For broad-phase collision detection to quickly find potentially colliding pairs.
- **AI:** For sensory systems, allowing entities to find nearby targets, allies, or obstacles.
- **Rendering:** For view frustum culling, determining which objects are visible to the camera.
- **Networking:** For interest management, deciding which entity updates need to be sent to which clients based on proximity.

By depending on this interface rather than a concrete implementation, engine systems are decoupled from the specific data structure, allowing for different strategies to be employed based on the use case (e.g., a dense grid for uniform entities vs. a sparse octree for a game world).

## Lifecycle & Ownership
As an interface, SpatialStructure does not have a lifecycle of its own. The lifecycle and ownership semantics apply to its concrete implementations.

- **Creation:** Implementations are instantiated by high-level container objects or systems that manage a collection of spatial entities. For example, a World object would create and own a SpatialStructure implementation to manage all entities within it.
- **Scope:** The lifetime of a SpatialStructure implementation is strictly tied to its owner. A structure for a world chunk persists only as long as that chunk is loaded in memory. A structure for a player's immediate vicinity might be short-lived and rebuilt frequently.
- **Destruction:** The object is marked for garbage collection when its owner is destroyed. There are no explicit `destroy` or `close` methods on the interface, as cleanup is expected to be handled by the JVM garbage collector.

## Internal State & Concurrency
- **State:** Any implementation of this interface is inherently stateful and highly mutable. It maintains a complex internal representation of the spatial relationships between all objects provided to it. The `rebuild` method is a destructive operation that completely replaces the internal state.

- **Thread Safety:** **WARNING:** This interface makes no guarantees about thread safety. Implementations should be assumed to be **not thread-safe**. The internal data structures (trees, grids) are complex and not designed for concurrent modification.
    - All write operations (e.g., `rebuild`) and read operations (e.g., `collect`, `closest`) must be externally synchronized.
    - The standard engine pattern is to confine all interactions with a given SpatialStructure instance to a single thread, typically the main game loop or a dedicated physics thread. Unsynchronized access from multiple threads will lead to data corruption, incorrect query results, and non-deterministic crashes.

## API Surface
The public API is designed for two primary purposes: rebuilding the entire structure from a new set of data and performing various spatial queries against the existing data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| rebuild(data) | void | O(N) to O(N log N) | **Heavy Operation.** Atomically replaces the entire internal structure with new data. This is the primary method for updating the state. |
| closest(point) | T | O(log N) | Finds the single closest element to a given point. Returns null if the structure is empty. |
| collect(point, radius, out) | void | O(log N + K) | Collects all elements within a spherical radius of a point. K is the number of elements found. |
| collectCylinder(point, radius, height, out) | void | O(log N + K) | Collects all elements within a cylindrical volume. |
| collectBox(min, max, out) | void | O(log N + K) | Collects all elements within an Axis-Aligned Bounding Box (AABB). This is often the most performant query type. |
| ordered(point, radius, out) | void | > O(log N + K) | Collects elements within a radius and guarantees the output list is sorted by distance. Incurs a performance penalty for sorting. |

## Integration Patterns

### Standard Usage
The intended pattern is for a manager system to own the structure, rebuild it with fresh data when entities have moved, and then make it available for querying by other systems within the same update tick.

```java
// Example: A World system updating its spatial index
// Assume 'world' owns the SpatialStructure and a list of entities

// 1. Collect current entity positions into a SpatialData object
SpatialData<Entity> currentEntityData = buildSpatialDataFromEntities(world.getEntities());

// 2. Rebuild the structure (a heavy, single-threaded operation)
SpatialStructure<Entity> spatialIndex = world.getSpatialIndex();
spatialIndex.rebuild(currentEntityData);

// 3. Other systems can now safely query the updated structure
List<Entity> nearbyEntities = new ArrayList<>();
Vector3d playerPosition = player.getPosition();
spatialIndex.collect(playerPosition, 10.0, nearbyEntities);

// ... process nearbyEntities for AI, effects, etc.
```

### Anti-Patterns (Do NOT do this)
- **Per-Entity Updates:** Do not attempt to add or remove single elements. The interface is designed for bulk updates via `rebuild`. Attempting to wrap it for incremental updates will result in poor performance.
- **Concurrent Access:** Never allow one thread to call `rebuild` while another thread is calling a `collect` method on the same instance. This is a severe race condition that must be prevented with external locking or a single-threaded access model.
- **Frequent Rebuilding of Static Geometry:** Do not mix static and dynamic objects in a single structure that is rebuilt every frame. This is highly inefficient. The correct pattern is to maintain separate SpatialStructure instances for static and dynamic objects, rebuilding each at an appropriate frequency.

## Data Pipeline
The SpatialStructure serves as a transformation point, converting a raw list of positional data into an optimized, queryable index.

> Flow:
> Raw Entity Positions -> `SpatialData` Object -> **`SpatialStructure.rebuild()`** -> Optimized Internal Tree/Grid -> Query (e.g., `collectBox`) -> Filtered List of Proximate Entities -> Consumer System (Physics, AI)

