---
description: Architectural reference for BucketItemPool
---

# BucketItemPool

**Package:** com.hypixel.hytale.common.collection
**Type:** Transient

## Definition
```java
// Signature
public class BucketItemPool<E> {
```

## Architecture & Concepts
The BucketItemPool is a high-performance, specialized object pool designed to mitigate memory allocation overhead and reduce garbage collection pressure in performance-critical systems. Its primary function is to recycle instances of BucketItem, a wrapper class likely used in spatial partitioning or proximity detection algorithms.

This class implements a classic object pool pattern with a LIFO (Last-In, First-Out) retrieval strategy. By reusing objects instead of constantly creating new ones, it prevents memory churn that can lead to noticeable stutters or frame drops in the game engine. The use of the fastutil library's ObjectArrayList underscores its focus on performance, as it avoids the boxing and overhead associated with standard Java collections in certain use cases.

Architecturally, this component serves as a low-level memory management utility, owned and operated by a higher-level system that processes large collections of objects on a per-frame basis, such as an entity culling grid or a physics broad-phase system.

### Lifecycle & Ownership
- **Creation:** An instance of BucketItemPool is created and owned by a manager class that requires frequent, temporary allocation of BucketItem objects. It is not a global singleton and multiple instances can exist, each managing a separate pool.
- **Scope:** The lifecycle of a BucketItemPool is strictly tied to its owner. For example, a pool used by a world's entity manager will persist for the lifetime of that world.
- **Destruction:** The object is reclaimed by the Java garbage collector when its owner is destroyed and all references to it are released. It does not manage any native resources and has no explicit cleanup or shutdown method.

## Internal State & Concurrency
- **State:** The internal state is highly **mutable**, consisting of a list of available BucketItem instances. The `pool` field is continuously modified as items are allocated and deallocated. The pool can grow infinitely if more items are allocated than deallocated over time.
- **Thread Safety:** This class is **not thread-safe**. The underlying collection, ObjectArrayList, is not synchronized. Concurrent access to `allocate` or `deallocate` from multiple threads will lead to race conditions and corrupt the internal state of the pool.

**WARNING:** All interactions with a BucketItemPool instance must be confined to a single thread or protected by external synchronization mechanisms. It is typically expected to be used exclusively on the main game thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| allocate(E reference, double squaredDistance) | BucketItem<E> | O(1) | Retrieves a recycled BucketItem from the pool. If the pool is empty, a new instance is created. |
| deallocate(BucketItem<E>[] entityHolders, int count) | void | O(N) | Returns a batch of N used BucketItem instances to the pool for future reuse, where N is `count`. |

## Integration Patterns

### Standard Usage
The BucketItemPool is designed to be used within a single frame or update cycle. A managing system allocates all necessary items, performs its computations, and then deallocates them in a batch.

```java
// A hypothetical manager class holds a pool instance
BucketItemPool<Entity> pool = new BucketItemPool<>();
BucketItem<Entity>[] activeItems = new BucketItem[1024];

// During an update tick
int itemCount = 0;
for (Entity entity : getNearbyEntities()) {
    double distSq = calculateDistanceSquared(entity);
    activeItems[itemCount++] = pool.allocate(entity, distSq);
}

// ... perform work with activeItems ...

// At the end of the tick, return all items to the pool
pool.deallocate(activeItems, itemCount);
```

### Anti-Patterns (Do NOT do this)
- **Reference Hoarding:** Do not store a reference to a BucketItem returned by `allocate` beyond the immediate scope of its use. Holding onto a pooled object prevents it from being recycled and is functionally equivalent to a memory leak.
- **Cross-Thread Access:** Do not share a single pool instance across multiple threads without implementing external locking. This will cause state corruption and unpredictable behavior.
- **Partial Deallocation:** Failing to deallocate all items that were allocated within a cycle will cause the pool to "leak" objects, diminishing its effectiveness and leading to unnecessary memory growth.

## Data Pipeline
BucketItemPool is not a data processing component but rather a memory management utility. Its flow pertains to the lifecycle of the objects it manages.

> Flow:
> Owning System requires a temporary wrapper -> `allocate()` -> **BucketItemPool** provides a recycled or new `BucketItem` -> Owning System uses the `BucketItem` for computation -> Owning System calls `deallocate()` -> **BucketItemPool** stores the `BucketItem` for the next cycle.

