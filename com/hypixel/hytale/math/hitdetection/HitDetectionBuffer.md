---
description: Architectural reference for HitDetectionBuffer
---

# HitDetectionBuffer

**Package:** com.hypixel.hytale.math.hitdetection
**Type:** Transient State Object

## Definition
```java
// Signature
public class HitDetectionBuffer {
```

## Architecture & Concepts
The HitDetectionBuffer is a specialized, pre-allocated data structure designed to serve as a high-performance scratchpad for geometric calculations within the engine's hit detection systems. Its primary architectural role is to eliminate memory allocation overhead and reduce garbage collection pressure during performance-critical operations, such as raycasting or collision testing, which may run thousands of times per frame.

Instead of creating new vectors, shapes, and lists for each calculation, a single HitDetectionBuffer instance is passed into a detection algorithm. The algorithm performs all intermediate and final calculations using the fields within the buffer. The calling system then reads the results directly from the same object. This pattern of reusing a mutable state container is fundamental to achieving high-throughput, low-latency physics and interaction simulations in the Hytale engine.

This class is not a service; it is pure state. It holds no logic and exists solely to be manipulated by external static methods or service classes in the `com.hypixel.hytale.math.hitdetection` package.

### Lifecycle & Ownership
- **Creation:** Instances are expected to be created once by a managing system and then reused. Common owners include thread-local storage pools, a central `CollisionSystem`, or as a member field of a long-lived object that performs frequent hit detection, like a `World` or `Player` entity.
- **Scope:** The lifetime of a HitDetectionBuffer is tied to its owner. For a thread-local buffer, it persists for the life of the thread. For a component-owned buffer, it persists for the life of that component. Its *data*, however, is transient and only considered valid for the duration of a single, synchronous hit detection operation.
- **Destruction:** The object is managed by the Java Garbage Collector. It holds no native resources and requires no explicit cleanup. It is reclaimed when its owner is destroyed and all references are released.

## Internal State & Concurrency
- **State:** The HitDetectionBuffer is fundamentally **Mutable**. Its entire purpose is to have its public fields modified by external algorithms. The state contained within is ephemeral and should be considered undefined before the start of any new hit detection operation. It does not cache data between operations.

- **Thread Safety:** This class is **not thread-safe**. All fields are public and non-synchronized.

    **WARNING:** Sharing a single HitDetectionBuffer instance across multiple threads will result in severe data corruption and race conditions. Each thread performing collision checks **must** have its own dedicated instance of HitDetectionBuffer. A common and recommended pattern is to use a `ThreadLocal<HitDetectionBuffer>`.

## API Surface
The public contract of this class consists entirely of its fields. There are no methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hitPosition | Vector4d | N/A | **Primary Output.** Stores the world-space coordinates of the intersection point if a hit occurs. |
| containsFully | boolean | N/A | **Primary Output.** A flag indicating if the tested shape was fully contained within another, rather than just intersecting. |
| tempHitPosition | Vector4d | N/A | An internal scratchpad vector for intermediate calculations. |
| transformedQuad | Quad4d | N/A | An internal buffer for storing a transformed quadrilateral during intersection tests. |
| vertexList1 | Vector4dBufferList | N/A | A pre-allocated list for storing vertex data, typically used in clipping algorithms. |
| vertexList2 | Vector4dBufferList | N/A | A second pre-allocated vertex list, often used as the output destination for a clipping operation. |

## Integration Patterns

### Standard Usage
The buffer is retrieved from a pool or a thread-local variable, passed to a static utility that performs the calculation, and then the results are read from the buffer's public fields.

```java
// Hypothetical usage within a CollisionSystem
// 1. Obtain a thread-safe buffer instance
HitDetectionBuffer buffer = THREAD_LOCAL_BUFFER.get();

// 2. Pass the buffer to a detection algorithm
boolean didHit = HitDetector.raycastAgainstAABB(ray, targetBox, buffer);

// 3. Read results from the buffer
if (didHit) {
    Vector4d impactPoint = buffer.hitPosition;
    System.out.println("Hit detected at: " + impactPoint);
}
```

### Anti-Patterns (Do NOT do this)
- **Instantiation in a Loop:** Creating a new HitDetectionBuffer inside a game loop or any frequently executed code path is a critical performance anti-pattern. This completely negates the class's purpose and will cause significant GC stutter.
    ```java
    // ANTI-PATTERN: Creates excessive garbage
    for (Entity e : world.entities) {
        HitDetectionBuffer buffer = new HitDetectionBuffer(); // DO NOT DO THIS
        HitDetector.test(player, e, buffer);
    }
    ```
- **Sharing Instances Across Threads:** As stated previously, using a singleton or globally shared instance in a multi-threaded environment is unsafe and will lead to unpredictable behavior.
- **Assuming State Persistence:** Do not rely on data from a previous operation. The buffer's state must be treated as invalid before each new call to a detection algorithm.

## Data Pipeline
The HitDetectionBuffer acts as a transient data container within a larger computational flow. It does not process data itself but is instead written to and read from.

> Flow:
> Input Primitives (e.g., Ray, AABB) -> Collision Algorithm -> **HitDetectionBuffer** (is populated with intermediate and final results) -> Calling System (reads `hitPosition`, `containsFully`, etc.)

