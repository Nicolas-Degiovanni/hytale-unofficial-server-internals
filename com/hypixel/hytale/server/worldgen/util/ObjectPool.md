---
description: Architectural reference for ObjectPool
---

# ObjectPool

**Package:** com.hypixel.hytale.server.worldgen.util
**Type:** Transient

## Definition
```java
// Signature
public class ObjectPool<T extends Function<T, T>> implements Function<T, T> {
```

## Architecture & Concepts
The ObjectPool is a high-performance, thread-safe utility designed to mitigate garbage collector pressure in performance-critical systems, primarily the server-side world generation pipeline. By recycling and reusing expensive-to-create objects, it significantly reduces memory allocation churn during intensive, repetitive tasks like noise evaluation or biome data calculation.

The most critical architectural feature is its generic constraint: **T extends Function<T, T>**. This is not a generic container; it is a pool of specialized, state-transforming objects. This constraint mandates that any object stored within the pool must be a function capable of taking an instance of its own type and returning an instance of its own type.

In practice, this pattern is used to efficiently reset the state of a pooled object. The `apply` method on a pooled object is typically implemented to copy the state from a template or default object, preparing it for reuse without needing to run a complex constructor. This makes the `acquire` and `reset` cycle extremely fast.

This component is fundamental to achieving scalable and performant world generation across multiple threads.

## Lifecycle & Ownership
-   **Creation:** An ObjectPool is instantiated directly by a service or manager that requires object recycling. It is not a globally managed singleton. For example, a specific `ChunkGenerator` may create and own several pools for different types of data structures it uses.
-   **Scope:** The pool's lifetime is strictly bound to its owner. It persists as long as the owning object is in scope and is discarded along with it.
-   **Destruction:** There is no explicit `destroy` or `shutdown` method. The pool and its contained objects become eligible for garbage collection once the owning object is no longer referenced.

**WARNING:** Failure to release the reference to the owning object will result in a memory leak of the pool and all objects currently cached within it.

## Internal State & Concurrency
-   **State:** The internal state is mutable, consisting of an `ArrayBlockingQueue` that stores the available objects. The number of items in the queue fluctuates as objects are acquired and recycled.
-   **Thread Safety:** This class is **thread-safe**. The use of `java.util.concurrent.BlockingQueue` ensures that the `acquire` and `recycle` operations are atomic and can be safely called from multiple worker threads. This is essential for its use in parallelized world generation tasks.

**WARNING:** While individual operations are thread-safe, combinations are not. For example, checking `size()` and then calling `acquire()` based on that result is not an atomic operation and can lead to race conditions in highly concurrent scenarios.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| acquire() | T | O(1) | Retrieves an object from the pool. If the pool is empty, a new object is created via the provided Supplier. |
| recycle(K v) | void | O(1) | Returns an object to the pool for reuse. This is a non-blocking offer; if the pool is at capacity, the object is discarded. |
| apply(T cachedKey) | T | O(1) | Acquires an object and immediately invokes its `apply` method with the given key. This is a convenience for the acquire-reset-use pattern. |

## Integration Patterns

### Standard Usage
The canonical usage pattern involves a `try-finally` block to guarantee that an acquired object is always recycled, even if an exception occurs.

```java
// Assumes 'vectorPool' is an initialized ObjectPool<Vector3>
ObjectPool<Vector3> vectorPool = ...;

Vector3 myVector = vectorPool.acquire();
try {
    // Perform work with myVector
    myVector.set(10, 20, 30);
    world.updatePosition(myVector);
} finally {
    vectorPool.recycle(myVector);
}
```

### Anti-Patterns (Do NOT do this)
-   **Pool Leaking:** Do not acquire an object and fail to recycle it. This defeats the purpose of the pool, leading to continuous object creation and eventual GC pressure. This is the most common and severe misuse of the component.
-   **Use After Recycle:** Do not retain a reference to an object after it has been passed to `recycle`. The same object instance may be acquired by another thread, leading to unpredictable state corruption and severe, hard-to-debug concurrency bugs.
-   **Double Recycle:** Do not recycle the same object instance twice. While `ArrayBlockingQueue` may prevent adding the same instance if the pool is full, this pattern indicates a critical logic flaw and can lead to the same object being acquired and used by multiple threads simultaneously.

## Data Pipeline
The ObjectPool does not process data itself; it acts as a managed repository for stateful worker objects. Its flow governs the lifecycle of a reusable object, not a stream of data.

> Flow:
> WorldGen Thread requests object -> **ObjectPool.acquire()** -> Pool provides existing or new instance -> Thread uses object for calculation -> Thread returns object via **ObjectPool.recycle()** -> Pool stores object for next request
---

