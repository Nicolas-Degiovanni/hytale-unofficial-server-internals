---
description: Architectural reference for CollisionResult
---

# CollisionResult

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient

## Definition
```java
// Signature
public class CollisionResult implements BoxBlockIterator.BoxIterationConsumer {
```

## Architecture & Concepts
The CollisionResult class is a stateful container and processor for a single, complex collision query within the server's physics engine. It is not a long-lived system, but rather a short-lived "scratchpad" object that encapsulates the entire context and outcome of one entity's movement or one raycast operation.

Its core architectural function is to decouple the collision *detection* algorithm from the collision *response* logic. It acts as the central aggregation point for all collision-related data generated during a physics tick. It holds the configuration for the query (via the nested CollisionConfig), the low-level geometric evaluators, and the categorized results.

By implementing the BoxIterationConsumer interface, CollisionResult integrates directly with spatial iteration systems. An iterator, such as BoxBlockIterator, traverses the world geometry along a path and feeds candidate blocks to this object's **accept** method for evaluation. This design allows the core physics loop to remain agnostic of the specific collision shapes or rules, which are entirely managed within CollisionResult.

The class categorizes detected intersections into three primary types:
- **Collisions:** Hard intersections that impede movement.
- **Slides:** Tangential contacts with walkable surfaces.
- **Triggers:** Overlaps with non-solid, interactive blocks or fluids (e.g., water, lava, trigger volumes).

## Lifecycle & Ownership
- **Creation:** An instance of CollisionResult is created directly via its public constructor, typically at the beginning of an entity movement calculation or a world query. It is not managed by a dependency injection container or a central registry.

- **Scope:** The object's lifetime is exceptionally brief, confined to the scope of the single method that performs the collision query. It is created, populated, processed, read, and then immediately becomes eligible for garbage collection.

- **Destruction:** There is no explicit destruction method. The object is cleaned up by the Java garbage collector once all references to it are dropped. The presence of the **reset** method suggests that instances *could* be pooled to reduce GC overhead in performance-critical loops, although the public constructor allows for direct, per-use instantiation.

## Internal State & Concurrency
- **State:** CollisionResult is highly mutable. Its primary purpose is to accumulate state. Internal collections such as **blockCollisions**, **blockSlides**, and **blockTriggers** are populated during the query execution. The **process** method then mutates this state further by sorting the collections and calculating derived values like **isSliding**.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed exclusively for synchronous, single-threaded access within the server's main game loop. Its internal collections are non-concurrent, and its methods are not synchronized. Concurrent modification would lead to data corruption, race conditions, and unpredictable physics behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process() | void | O(N log N) | Finalizes the query. Sorts all collected collision data and computes derived state like sliding intervals. **WARNING:** Must be called after iteration and before reading results. |
| reset() | void | O(N) | Clears all internal result lists. Essential for object reuse or pooling. |
| iterateBlocks(collider, pos, direction, length, stop) | void | O(K) | Initiates the collision query by passing this object to a spatial iterator. K is the number of blocks checked. |
| accept(x, y, z) | boolean | O(1) | Callback method for the spatial iterator. Performs the core collision test for a single block. |
| addCollision(evaluator, index) | void | O(1) | Adds a hard collision result to the internal list. |
| addSlide(evaluator, index) | void | O(1) | Adds a slide contact result to the internal list. |
| addTrigger(evaluator, index) | void | O(1) | Adds a trigger overlap result to the internal list. |
| defaultTriggerBlocksProcessing(...) | int | O(T) | Processes all trigger results, manages enter/leave states, and queues game interactions with the InteractionManager. T is the number of triggers. |
| getFirstBlockCollision() | BlockCollisionData | O(1) | Retrieves the earliest hard collision event. Returns null if none exist. |
| forgetFirstBlockCollision() | BlockCollisionData | O(1) | Retrieves and removes the earliest hard collision event from the result set. |

## Integration Patterns

### Standard Usage
A higher-level system, such as an entity physics controller, orchestrates the use of CollisionResult. The standard lifecycle is a sequence of configuration, execution, processing, and consumption.

```java
// 1. Create a new, clean result object for this physics tick.
CollisionResult result = new CollisionResult();

// 2. Configure the query based on the entity's properties.
result.setDefaultPlayerSettings();
result.getConfig().setWorld(entity.getWorld());
// ... other configurations

// 3. Execute the query along the entity's desired movement vector.
//    This populates the internal lists of the result object.
result.iterateBlocks(entity.getCollider(), entity.getPosition(), entity.getVelocity(), tickDuration, true);

// 4. Process the raw results into a sorted, usable format.
result.process();

// 5. Consume the results to modify entity state.
BlockCollisionData firstHit = result.getFirstBlockCollision();
if (firstHit != null) {
    // Adjust entity velocity and position based on firstHit.normal, firstHit.collisionStart, etc.
}

// 6. Process triggers to fire game events (e.g., taking damage in lava).
result.defaultTriggerBlocksProcessing(interactionManager, entity, ...);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use without Reset:** Re-using a CollisionResult instance for a new query without first calling **reset** will contaminate the new query with stale data from the previous one, leading to phantom collisions.

- **Reading Results Before Processing:** Accessing collision data via methods like **getFirstBlockCollision** before calling **process** will yield unsorted and potentially incorrect results. The **process** method is not optional.

- **Concurrent Access:** Modifying or reading a CollisionResult instance from multiple threads will cause critical data corruption and server instability. Confine its use to a single thread.

## Data Pipeline
The CollisionResult acts as a processing node in the entity movement pipeline. Data flows through it in a well-defined sequence.

> Flow:
> Entity Movement Intent -> **CollisionResult** (Configuration) -> BoxBlockIterator (Spatial Query) -> **CollisionResult.accept** (Geometric Test) -> **CollisionResult** (Internal State Population) -> **CollisionResult.process** (Sorting & Finalization) -> Entity Physics Controller (Collision Response) -> **CollisionResult.defaultTriggerBlocksProcessing** (Event Generation) -> InteractionManager (Game Logic Execution)

