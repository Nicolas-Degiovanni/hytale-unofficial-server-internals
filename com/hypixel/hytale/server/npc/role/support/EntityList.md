---
description: Architectural reference for EntityList
---

# EntityList

**Package:** com.hypixel.hytale.server.npc.role.support
**Type:** Transient

## Definition
```java
// Signature
public class EntityList extends BucketList<Ref<EntityStore>> {
```

## Architecture & Concepts
The EntityList is a high-performance, specialized data structure designed for efficient spatial querying of game entities. It serves as a core optimization within the NPC AI system, enabling behaviors to quickly find nearby entities without scanning the entire world state.

Its primary architectural pattern is **spatial bucketing**. Instead of maintaining a single, flat list of entities, it partitions entities into a series of pre-defined distance-based buckets relative to a central origin point (typically the NPC itself). When entities are added, their squared distance to the origin is calculated, and they are placed into the appropriate bucket.

This design transforms expensive O(N) proximity searches into near-constant time or O(k) operations, where k is the number of entities within the relevant buckets. Queries for entities within a certain range only need to inspect the buckets corresponding to that range, dramatically reducing the iteration count. The class heavily utilizes squared distances to avoid costly square root calculations during population and querying.

EntityList is a stateful, configurable object. It must first be configured with the required search distances for different purposes (e.g., general awareness, collision avoidance) before being populated with entities for a given game tick.

### Lifecycle & Ownership
- **Creation:** An EntityList is instantiated directly by an NPC behavior or system that requires proximity data. It is not a shared service and is typically created on the stack or as a member field of a short-lived AI task object.
- **Scope:** The lifecycle of an EntityList is intentionally brief, usually confined to a single AI update cycle or a specific behavior's execution. It is populated with a snapshot of nearby entities, queried, and then either reset for reuse or discarded.
- **Destruction:** The object is eligible for garbage collection once the AI task that created it completes and no longer holds a reference. The `reset` method should be called if the instance is to be reused in a subsequent tick to clear all internal collections and state.

## Internal State & Concurrency
- **State:** EntityList is highly mutable. Its internal state includes:
    - A collection of distance buckets containing entity references.
    - Configuration parameters such as `maxDistanceSorted`, `maxDistanceUnsorted`, and `maxDistanceAvoidance`.
    - Calculated squared distance thresholds for optimized querying.
    - A `validator` predicate to filter entities upon addition.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms and is designed for use within a single, synchronous game loop thread (e.g., the main server tick).
    - **WARNING:** Concurrent modification from multiple threads will lead to race conditions, inconsistent state, and likely `ArrayIndexOutOfBoundsException` or other undefined behavior. All interactions—configuration, population, and querying—must be performed from the same thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| requireDistanceSorted(value) | int | O(1) | Configures the maximum distance for sorted queries. Must be called before `finalizeConfiguration`. |
| requireDistanceUnsorted(value) | int | O(1) | Configures the maximum distance for unsorted queries. Must be called before `finalizeConfiguration`. |
| requireDistanceAvoidance(value) | int | O(1) | Configures the maximum distance for avoidance queries. Must be called before `finalizeConfiguration`. |
| finalizeConfiguration() | void | O(B log B) | Finalizes the distance configuration, calculates squared thresholds, and prepares the internal buckets. **This method is mandatory** and must be called after all `requireDistance` calls and before any `add` calls. |
| add(ref, parentPosition, commandBuffer) | void | O(1) | Adds an entity to the list, calculating its distance from `parentPosition` and placing it in the correct bucket. |
| reset() | void | O(B) | Clears all entities and resets all configuration state, preparing the instance for reuse. B is the number of buckets. |
| getClosestEntityInRange(...) | Ref | O(k) | Finds the single closest entity matching the given filters and range. May trigger an internal sort on a bucket if it is unsorted. k is the number of entities in the relevant buckets. |
| forEachEntityAvoidance(...) | void | O(k) | Iterates over all entities within the configured avoidance distance. |
| countEntitiesInRange(...) | int | O(k) | Counts the number of entities that match a filter within a given range, up to a specified maximum. |

## Integration Patterns

### Standard Usage
The standard operational flow is a strict, multi-step process. Deviating from this sequence will result in incorrect behavior or runtime errors.

```java
// 1. Create a new instance or get a pooled/reset one.
// The validator predicate ensures only valid entities are ever added.
EntityList nearbyEntities = new EntityList(pool, (ref, accessor) -> ref.isAlive());

// 2. Configure the required search distances for this specific task.
nearbyEntities.requireDistanceSorted(30); // For finding targets
nearbyEntities.requireDistanceAvoidance(8); // For collision avoidance

// 3. Finalize the configuration. This is a mandatory step.
nearbyEntities.finalizeConfiguration();

// 4. Populate the list with entities from a broad-phase query.
// This is typically done once per AI tick.
for (Ref<EntityStore> entityRef : world.getEntitiesInSphere(npc.getPosition(), nearbyEntities.getSearchRadius())) {
    nearbyEntities.add(entityRef, npc.getPosition(), commandBuffer);
}

// 5. Perform one or more specific, optimized queries.
Ref<EntityStore> closestTarget = nearbyEntities.getClosestEntityInRange(0, 30, myTargetFilter, componentAccessor);
nearbyEntities.forEachEntityAvoidance(ignored, (ref, npc, cmd) -> {
    // Logic to steer away from 'ref'
}, npc, commandBuffer);

// 6. Reset the list if it will be reused.
nearbyEntities.reset();
```

### Anti-Patterns (Do NOT do this)
- **Forgetting Finalization:** Failure to call `finalizeConfiguration` before `add` will result in an unconfigured object. Queries will fail or return empty results because the internal bucket ranges have not been calculated.
- **Re-ordering Operations:** Calling `requireDistance` methods after `finalizeConfiguration` has no effect. Calling `add` before `finalizeConfiguration` is an error.
- **State Leakage:** Reusing an EntityList instance across multiple ticks or for different AI behaviors without calling `reset` first. This will cause data from the previous state to contaminate the new query, leading to NPCs reacting to phantom or out-of-date entities.
- **Concurrent Access:** Accessing a single EntityList instance from multiple threads without external locking. This will corrupt the internal state of the buckets.

## Data Pipeline
EntityList acts as a filter and accelerator for entity data. It does not transform the data itself but rather organizes it for rapid access.

> Flow:
> Broad-Phase World Query -> **EntityList::add** (Filters via validator, calculates distance, places in bucket) -> Internal Distance Buckets -> **EntityList Query Method** (e.g., `getClosestEntityInRange`) -> Filtered & Sorted Entity Reference(s) -> NPC Behavior Logic

