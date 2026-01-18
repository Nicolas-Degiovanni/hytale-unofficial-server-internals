---
description: Architectural reference for CollisionDataArray
---

# CollisionDataArray

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient

## Definition
```java
// Signature
public class CollisionDataArray<T> {
```

## Architecture & Concepts
The CollisionDataArray is a high-performance, specialized data structure designed for managing temporary collections of objects during performance-critical operations, such as a single pass of the server's collision detection system. It is not a general-purpose list; its design is optimized to minimize object allocation and garbage collection overhead.

The core architecture combines two key patterns:

1.  **Object Pooling:** The class does not create objects from scratch on every allocation. Instead, it participates in an external object pool, represented by the `freeList`. When an object is requested via `alloc`, it is first drawn from this pool. When the array is cleared via `reset`, all its contained objects are returned to the pool for reuse. This dramatically reduces GC pressure in tight loops.

2.  **Sliding Window Access:** The class uses an integer `head` pointer to represent the start of the active data. Operations like `forgetFirst` simply increment this pointer, which is an O(1) operation. This avoids the expensive O(N) cost of removing an element from the beginning of a standard list. The "forgotten" elements are only truly cleaned up and returned to the object pool when `reset` is called, making it highly efficient for sequential processing of the contained data.

This component is fundamental to systems that process large, transient datasets on a per-tick basis. Its design explicitly trades memory (by holding onto "forgotten" objects until a `reset`) for CPU performance.

### Lifecycle & Ownership
-   **Creation:** An instance is created on-demand by a system that needs to perform a short-lived, stateful operation. The creator is responsible for supplying a `Supplier` for new object creation and, critically, a reference to a shared `freeList` which acts as the object pool.
-   **Scope:** The lifecycle of a CollisionDataArray is extremely short, typically confined to the scope of a single method or a single game tick. It is designed to be created, populated, processed, and then immediately reset.
-   **Destruction:** The CollisionDataArray object itself is eligible for garbage collection once it goes out of scope. However, the objects it manages are not. **It is imperative to call the `reset` method** to release these managed objects back to the shared `freeList`. Failure to do so will result in a severe memory leak, as the pooled objects will be orphaned and never reused.

## Internal State & Concurrency
-   **State:** The class is highly mutable and stateful. Its internal `array` and `head` pointer are continuously modified during its operational lifecycle. Its state is only considered valid within the context of a single, sequential operation.
-   **Thread Safety:** **This class is not thread-safe.** All internal operations on the `ObjectArrayList` and `head` field are unsynchronized. It is designed exclusively for use within a single thread, such as the main server thread during a world update tick. Concurrent access from multiple threads will result in `ConcurrentModificationException`, data corruption, and unpredictable behavior.

## API Surface
The public API is designed for a strict create-populate-process-reset workflow.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| alloc() | T | O(1) amortized | Acquires an object from the free list or creates a new one, adding it to the internal array. |
| reset() | void | O(N) | Returns all managed objects to the free list, clearing the internal array. **This must be called.** |
| getFirst() | T | O(1) | Returns the first active element at the current `head` position. Returns null if empty. |
| forgetFirst() | T | O(1) | Advances the `head` pointer, effectively discarding the first element without cleanup. |
| get(int i) | T | O(1) | Accesses the element at the specified index relative to the `head`. |
| remove(int l) | void | O(N) | Removes an element at an index relative to the `head`. This is a slow operation and should be used sparingly. |
| sort(Comparator) | void | O(N log N) | Sorts the active elements in the array. |

## Integration Patterns

### Standard Usage
The class must be used within a `try...finally` block or equivalent structure to guarantee that `reset` is called, preventing memory leaks.

```java
// Assume 'collisionPairPool' is a shared List<CollisionPair>
CollisionDataArray<CollisionPair> pairs = new CollisionDataArray<>(
    CollisionPair::new, 
    CollisionPair::reset, 
    collisionPairPool
);

try {
    // 1. Populate the array
    for (Entity a : entities) {
        for (Entity b : otherEntities) {
            if (broadPhaseCheck(a, b)) {
                CollisionPair p = pairs.alloc();
                p.set(a, b);
            }
        }
    }

    // 2. Process the data
    for (int i = 0; i < pairs.size(); i++) {
        processPair(pairs.get(i));
    }

} finally {
    // 3. CRITICAL: Reset the array to release objects back to the pool
    pairs.reset();
}
```

### Anti-Patterns (Do NOT do this)
-   **Omitting `reset`:** The most critical error. This will permanently remove all allocated objects from the shared pool, causing a memory leak that grows with each operation.
-   **Reusing Across Ticks:** Do not hold a reference to a CollisionDataArray across multiple game ticks without calling `reset`. Its internal state will become invalid and lead to processing stale or incorrect data.
-   **Concurrent Modification:** Do not read from a CollisionDataArray on one thread while another thread is calling `alloc`, `remove`, or `reset`. The internal state is not protected by locks.

## Data Pipeline
CollisionDataArray acts as a temporary, high-performance buffer within a larger data processing pipeline. It does not transform data itself but holds it for subsequent stages.

> Flow:
> Broad-Phase System -> **CollisionDataArray.alloc()** (stores potential collision pairs) -> Sorting & Filtering Stage (`sort`, `remove`) -> Narrow-Phase System (iterates with `get` or `getFirst`) -> **CollisionDataArray.reset()** (releases pairs to global pool)

