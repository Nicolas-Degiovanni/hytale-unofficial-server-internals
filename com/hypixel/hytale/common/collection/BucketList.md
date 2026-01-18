---
description: Architectural reference for BucketList
---

# BucketList

**Package:** com.hypixel.hytale.common.collection
**Type:** Transient

## Definition
```java
// Signature
public class BucketList<E> {
```

## Architecture & Concepts

The BucketList is a high-performance, specialized data structure designed for spatial partitioning. Its primary function is to efficiently organize a collection of objects based on their squared distance from a central origin point. This is a common and critical optimization pattern in game development for tasks like entity culling, AI target selection, and proximity queries.

The core concept is to divide a continuous range of distances into a discrete number of "buckets". When an item is added, it is placed into the bucket corresponding to its distance, avoiding the need for a linear scan of all items during a query.

Key architectural features include:

*   **Distance-Based Partitioning:** Items are not stored linearly but are grouped into buckets defined by distance ranges. This allows queries to immediately discard large numbers of irrelevant objects by only inspecting buckets that fall within the query's range.
*   **Squared Distance Optimization:** The entire system operates on squared distances. This is a standard engine optimization that avoids computationally expensive square root operations when comparing distances.
*   **Lookup Table Acceleration:** A `bucketIndices` byte array acts as a direct-address table, mapping an integer squared distance directly to a bucket index. This provides O(1) lookup for determining the correct bucket, trading memory space for significant time savings.
*   **Lazy Sorting:** Within each bucket, items are not sorted upon insertion. Sorting is deferred until a query like `getClosestInRange` is executed on that specific bucket, and only if it has been marked as unsorted. This prevents unnecessary work if a bucket is populated but never queried.
*   **Object Pooling:** The class does not manage the memory of the `BucketItem` wrappers directly. It relies on an externally provided `BucketItemPool`. This decouples the data structure from its memory allocation strategy and integrates it into the engine's broader memory management system to reduce garbage collection pressure.

This class is a foundational component for any system that needs to answer the question "what is near me?" in a performant way.

### Lifecycle & Ownership

*   **Creation:** An instance is created by a consumer system, such as an entity manager or an AI controller. A valid `BucketItemPool` must be provided during construction, establishing a critical dependency. The list is not usable until the `configure` method is called to define the bucket ranges.
*   **Scope:** The lifecycle of a BucketList is typically short and tied to a specific, recurring task. For example, an instance might be created, configured, used to process all entities for a single game tick, and then cleared. It is designed to be reused across frames by calling `clear` rather than being re-instantiated.
*   **Destruction:** The Java garbage collector reclaims the BucketList object when it is no longer referenced. However, manual cleanup is mandatory. The consumer **must** call `clear()` to return all contained `BucketItem` objects to the shared `BucketItemPool`. Failure to do so will result in a severe memory leak, as the pooled objects will never be reclaimed. The `reset()` method can be used for a more forceful cleanup that also discards the bucket configuration.

## Internal State & Concurrency

*   **State:** The BucketList is a highly mutable and stateful container. Its internal arrays (`buckets`, `bucketIndices`) and state flags (`isUnsorted`, `isEmpty` in each bucket) are modified by nearly all public methods. The state is complex, representing both the contained data and the structure's configuration.
*   **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms like locks or atomic operations. All methods assume they have exclusive access to the internal state. Concurrent calls to `add`, `clear`, or `getClosestInRange` from different threads will lead to race conditions, data corruption, and unpredictable exceptions. All access to a BucketList instance must be externally synchronized or confined to a single thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| configure(int[] bucketRanges, ...) | void | O(N) | **Critical Initializer.** Configures bucket boundaries. N is the max squared distance. Throws if ranges are invalid. |
| add(E item, double squaredDistance) | boolean | O(1) | Adds an item to the appropriate bucket. Returns false if the distance is out of bounds. |
| getClosestInRange(...) | E | O(M + K log K) | Primary query method. Finds the closest item matching a filter within a distance range. M is items in relevant buckets; K is the size of the largest unsorted bucket. |
| clear() | void | O(B) | Empties all buckets, returning `BucketItem`s to the pool. The configuration is preserved. B is the number of buckets. |
| reset() | void | O(1) | Clears all items and destroys the bucket configuration. The list must be re-configured before use. |
| getBucket(int index) | Bucket<E> | O(1) | Retrieves a specific bucket. Returns null for invalid indices. |

## Integration Patterns

### Standard Usage

The canonical use case involves configuring the list once, then populating and querying it within a tight loop, such as the main game loop. The list must be cleared at the end of each cycle to release pooled resources.

```java
// Assumes `pool` and `sortBufferProvider` are already initialized
BucketList<Entity> nearbyEntities = new BucketList<>(pool);
int[] ranges = new int[]{16, 64, 256}; // Define distance boundaries
nearbyEntities.configure(ranges);

// --- In the game loop ---
for (Entity entity : allEntities) {
    double distSq = player.getSquaredDistanceTo(entity);
    nearbyEntities.add(entity, distSq);
}

// Find the closest hostile entity between 8 and 32 units away
Predicate<Entity> hostileFilter = e -> e.isHostile();
Entity target = nearbyEntities.getClosestInRange(8, 32, hostileFilter, sortBufferProvider);

// CRITICAL: Clear the list to return items to the pool for the next tick
nearbyEntities.clear();
```

### Anti-Patterns (Do NOT do this)

*   **Forgetting to Clear:** The most common and severe error is failing to call `clear()` after use. This starves the `BucketItemPool` of objects, effectively creating a memory leak that will degrade and eventually crash the application.
*   **Frequent Re-configuration:** The `configure` method is expensive. It allocates a large lookup table and sets up all bucket objects. It should only be called once during initialization, not per-frame or per-query.
*   **Concurrent Access:** Using a single BucketList instance across multiple threads without external locking will corrupt its internal state and is guaranteed to cause difficult-to-diagnose bugs and crashes.
*   **Incorrect Distance Values:** The entire class operates on the assumption that the input distances are *squared*. Providing linear distances will result in incorrect bucketing and query results.

## Data Pipeline

The BucketList acts as a transformation and filtering stage for collections of objects, enabling efficient spatial queries.

> Flow:
> Raw Object Collection -> `add(object, distance²)` -> **BucketList (Partitioning & Caching)** -> `getClosestInRange(query)` -> Lazily Sorted & Filtered Result

