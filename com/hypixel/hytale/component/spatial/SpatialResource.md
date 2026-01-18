---
description: Architectural reference for SpatialResource
---

# SpatialResource

**Package:** com.hypixel.hytale.component.spatial
**Type:** Stateful Resource Component

## Definition
```java
// Signature
public class SpatialResource<T, ECS_TYPE> implements Resource<ECS_TYPE> {
```

## Architecture & Concepts
The SpatialResource class is a fundamental component of the engine's spatial partitioning system. It acts as a high-level manager that binds a generic spatial indexing data structure (e.g., an Octree, a Grid) to the Entity Component System (ECS).

Its primary architectural role is to serve as the authoritative container for entities that exist within the game world's physical space. Game systems, such as Physics, AI, and Rendering, do not interact with raw spatial data structures directly. Instead, they query the world for a SpatialResource, which provides a managed, thread-aware interface to the underlying spatial index.

This class is parameterized with two generic types:
-   **T:** The data type used by the underlying SpatialStructure for partitioning, such as a vector or a bounding box.
-   **ECS_TYPE:** The specific ECS component type being indexed, allowing for multiple distinct spatial indexes within the same world (e.g., one for static collision geometry, another for dynamic entities).

By abstracting the specific implementation of the `SpatialStructure`, the engine can swap indexing strategies (e.g., from a flat grid for a 2D world to an octree for a 3D world) without altering the systems that consume the spatial data.

### Lifecycle & Ownership
The lifecycle of a SpatialResource is strictly controlled by its parent ECS World. It is not intended for direct, unmanaged instantiation.

-   **Creation:** A SpatialResource is instantiated by the ECS World or a dedicated Resource Manager during world initialization. A concrete implementation of a SpatialStructure must be injected via the constructor at this time. This dependency injection pattern defines the indexing strategy for the resource's entire lifetime.
-   **Scope:** The resource's lifetime is directly bound to the lifetime of the parent ECS World. It persists as long as the world exists.
-   **Destruction:** The resource is de-referenced and becomes eligible for garbage collection when its parent ECS World is destroyed. The `clone` method explicitly throws an UnsupportedOperationException, reinforcing that this is a unique, stateful object with a managed identity and cannot be duplicated.

## Internal State & Concurrency
-   **State:** This component is highly mutable. It maintains the canonical state of all registered entities within its associated spatial index. It is the source of truth for spatial relationships, not a temporary cache. Its internal state is composed of the `spatialData` (the collection of entity references) and the `spatialStructure` (the acceleration structure itself).

-   **Thread Safety:** **CRITICAL:** This class is **not thread-safe** for concurrent writes. All modifications to the spatial index (adding, removing, or updating entities) must be synchronized and executed on a single, designated thread, typically the main game logic thread. Failure to adhere to this will result in index corruption, race conditions, and undefined behavior.

    However, the design explicitly facilitates safe concurrent *reads*. The static `getThreadLocalReferenceList` method provides a per-thread, reusable list buffer. This allows multiple worker threads (e.g., for rendering or AI) to perform spatial queries simultaneously without interfering with each other's results or incurring the performance cost of repeated list allocations.

## API Surface
The public API is minimal, exposing the core components for consumption by game systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getThreadLocalReferenceList() | static ObjectList | O(1) | **Warning:** Returns a temporary, thread-local list for query results. The list is cleared upon every call. Do not hold a reference to this list across operations. |
| getSpatialData() | SpatialData | O(1) | Returns the raw container holding the entity references. Direct modification is discouraged. |
| getSpatialStructure() | SpatialStructure | O(1) | Returns the underlying spatial index implementation (e.g., Octree). This is the primary entry point for performing queries. |
| clone() | Resource | O(1) | Throws UnsupportedOperationException. This resource has a unique identity and cannot be cloned. |

## Integration Patterns

### Standard Usage
A system retrieves the SpatialResource from the ECS World's resource registry. It then accesses the underlying SpatialStructure to perform a query, using the thread-local list to receive the results without new memory allocations.

```java
// Example within an ECS System's update method
SpatialResource<Bounds, Renderable> resource = world.getResource(SpatialResource.class);
SpatialStructure<Bounds> index = resource.getSpatialStructure();

// Get a temporary, pre-cleared list for this thread
ObjectList<Ref<Renderable>> results = SpatialResource.getThreadLocalReferenceList();

// Perform the query; the index will populate the 'results' list
index.query(camera.getFrustum(), results);

// Process the results for this frame
for (Ref<Renderable> entityRef : results) {
    // ... render the entity
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new SpatialResource()`. The ECS World is responsible for creating and managing this resource. Unmanaged instances will not be known to any game systems.
-   **Concurrent Modification:** Do not update the `SpatialStructure` (e.g., by adding or moving an entity) from a worker thread while other threads may be reading from it. All writes must be synchronized on the main thread.
-   **List Reference Caching:** Do not store the list returned by `getThreadLocalReferenceList` in a field. It is a volatile, reusable buffer that will be cleared on the next call from the same thread, leading to data corruption and unpredictable behavior.

## Data Pipeline
The primary data flow involves a system initiating a query and processing the resulting set of entity references.

> Flow:
> ECS System (e.g., Renderer) -> World.getResource(SpatialResource.class) -> **SpatialResource**.getSpatialStructure() -> SpatialStructure.query(bounds, threadLocalList) -> ECS System processes entities in list

