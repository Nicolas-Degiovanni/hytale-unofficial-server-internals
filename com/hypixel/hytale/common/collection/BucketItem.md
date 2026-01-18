---
description: Architectural reference for BucketItem
---

# BucketItem

**Package:** com.hypixel.hytale.common.collection
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class BucketItem<E> {
```

## Architecture & Concepts

The BucketItem class is a highly specialized, performance-oriented data structure. It is not intended for general-purpose use. Its primary role is to act as a temporary container during spatial query operations, such as nearest-neighbor searches or radius searches within collections like a Spatial Grid or k-d tree.

It encapsulates two fundamental pieces of information:
1.  A reference to an object or entity, `item`.
2.  The pre-calculated squared distance to that item from a query's origin point, `squaredDistance`.

The use of squared distance is a critical performance optimization. By comparing squared distances, the system avoids the computationally expensive square root operation that would be required to calculate the true Euclidean distance. This pattern is common in game engines and real-time simulations where thousands of distance checks may occur per frame.

BucketItem is designed to be lightweight and mutable, often participating in an object pooling system to minimize garbage collection overhead during intensive query loops.

### Lifecycle & Ownership
-   **Creation:** BucketItem instances are typically created or retrieved from an object pool at the beginning of a spatial query. They are not managed by a central service or registry.
-   **Scope:** The lifecycle of a BucketItem is extremely short. It is scoped to the execution of a single query method. It exists only to hold intermediate results before they are processed or discarded.
-   **Destruction:** If not part of an object pool, the instance is abandoned after the query completes and becomes eligible for garbage collection. In a pooling scenario, it is "released" back to the pool for immediate reuse.

## Internal State & Concurrency
-   **State:** The state of a BucketItem is fully mutable via its public fields and the `set` method. It is designed to be reused and repopulated repeatedly. It holds no persistent state and does not cache data beyond the scope of a single `set` operation.
-   **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access within a tight loop. Concurrent modification of a BucketItem instance will lead to race conditions and undefined behavior. It must not be shared across threads.

## API Surface

The public contract consists of two public fields for direct, high-performance access and one fluent mutator method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| item | E | O(1) | Public field holding a reference to the contained object. |
| squaredDistance | double | O(1) | Public field holding the pre-calculated squared distance. |
| set(E, double) | BucketItem<E> | O(1) | Re-initializes the object with a new item and its squared distance. Returns a reference to itself for chaining or fluent usage. |

## Integration Patterns

### Standard Usage

The intended use is within a spatial search algorithm. The caller retrieves an instance, populates it with a candidate object and its calculated distance, and adds it to a temporary collection of results.

```java
// Assume 'results' is a List<BucketItem<Entity>> and 'pool' is an object pool
// This code would exist inside a spatial query implementation

for (Entity candidate : potentialCandidates) {
    double sqrDist = calculateSquaredDistance(queryOrigin, candidate.getPosition());
    if (sqrDist < queryRadiusSquared) {
        BucketItem<Entity> resultItem = pool.acquire(); // Or new BucketItem<>();
        resultItem.set(candidate, sqrDist);
        results.add(resultItem);
    }
}

// The 'results' list can now be sorted by squaredDistance
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not store BucketItem instances in long-lived collections or as members of persistent game objects. They are transient and may be recycled by an object pool, leading to stale or incorrect data.
-   **State Assumption:** Do not read from a BucketItem instance without first calling the `set` method to guarantee its state is valid for the current context. An instance retrieved from a pool will contain data from a previous, unrelated operation.
-   **Concurrent Access:** Do not pass a BucketItem to another thread or access it from a parallel task. All access must be confined to the thread that initiated the query.

## Data Pipeline

BucketItem acts as a data carrier within a larger spatial query pipeline. It does not initiate or terminate a flow but is a critical intermediate component.

> Flow:
> Spatial Query Initiated -> Candidate Object Iteration -> Distance Calculation -> **BucketItem.set(object, distance)** -> Addition to Result Set -> Result Set Sorting/Filtering

