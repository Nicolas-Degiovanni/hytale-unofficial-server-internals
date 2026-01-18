---
description: Architectural reference for EntityCollisionProvider
---

# EntityCollisionProvider

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Stateful Utility

## Definition
```java
// Signature
public class EntityCollisionProvider {
```

## Architecture & Concepts
The EntityCollisionProvider is a server-side computational utility designed to perform discrete, swept collision detection queries against entities. It is not a persistent system that runs in the main game loop; rather, it is a tool invoked on-demand by other systems, such as projectile physics or character movement logic, to determine the first point of impact along a linear path.

Its core function is to answer the question: "If an object with this bounding box moves from point A to point B, what is the first entity it will hit, and where?"

To achieve this efficiently, the provider integrates with the server's spatial partitioning system. It first performs a broad-phase query using `SpatialResource` to gather a list of potential candidates within a given radius, drastically reducing the number of entities to check. It then proceeds to a narrow-phase, performing precise mathematical swept Axis-Aligned Bounding Box (AABB) intersection tests (`CollisionMath.intersectSweptAABBs`) against each candidate. The provider maintains state to track the nearest collision found, ensuring that only the very first impact is reported.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand via its public constructor. It is designed to be a short-lived or pooled object. Systems requiring a collision check will typically create a new instance or acquire one from an object pool for the duration of the operation.
- **Scope:** The internal state of an EntityCollisionProvider is valid only for a single top-level query (e.g., one call to `computeNearest`). Its lifecycle is intrinsically tied to the scope of the calling function.
- **Destruction:** The object is eligible for garbage collection once the reference goes out of scope. The `clear` and `clearRefs` methods are critical for preventing memory leaks if the instance is reused or pooled, as they nullify references to entities and other query-specific data.

## Internal State & Concurrency
- **State:** Highly mutable. An instance of EntityCollisionProvider acts as a state machine for a single collision query. It stores input parameters like position and direction, filter predicates, and the results of the computation in its `contacts` array. The internal arrays (`contacts`, `sortBuffer`) are pre-allocated to a small default size to minimize memory allocation overhead during queries.

- **Thread Safety:** **This class is not thread-safe.** It is fundamentally designed for single-threaded access. Its methods modify internal state without any synchronization. Furthermore, its reliance on `SpatialResource.getThreadLocalReferenceList()` explicitly ties its operation to thread-local data structures.

    **WARNING:** Sharing an instance of EntityCollisionProvider across multiple threads will result in race conditions, state corruption, and unpredictable behavior. Each thread performing collision queries must use its own distinct instance.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeNearest(...) | double | Dependent on spatial density | The primary entry point. Executes the full collision query and returns the distance to the nearest impact. Populates internal state with contact details. |
| getCount() | int | O(1) | Returns the number of contacts found in the last query (typically 0 or 1 for `computeNearest`). |
| getContact(int) | EntityContactData | O(1) | Retrieves the detailed contact data for a given index from the last query's results. |
| clear() | void | O(N) | Resets the provider's state, clearing contact data and count. Essential for object reuse. |

## Integration Patterns

### Standard Usage
The provider is intended to be used as a transient tool. A system acquires an instance, configures and executes a single query, processes the results, and then clears the instance for reuse or allows it to be garbage collected.

```java
// How a developer should normally use this
EntityCollisionProvider collisionProvider = new EntityCollisionProvider(); // Or get from a pool
ComponentAccessor<EntityStore> accessor = ...;
Vector3d startPos = ...;
Vector3d movementVec = ...;
Box myBoundingBox = ...;
Ref<EntityStore> self = ...; // Reference to the entity performing the query

double distance = collisionProvider.computeNearest(
    myBoundingBox,
    startPos,
    movementVec,
    self,
    null, // No other entity to ignore
    accessor
);

if (collisionProvider.getCount() > 0) {
    EntityContactData contact = collisionProvider.getContact(0);
    // Process the collision with contact.getContactEntity()
}

collisionProvider.clear(); // Critical for reuse
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not assume the results from `getContact` are valid after calling `clear` or initiating a new `computeNearest` query. The internal `contacts` array is overwritten.
- **Concurrent Access:** Never call methods on a single `EntityCollisionProvider` instance from multiple threads simultaneously.
- **Forgetting `ignoreSelf`:** Failing to provide a reference to the querying entity itself in the `ignoreSelf` parameter will often result in the entity immediately detecting a collision with itself at the start of its movement vector.
- **Static Instance:** Do not create a single static instance of this provider for global use. Its statefulness makes this pattern extremely dangerous and prone to race conditions.

## Data Pipeline
The flow of data for a `computeNearest` operation is a multi-stage filtering process.

> Flow:
> 1. **Input:** A query is initiated with a starting position, a movement vector, a bounding box, and optional entity filters.
> 2. **Broad-Phase (Spatial Query):** The provider queries the `EntityModule`'s `SpatialResource` for both standard entities and players, collecting a list of all entities within a generous radius of the movement path. This is a coarse but fast filtering step.
> 3. **Narrow-Phase (Iteration & Filtering):** The provider iterates through the list of potential candidates.
>     - It applies basic filters: is the entity collidable? Is it the querying entity itself (`ignoreSelf`)? Does it match the custom `entityFilter` predicate?
> 4. **Narrow-Phase (Precise Test):** For each valid candidate, it executes `CollisionMath.intersectSweptAABBs` to perform a precise mathematical test for collision along the movement vector.
> 5. **State Update:** If a collision is found, it compares the collision distance (`minMax.x`) to the current `nearestCollisionStart`. If the new collision is closer, it overwrites the previous result and stores the new contact data.
> 6. **Output:** The operation returns the distance to the nearest collision. The full details of that collision are stored internally, accessible via `getContact`.

