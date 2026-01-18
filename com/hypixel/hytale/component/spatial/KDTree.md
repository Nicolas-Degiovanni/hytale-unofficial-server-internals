---
description: Architectural reference for KDTree
---

# KDTree

**Package:** com.hypixel.hytale.component.spatial
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class KDTree<T> implements SpatialStructure<T> {
```

## Architecture & Concepts

The KDTree is a high-performance, memory-optimized spatial partitioning data structure. It organizes a set of 3D points to enable extremely fast spatial queries, such as finding the nearest neighbor or collecting all points within a given radius. This component is fundamental for systems that require efficient proximity detection, including physics broad-phasing, AI targeting, culling, and particle systems.

The core architectural choice of this implementation is its **bulk-loading design**. Unlike data structures that support incremental insertions, the KDTree is built or rebuilt in a single, highly optimized operation via the `rebuild` method. This approach constructs a well-balanced tree from a complete dataset, prioritizing fast and predictable query performance over the flexibility of dynamic insertions. This makes it ideal for scenarios where spatial data is updated in discrete ticks or frames, rather than continuously.

A key optimization is the aggressive use of **internal object pools** for both tree nodes and the data lists they contain. During a `rebuild`, nodes and lists are recycled from these pools, drastically reducing memory allocation overhead and subsequent garbage collection pauses. This is critical for maintaining stable performance in a real-time game engine environment.

The partitioning algorithm cycles through the X, Z, and Y axes at each level of the tree's depth, creating axis-aligned splitting planes to divide the dataset. The initial sorting of input data by Morton code helps preserve spatial locality, which contributes to the construction of a more efficient and balanced tree.

## Lifecycle & Ownership

-   **Creation:** An instance of KDTree is created directly by a consumer system using its public constructor: `new KDTree(predicate)`. The owner is typically a higher-level manager that oversees a collection of spatial objects, such as an EntityManager or a WorldPartition.

-   **Scope:** The object's lifetime is bound to its owner. It is a transient, stateful object, not a global service. It is designed to be populated via `rebuild`, queried extensively, and then either rebuilt with new data on a subsequent game tick or discarded entirely when its owner is destroyed.

-   **Destruction:** The KDTree instance and its internal pools become eligible for garbage collection once all external references are released. There is no explicit `destroy` or `close` method.

## Internal State & Concurrency

-   **State:** The KDTree is highly stateful. Its primary state consists of the root node and the interconnected graph of child nodes, which represents the spatial hierarchy. This state is **aggressively mutated** during the `rebuild` operation. Between rebuilds, the tree structure is effectively immutable from a public API perspective. The internal object pools are also part of its mutable state.

-   **Thread Safety:** **This class is not thread-safe.** All methods mutate or read internal state without any synchronization mechanisms. Concurrent calls to `rebuild` and any query method (e.g., `collect`, `closest`) will result in undefined behavior, including `NullPointerException`, `IndexOutOfBoundsException`, and silent data corruption.

    **WARNING:** All interactions with a KDTree instance, including its construction, rebuilds, and queries, MUST be confined to a single thread or protected by external synchronization primitives managed by the owner.

## API Surface

The public API is focused on two distinct operations: populating the structure and querying it.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| rebuild(SpatialData) | void | O(N log N) | Discards the entire existing tree and builds a new, balanced tree from the provided spatial data. This is an expensive, blocking operation. |
| closest(Vector3d) | T | O(log N) | Performs a nearest-neighbor search. Returns the single closest data point to the query vector. Complexity can degrade to O(N) in worst-case scenarios. |
| collect(center, radius, results) | void | O(log N) | Collects all data points within a spherical volume into the provided results list. |
| collectCylinder(...) | void | O(log N) | Collects all data points within a cylindrical volume into the provided results list. |
| collectBox(min, max, results) | void | O(log N) | Collects all data points within an axis-aligned bounding box into the provided results list. |
| ordered(center, radius, results) | void | O(N log N) | Collects all data points within a sphere and sorts them by distance from the center. This is more expensive than a standard `collect` call due to the internal collection and sort. |

## Integration Patterns

### Standard Usage

The intended pattern is to batch spatial data, rebuild the tree once per simulation step, and then perform all necessary queries for that step.

```java
// Example: In a world update tick
SpatialData<Entity> entityData = getEntitySpatialData();
KDTree<Entity> worldTree = new KDTree<>(entity -> entity.isActive());

// 1. Rebuild the tree once with all current entity data
worldTree.rebuild(entityData);

// 2. Perform multiple, fast queries against the static tree
List<Entity> nearbyEntities = new ArrayList<>();
worldTree.collect(player.getPosition(), 10.0, nearbyEntities);

Entity closestEnemy = worldTree.closest(player.getPosition());
```

### Anti-Patterns (Do NOT do this)

-   **Incremental Rebuilds:** Do not call `rebuild` multiple times within a single game tick for minor data changes. The cost of rebuilding outweighs the benefit. The structure is designed for bulk updates, not single insertions.

-   **Concurrent Access:** Do not share a KDTree instance across threads without explicit, external locking. For example, do not allow a physics thread to query the tree while the main game thread is rebuilding it. This will lead to crashes and data corruption.

-   **Ignoring the Collection Filter:** The `Predicate` provided in the constructor is a powerful optimization. Failing to use it effectively forces the caller to perform a second filtering pass on the query results, which is less efficient.

## Data Pipeline

The KDTree acts as a processor that transforms an unordered collection of spatial data into a structured, queryable hierarchy.

> Flow:
> Unordered `SpatialData` object -> `rebuild()` (Morton Sort & Tree Construction) -> **KDTree Internal Node Graph** -> Query Method (e.g., `collect`) -> Traversal & Filtering -> Populated `List<T>` of results

