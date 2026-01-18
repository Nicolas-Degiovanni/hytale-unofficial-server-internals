---
description: Architectural reference for Vector4dBufferList
---

# Vector4dBufferList

**Package:** com.hypixel.hytale.math.hitdetection
**Type:** Data Structure / Object Pool

## Definition
```java
// Signature
public class Vector4dBufferList {
```

## Architecture & Concepts
The Vector4dBufferList is a high-performance, fixed-capacity object pool designed to eliminate runtime memory allocations within performance-critical systems like physics and rendering. Its core purpose is to combat garbage collection pressure by pre-allocating a pool of Vector4d objects on initialization.

Instead of creating new Vector4d instances on the heap during each frame or physics tick, systems can request a pre-existing, "recycled" object from this buffer. The "list" is not a conventional list; it does not grow or shrink. "Adding" an element is an O(1) operation that simply returns the next available object from the internal array. Likewise, "clearing" the list is an O(1) operation that resets an internal counter, making the entire pool available for reuse in the next computation cycle.

This component is fundamental to the engine's performance strategy in mathematically intensive domains. It represents a trade-off, consuming more memory upfront to guarantee smoother, stutter-free execution by avoiding the performance cost of frequent, small object allocations.

### Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by a system that requires a temporary scratchpad for vector calculations. It is common to see this created as a member field of a system (e.g., CollisionSystem) or as a local variable within a complex update method.
-   **Scope:** The intended scope is short-lived, typically for the duration of a single method execution or a single frame update. It is designed to be filled, processed, and cleared rapidly.
-   **Destruction:** The object and its entire pre-allocated buffer become eligible for garbage collection when the owning reference goes out of scope. The `clear` method does **not** deallocate memory; it only resets the internal usage counter for immediate reuse of the buffer.

## Internal State & Concurrency
-   **State:** Highly mutable. The internal state consists of the `vectors` array, whose elements are mutable, and the `size` integer, which tracks the number of "active" vectors in the pool. The state is expected to change rapidly during a computation.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for high-speed, single-threaded access. Concurrent calls to `next` from different threads will result in a race condition, where multiple threads receive a reference to the same Vector4d object, leading to data corruption. All access must be externally synchronized, though the performance implications of doing so would negate the purpose of the class. It should be confined to a single thread of execution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Vector4dBufferList(int size) | Constructor | O(N) | Creates the buffer, pre-allocating N Vector4d objects. |
| next() | Vector4d | O(1) | Returns the next available, pre-allocated Vector4d from the pool. |
| clear() | void | O(1) | Resets the list for reuse by setting the internal size counter to zero. |
| size() | int | O(1) | Returns the current number of *used* vectors in the buffer. |
| get(int i) | Vector4d | O(1) | Retrieves the vector at the specified index. |

## Integration Patterns

### Standard Usage
This pattern is common inside a system's update loop. The buffer is created with a sufficient capacity, populated during a calculation, read from, and then cleared for the next frame.

```java
// A system holds a buffer, sized for the worst-case scenario.
private final Vector4dBufferList intersectionPoints = new Vector4dBufferList(256);

public void findIntersections(World world) {
    // 1. Clear the buffer at the start of the operation.
    this.intersectionPoints.clear();

    // 2. Populate the buffer during calculations.
    for (Entity entity : world.getEntities()) {
        if (isColliding(entity)) {
            Vector4d point = this.intersectionPoints.next();
            point.set(entity.getCollisionPoint());
        }
    }

    // 3. Process the results.
    processPoints(this.intersectionPoints);
}
```

### Anti-Patterns (Do NOT do this)
-   **Exceeding Capacity:** Calling `next()` more times than the capacity specified in the constructor will throw an `ArrayIndexOutOfBoundsException`. The buffer does not resize.
-   **Concurrent Access:** Sharing a single instance across multiple threads without external locking will lead to undefined behavior and data corruption.
-   **Persistent Storage:** Using this class to store data between frames or across major state changes is incorrect. It is a temporary scratchpad, not a permanent data store. The `clear` method must be called before each new computational cycle.

## Data Pipeline
This class does not typically participate in a broad, cross-system data pipeline. Instead, it serves as a high-performance, localized buffer *within* a single stage of a larger process, such as a physics or culling system.

> **Flow within a Physics Tick:**
>
> Physics System Update -> **Vector4dBufferList (Creation/Clear)** -> Broadphase Collision Check -> Narrowphase (Populate buffer with contact points) -> Collision Resolution (Read from buffer) -> End of Tick

