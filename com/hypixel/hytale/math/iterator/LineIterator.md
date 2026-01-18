---
description: Architectural reference for LineIterator
---

# LineIterator

**Package:** com.hypixel.hytale.math.iterator
**Type:** Transient

## Definition
```java
// Signature
public class LineIterator implements Iterator<Vector3i> {
```

## Architecture & Concepts
The LineIterator is a low-level, high-performance utility for traversing a 3D integer grid along a straight line. It implements a 3D version of Bresenham's line algorithm, a foundational algorithm in computer graphics and computational geometry.

Its primary role within the engine is to provide an efficient, integer-only method for determining which voxels or grid cells lie along a path between two points. This is critical for systems performing raycasting, line-of-sight calculations, projectile trajectory simulation, and certain AI behaviors. By avoiding floating-point arithmetic, it ensures deterministic and fast traversal, which is essential for performance-sensitive game logic.

The class is designed as a standard Java Iterator, allowing it to be used seamlessly in loops and with other parts of the collections framework. Each call to the next method returns a new Vector3i representing the next voxel on the line.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its public constructor: `new LineIterator(...)`. It is not managed by a service locator or dependency injection framework. It is intended to be instantiated on-demand by any system that requires a line traversal.
- **Scope:** The object's lifetime is typically confined to the scope of a single method. It is created to perform one specific traversal and is then discarded.
- **Destruction:** The LineIterator holds no external resources and is managed entirely by the Java garbage collector. It becomes eligible for collection as soon as it is no longer referenced.

## Internal State & Concurrency
- **State:** The LineIterator is a stateful, **mutable** object. Its internal state, including the current coordinates (pointX, pointY, pointZ) and the algorithm's error terms (err1, err2), is modified with every call to the next method. The initial parameters calculated in the constructor are immutable for the life of the instance.

- **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access. Sharing a single LineIterator instance across multiple threads will result in a race condition, leading to corrupted internal state and unpredictable, non-deterministic iteration.

    **WARNING:** Any system that performs line traversals in a multi-threaded context must ensure that each thread creates and uses its own unique LineIterator instance.

## API Surface
The public contract is minimal, adhering to the standard Java Iterator interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| LineIterator(x1, y1, z1, x2, y2, z2) | constructor | O(1) | Constructs a new iterator for the line between the two specified points. |
| hasNext() | boolean | O(1) | Returns true if the iteration has more voxels to visit. |
| next() | Vector3i | O(1) | Returns the next voxel coordinate on the line and advances the iterator. Throws NoSuchElementException if the iteration is complete. |

## Integration Patterns

### Standard Usage
The canonical use of LineIterator is within a standard `while` loop to process each voxel along a path.

```java
// Example: Perform a line-of-sight check to see if a target is visible.
LineIterator lineOfSight = new LineIterator(
    observer.getX(), observer.getY(), observer.getZ(),
    target.getX(), target.getY(), target.getZ()
);

boolean isObstructed = false;
while (lineOfSight.hasNext()) {
    Vector3i currentVoxel = lineOfSight.next();

    // Skip the starting voxel itself
    if (currentVoxel.equals(observer.getPosition())) {
        continue;
    }

    if (world.isVoxelOpaque(currentVoxel)) {
        isObstructed = true;
        break; // Path is blocked, no need to check further.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Iterator Re-use:** An iterator instance is single-use and cannot be reset. Once `hasNext()` returns false, the iterator is considered exhausted and must be discarded. Attempting to continue using it will result in exceptions. For a new traversal, even along the same path, create a new instance.
- **Concurrent Access:** As stated in the concurrency section, never share a LineIterator instance between threads. This will corrupt its state.
- **Modification of Internal State:** Do not attempt to use reflection or other means to modify the internal state of the iterator during traversal. This will break the underlying Bresenham's algorithm and produce invalid results.

## Data Pipeline
The data flow for this component is self-contained and linear. It does not interact with external systems or event buses; it is a pure computational utility.

> Flow:
> Start & End Coordinates -> **LineIterator Constructor (State Initialization)** -> `while(hasNext())` Loop -> **`next()` Method (State Update & Vector Creation)** -> Output Vector3i

