---
description: Architectural reference for Bounds3d
---

# Bounds3d

**Package:** com.hypixel.hytale.builtin.hytalegenerator.bounds
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Bounds3d implements MemInstrument {
```

## Architecture & Concepts
The Bounds3d class is a fundamental geometric primitive representing an Axis-Aligned Bounding Box (AABB). It is a core data structure used throughout the engine for any system requiring spatial awareness, including physics, rendering culling, world generation, and collision detection.

Its design prioritizes performance and low-level control. The class is intentionally mutable, with most operations modifying the instance in-place to reduce object allocation and garbage collection overhead during intensive calculations, such as in the main game loop.

The public final fields, `min` and `max`, expose the underlying Vector3d objects for direct read access, avoiding method call overhead. However, because Vector3d is itself a mutable type, the internal state of a Bounds3d instance can be changed. The class relies on a `correct` method, called during construction and assignment, to maintain the invariant that all components of `min` are less than or equal to the corresponding components of `max`.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by any system performing spatial calculations. There is no central factory or manager. It is typically instantiated directly via `new Bounds3d()`.
- **Scope:** The lifetime of a Bounds3d object is context-dependent. It is often short-lived, existing only within the scope of a single method or frame update. In other cases, it may be a long-lived member of a component, such as a physics body, where its lifetime is tied to its owner.
- **Destruction:** Management is handled by the Java Garbage Collector. Due to the potential for high-frequency allocation, engine systems should consider object pooling for Bounds3d instances in performance-critical code paths to mitigate GC impact.

## Internal State & Concurrency
- **State:** Bounds3d is a **highly mutable** object. Methods such as `assign`, `offset`, `encompass`, and `intersect` directly modify the state of the `min` and `max` vectors. The `clone` method is the designated mechanism for creating a detached copy with independent state.

- **Thread Safety:** This class is **not thread-safe**. Its mutable nature and lack of any internal locking mechanisms make it vulnerable to race conditions and state corruption if accessed concurrently by multiple threads.

    **WARNING:** Any instance of Bounds3d shared between threads must be protected by external synchronization. It is strongly recommended to confine usage to a single thread or to pass copies (via `clone`) between thread boundaries.

## API Surface
The public API is designed for chained, in-place mutations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| contains(Vector3d position) | boolean | O(1) | Performs a non-inclusive check on the max boundary to see if a point is within the volume. |
| intersects(Bounds3d other) | boolean | O(1) | Checks for any spatial overlap with another Bounds3d. |
| encompass(Bounds3d other) | Bounds3d | O(1) | Expands the current bounds to fully contain the other bounds. Modifies the instance. |
| encompass(Vector3d position) | Bounds3d | O(1) | Expands the current bounds to include the specified point. Modifies the instance. |
| intersect(Bounds3d other) | Bounds3d | O(1) | Calculates the intersection volume and updates the instance to match it. If no intersection exists, becomes a zero-volume box. |
| assign(Bounds3d other) | Bounds3d | O(1) | Copies the min and max values from another Bounds3d into this instance. |
| clone() | Bounds3d | O(1) | Creates and returns a new Bounds3d instance with a deep copy of the min and max vectors. |

## Integration Patterns

### Standard Usage
The intended pattern is to create an instance, perform a series of in-place modifications, and then use the result for a final query. This minimizes object allocations.

```java
// Example: Calculating the total bounding box for a list of points
Bounds3d totalBounds = new Bounds3d(); // Starts as a zero-volume box

for (Vector3d point : points) {
    totalBounds.encompass(point); // Mutates totalBounds in-place
}

if (totalBounds.intersects(player.getBounds())) {
    // Perform game logic
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Do not pass a single Bounds3d instance to multiple subsystems without a clear ownership contract. If one system modifies the bounds, it will unexpectedly affect the other. Use `clone()` to provide a safe copy when handing off to another system.
- **Ignoring In-Place Mutation:** Assuming that methods like `intersect` or `encompass` return a new object. They modify the instance they are called on, which can lead to subtle and hard-to-trace bugs.

    ```java
    // BAD: originalBounds is now modified
    Bounds3d originalBounds = entity.getBounds();
    Bounds3d intersection = originalBounds.intersect(other.getBounds());
    // At this point, originalBounds and intersection refer to the same
    // modified object. The original state is lost.

    // GOOD: Operate on a clone to preserve the original
    Bounds3d originalBounds = entity.getBounds();
    Bounds3d intersection = originalBounds.clone().intersect(other.getBounds());
    // originalBounds is unchanged.
    ```
- **Direct Field Manipulation:** Modifying the components of the public `min` and `max` vectors directly bypasses the `correct()` logic that ensures `min <= max`. This can leave the object in an invalid state, causing subsequent intersection and containment checks to fail.

    **WARNING:** Always use methods like `assign` or `encompass` to modify the bounds to guarantee state integrity.

