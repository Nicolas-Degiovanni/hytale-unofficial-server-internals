---
description: Architectural reference for BoxBlockIterator
---

# BoxBlockIterator

**Package:** com.hypixel.hytale.math.iterator
**Type:** Utility

## Definition
```java
// Signature
public final class BoxBlockIterator {
```

## Architecture & Concepts
The BoxBlockIterator is a high-performance, low-level mathematics utility designed for discrete 3D grid traversal. Its primary function is to determine every integer grid cell (block) that a moving Axis-Aligned Bounding Box (AABB) intersects along a linear path. This is commonly known as a "swept AABB" or "continuous collision detection" query against a grid.

This class is a foundational component for systems requiring efficient spatial queries, such as:
*   Physics engines detecting collisions for moving entities.
*   AI pathfinding to check for line-of-sight or potential obstacles.
*   Game logic for area-of-effect calculations or volumetric selection tools.

The core architectural pattern is the use of a **ThreadLocal** buffer (`BoxIterationBuffer`). This design choice is critical for two reasons:
1.  **Performance:** It avoids heap allocations for buffer objects during iteration, which is crucial in tight game loops where garbage collection pressure must be minimized. Each thread reuses its own buffer for subsequent iterations.
2.  **Concurrency:** It makes the entire utility inherently thread-safe without the need for locks or synchronization. Parallel systems (e.g., a multi-threaded physics engine) can call `iterate` simultaneously without data corruption.

The iteration logic is decoupled from the action via the `BoxIterationConsumer` interface, a form of the Strategy pattern. The caller provides the behavior to be executed for each intersected block, and the iterator can terminate early if the consumer returns false.

## Lifecycle & Ownership
- **Creation:** The BoxBlockIterator class is a static utility and is never instantiated. A private constructor enforces this by throwing an UnsupportedOperationException. The internal `BoxIterationBuffer` objects are created lazily by the `ThreadLocal` factory, once per thread, the first time a thread invokes an `iterate` method.
- **Scope:** The utility class is application-scoped. The `BoxIterationBuffer` instances are thread-scoped; they persist for the lifetime of the thread that created them.
- **Destruction:** There is no explicit destruction mechanism. The `BoxIterationBuffer` objects are eligible for garbage collection when their parent thread terminates.

## Internal State & Concurrency
- **State:** The BoxBlockIterator class itself is stateless. All state required for a traversal operation is stored within a `BoxIterationBuffer` instance. This buffer is highly mutable during the execution of an `iterate` call but is confined to a single thread.
- **Thread Safety:** This class is **thread-safe**. The `ThreadLocal` storage model guarantees that each thread operates on its own private instance of `BoxIterationBuffer`. This eliminates the possibility of race conditions between different threads calling `iterate` concurrently.

**WARNING:** While the class is thread-safe, the `BoxIterationBuffer` object itself is not. Retrieving the buffer via `getBuffer` and sharing it across threads will violate the design contract and lead to undefined behavior.

## API Surface
The API consists of a series of static, overloaded `iterate` methods. They all provide the same core functionality with different argument signatures for convenience.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| iterate(minX, ..., consumer, buffer) | boolean | O(N) | The core traversal algorithm. Visits every grid cell intersected by the swept box. N is the number of intersected cells. Returns false if the consumer terminates the iteration early. |
| iterate(box, position, d, maxDistance, consumer) | boolean | O(N) | A convenience overload that uses the thread-local buffer automatically. This is the most common entry point. |
| getBuffer() | BoxIterationBuffer | O(1) | Retrieves the calling thread's unique iteration buffer. For advanced use cases only. |

## Integration Patterns

### Standard Usage
The standard pattern is to call a static `iterate` method and provide a `BoxIterationConsumer` implementation, often as a lambda expression. The consumer logic should return `false` to stop the iteration early, for example, after finding the first collision.

```java
// Find the first block a 1x1x1 box collides with when moving
// from 'startPos' along 'direction' for 10 units.

Box entityBox = new Box(new Vector3d(-0.5, 0, -0.5), new Vector3d(0.5, 1, 0.5));
Vector3d startPos = new Vector3d(10.5, 65.0, 20.5);
Vector3d direction = new Vector3d(1, 0, 0); // Moving along the +X axis

final AtomicReference<Vector3i> firstHit = new AtomicReference<>();

boolean iterationCompleted = BoxBlockIterator.iterate(
    entityBox,
    startPos,
    direction,
    10.0,
    (x, y, z) -> {
        // This lambda is the BoxIterationConsumer.
        // It is called for each intersected block coordinate.
        System.out.println("Checking block at: " + x + ", " + y + ", " + z);
        
        // Found a solid block, store it and stop iterating.
        if (world.isBlockSolid(x, y, z)) {
            firstHit.set(new Vector3i(x, y, z));
            return false; // Terminate early
        }
        
        return true; // Continue to next block
    }
);

if (firstHit.get() != null) {
    System.out.println("First collision at: " + firstHit.get());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new BoxBlockIterator()`. The class is not designed to be instantiated.
- **Cross-Thread Buffer Sharing:** Do not retrieve the buffer on one thread and pass it to another. This completely bypasses the `ThreadLocal` safety mechanism and will cause severe concurrency issues.
```java
// DO NOT DO THIS
// This code is fundamentally broken and will cause race conditions.
BoxBlockIterator.BoxIterationBuffer sharedBuffer = BoxBlockIterator.getBuffer();

new Thread(() -> {
    // Using a buffer from another thread is an error.
    BoxBlockIterator.iterate(box, pos, dir, 10, consumer, sharedBuffer); 
}).start();
```

## Control Flow
The iterator does not transform data but rather follows a control flow to execute user-provided logic at specific spatial coordinates.

> Flow:
> Caller provides `Box`, path vectors, and `Consumer` -> `iterate` method retrieves thread-local `BoxIterationBuffer` -> Internal state (signs, dimensions) is calculated and stored in buffer -> Core logic delegates to `BlockIterator` for single-point traversal -> For each step, a callback calculates the box's swept area -> `Consumer.accept(x, y, z)` is invoked for each block in the area -> `Consumer` return value controls early termination.

