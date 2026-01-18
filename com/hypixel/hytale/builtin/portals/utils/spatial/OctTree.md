---
description: Architectural reference for OctTree
---

# OctTree

**Package:** com.hypixel.hytale.builtin.portals.utils.spatial
**Type:** Transient

## Definition
```java
// Signature
public class OctTree<T> {
```

## Architecture & Concepts
The OctTree is a fundamental spatial partitioning data structure used to efficiently store and query objects in a three-dimensional space. It operates by recursively subdividing a cubic volume into eight smaller octants. This hierarchical organization allows for rapid spatial queries, such as proximity or range searches, by enabling the system to prune large, irrelevant sections of the search space.

Within the Hytale engine, this class serves as a low-level utility for any system that requires high-performance spatial lookups. Its primary role is to accelerate the process of finding objects within a given area of interest. This is critical for performance-sensitive features like physics collision detection, AI sensory systems, network culling, and, as its package suggests, portal mechanics. It is not a high-level engine service but rather a foundational building block used to construct such services.

The core principle is to trade memory and a small upfront insertion cost for significantly faster query times. Instead of a linear scan over all objects (O(N)), queries on a well-balanced OctTree approach logarithmic time (O(log N)).

## Lifecycle & Ownership
- **Creation:** An OctTree is instantiated directly via its public constructors, typically by a manager class that needs to index a collection of spatial data. The creator must provide the initial boundary of the space the tree will manage.

- **Scope:** The lifetime of an OctTree instance is explicitly managed by its owner. It is a transient object, designed to be created for a specific task or to represent the state of the world for a limited duration. For example, a system might build a new OctTree from scratch each game tick or whenever the set of indexed objects changes significantly.

- **Destruction:** The object is subject to standard Java garbage collection. There are no native resources or explicit cleanup methods. It is reclaimed by the garbage collector once its owner releases all references to it.

## Internal State & Concurrency
- **State:** The OctTree is a highly mutable and stateful data structure. The internal state consists of a tree of Node objects, which store the points and values added. The `add` operation modifies the tree's structure, potentially triggering recursive subdivision of nodes. The entire structure serves as a spatial cache of object positions.

- **Thread Safety:** This implementation is **not thread-safe**. All internal collections (ObjectArrayList, Object2ObjectOpenHashMap) and state modification logic are unsynchronized. Concurrent calls to `add` will lead to data corruption and an inconsistent tree state. Concurrent modification and querying (e.g., one thread calling `add` while another calls `queryRange`) will result in undefined behavior, likely throwing a ConcurrentModificationException or returning incomplete data.

**WARNING:** All interactions with a single OctTree instance must be externally synchronized or confined to a single thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| OctTree(double inradius) | Constructor | O(1) | Creates a tree with a cubic boundary centered at the origin. |
| OctTree(Box boundary, int nodeCapacity) | Constructor | O(1) | Creates a tree with a specific boundary and node capacity. |
| add(Vector3d pos, T value) | void | O(log N) | Inserts a value at a specific 3D position. Complexity assumes a balanced distribution of points. |
| getAllPoints() | Map<T, Vector3d> | O(N) | Retrieves all points by traversing the entire tree. Use with caution on large datasets. |
| queryRange(Box range) | Map<T, Vector3d> | O(log N + k) | Retrieves all points within the specified Box. *k* is the number of points found. This is the primary query method. |

## Integration Patterns

### Standard Usage
The OctTree is designed to be populated and then queried. A common pattern is to build the tree, perform a series of queries, and then discard it, rebuilding it when the underlying data has changed.

```java
// Example: Find all portals near the player
Box worldBounds = new Box(-1024, 0, -1024, 1024, 256, 1024);
OctTree<Portal> portalIndex = new OctTree<>(worldBounds, 8);

// 1. Populate the tree with all active portals
for (Portal portal : world.getActivePortals()) {
    portalIndex.add(portal.getPosition(), portal);
}

// 2. Define a query range around the player
Box queryArea = Box.centeredCube(player.getPosition(), 32.0);

// 3. Perform the efficient spatial query
Map<Portal, Vector3d> nearbyPortals = portalIndex.queryRange(queryArea);

// 4. Process the results
for (Portal nearbyPortal : nearbyPortals.keySet()) {
    // Render portal effects, check for transition, etc.
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never allow multiple threads to call `add` or `queryRange` on the same OctTree instance without external locking. This will corrupt the tree's internal state.
- **Persistent State:** Do not treat an OctTree as a long-term, persistent store for objects whose positions change frequently. The tree does not update object positions; you must remove and re-add them. For highly dynamic objects, rebuilding the tree periodically is often more efficient than performing many individual updates.
- **Unbounded Growth:** Adding points outside the initial boundary will fail silently. Ensure the root boundary is large enough to contain all potential objects.

## Data Pipeline
The OctTree functions as a spatial indexing stage in a larger data processing flow. It transforms an unstructured list of objects into a structured, spatially-aware hierarchy to enable efficient filtering.

> Flow:
> Raw Object Collection (e.g., List of Portals) -> **OctTree.add()** [Builds Spatial Index] -> Query Volume (e.g., Player's Area of Interest) -> **OctTree.queryRange()** -> Filtered Object Subset -> Game Logic (e.g., Rendering, Physics)

