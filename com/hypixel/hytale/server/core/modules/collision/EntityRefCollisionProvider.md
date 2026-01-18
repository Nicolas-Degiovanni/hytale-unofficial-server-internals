---
description: Architectural reference for EntityRefCollisionProvider
---

# EntityRefCollisionProvider

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Stateful Utility

## Definition
```java
// Signature
public class EntityRefCollisionProvider {
```

## Architecture & Concepts
The EntityRefCollisionProvider is a high-performance, stateful calculator designed for a single purpose: executing swept-volume collision queries against entities within the world. It is not a persistent service but a transient tool used by other systems, such as the projectile or combat modules, to determine intersections along a movement vector.

Its core design follows a two-phase collision detection strategy to ensure performance:

1.  **Broad-Phase:** The provider first queries a global `SpatialResource`, a spatial partitioning data structure (like an Octree or Grid), to gather a small subset of potentially colliding entities within a given radius. This dramatically reduces the number of entities that require expensive, precise checks. This is handled by the `iterateEntitiesInSphere` method.

2.  **Narrow-Phase:** For each entity identified in the broad-phase, the provider performs a precise mathematical intersection test using the `CollisionMath.intersectSweptAABBs` function. This function determines if a moving Axis-Aligned Bounding Box (the query volume) intersects with the static bounding boxes of the target entities. It can also test against more granular `DetailBox` components for higher-precision hit detection.

This class is heavily optimized to minimize memory allocations during its operation by reusing internal buffers and temporary objects like `tmpResults` and `tmpVector`.

## Lifecycle & Ownership
-   **Creation:** An instance is created directly via its constructor: `new EntityRefCollisionProvider()`. It is typically instantiated on the stack within a method that needs to perform a collision query, for example, inside a projectile's per-tick update logic.

-   **Scope:** The lifetime of an EntityRefCollisionProvider is intended to be extremely short, often confined to a single method call or a single frame update. Its internal state is only valid for the most recent query.

-   **Destruction:** The object is managed by the Java garbage collector. There is no explicit destruction method. The `clear()` method is provided not for destruction, but to reset the internal state, allowing a single instance to be reused for multiple queries. This is a performance pattern to reduce GC pressure in tight loops.

## Internal State & Concurrency
-   **State:** This class is highly **mutable**. It stores the parameters of the last query (position, direction, bounding box) and its results (contacts, count). This state is transient and is invalidated upon the next call to `computeNearest` or `clear`.

-   **Thread Safety:** **This class is not thread-safe.** Its internal fields are modified during every query. Sharing a single instance across multiple threads without external synchronization will result in race conditions and unpredictable behavior. Each thread performing collision checks must use its own dedicated instance of EntityRefCollisionProvider.

## API Surface
The public API is designed for a simple, procedural workflow: configure, execute, read results, and clear.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeNearest(...) | double | O(N) | Executes the full collision query. N is the number of entities returned by the broad-phase spatial query. Returns the distance to the nearest hit or a negative value if no collision occurred. **Warning:** This method mutates the internal state of the object. |
| getCount() | int | O(1) | Returns the number of contacts found by the last `computeNearest` call. |
| getContact(int i) | EntityContactData | O(1) | Retrieves the collision result data at the specified index. Throws an exception if the index is out of bounds. |
| clear() | void | O(C) | Resets the internal state, clearing all contacts. C is the number of contacts from the previous query. This prepares the instance for a new query. |

## Integration Patterns

### Standard Usage
The intended pattern is to create, use, and then either discard the provider or clear it for reuse within a tight loop.

```java
// How a developer should normally use this
EntityRefCollisionProvider collisionProvider = new EntityRefCollisionProvider();
double distance = collisionProvider.computeNearest(
    commandBuffer,
    projectilePosition,
    projectileVelocity,
    projectileBoundingBox,
    searchRadius,
    EntityRefCollisionProvider::defaultEntityFilter,
    projectileEntityRef,
    null
);

if (collisionProvider.getCount() > 0) {
    EntityContactData contact = collisionProvider.getContact(0);
    // Process the collision with contact.getHitEntity()
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Caching:** Do not cache and share a single instance of EntityRefCollisionProvider across different systems or threads. Its state is not protected and will be corrupted.
-   **Stateful Reuse without Clearing:** If reusing an instance in a loop, failing to call `clear()` between queries can lead to stale data from previous calculations being carried over.
-   **Reading Stale Results:** The results from `getContact()` are only valid immediately after a `computeNearest()` call and before the next `clear()` or `computeNearest()`. Do not store the provider itself with the expectation of reading its results later. Extract the `EntityContactData` immediately.

## Data Pipeline
The flow of data through the provider during a `computeNearest` call is a multi-stage filtering process.

> Flow:
> Query Parameters (Position, Direction, BoundingBox) -> `SpatialResource.collect()` (Broad-Phase) -> Candidate Entity List -> `acceptNearestIgnore()` Loop -> `isColliding()` (Narrow-Phase Swept AABB Test) -> Filter & Sort by Distance -> **EntityRefCollisionProvider** (Internal `contacts` array) -> `getContact()` (Result)

