---
description: Architectural reference for VectorUtil
---

# VectorUtil

**Package:** com.hypixel.hytale.builtin.hytalegenerator
**Type:** Utility

## Definition
```java
// Signature
public class VectorUtil {
```

## Architecture & Concepts
VectorUtil is a foundational, stateless utility class that provides a suite of advanced geometric and mathematical functions for vector types. It serves as a centralized computational kernel for systems requiring complex spatial reasoning, particularly within the procedural content generation pipeline of the `hytalegenerator`.

This class exists to augment the basic arithmetic capabilities of the core Vector classes (Vector3d, Vector3i, etc.). While the Vector objects themselves handle operations like addition and scaling, VectorUtil provides higher-level algorithms for distance calculation, collision detection, rotation, and line-segment analysis. By centralizing these implementations, the engine avoids code duplication and ensures that performance-critical geometric calculations are consistent and optimized.

Architecturally, VectorUtil is a pure function library. It holds no internal state and its methods' outputs are determined solely by their inputs. It is a low-level dependency for any system that manipulates object positions, orientations, or bounding volumes in 2D or 3D space.

## Lifecycle & Ownership
As a static utility class, VectorUtil does not follow a traditional object lifecycle.

- **Creation:** The VectorUtil class is never instantiated. There is no `new VectorUtil()` constructor call. The class is loaded into the JVM by the ClassLoader on its first use, at which point its static methods become available.
- **Scope:** The class and its static methods are available globally for the entire application lifetime once loaded.
- **Destruction:** The class definition is unloaded from the JVM only when its defining ClassLoader is eligible for garbage collection, which typically occurs during application shutdown.

## Internal State & Concurrency
- **State:** **Stateless**. VectorUtil contains no instance or static fields, making it entirely stateless. All computations are performed exclusively on the arguments passed to its methods.

    **WARNING:** While the class itself is stateless, several of its methods mutate the input vector objects directly (e.g., `rotateAroundAxis`, `bitShiftRight`). Callers must be aware of this pass-by-reference behavior. Methods that are non-mutating typically use `clone()` internally to create new vector instances, ensuring input parameters remain unchanged.

- **Thread Safety:** **Thread-Safe**. Due to its stateless nature, all methods in VectorUtil can be safely called by multiple threads concurrently without risk of internal data corruption.

    **WARNING:** The caller is responsible for ensuring thread safety of the *arguments* passed to the methods. Passing a Vector object that is being simultaneously modified by another thread to any method in this class will result in a race condition and undefined behavior. This is especially critical for methods that perform in-place mutations.

## API Surface
The API provides a collection of static methods for geometric queries and transformations. The following is a representative sample.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| areasOverlap(minA, maxA, minB, maxB) | boolean | O(1) | Performs an Axis-Aligned Bounding Box (AABB) intersection test. Returns true if the two boxes overlap. |
| distanceToSegment3d(point, p0, p1) | double | O(1) | Calculates the shortest distance from a point to a finite line segment in 3D space. |
| shortestDistanceBetweenTwoSegments(...) | double | O(1) | A computationally intensive method to find the minimum distance between two 3D line segments. |
| rotateAroundAxis(vec, axis, theta) | void | O(1) | **Mutates** the input `vec` by rotating it around the given axis by theta radians. |
| orderByDistanceFrom(origin, vectors) | List<Vector2i> | O(N log N) | Creates and returns a new list of vectors sorted by their distance from the origin point. The input list is not modified. |

## Integration Patterns

### Standard Usage
VectorUtil methods are called statically. It is commonly used within world generation algorithms, physics simulations, and AI pathfinding logic to make spatial decisions.

```java
// Example: Find the closest point on a path segment to an entity
Vector3d entityPosition = new Vector3d(15, 20, 10);
Vector3d pathNodeA = new Vector3d(0, 20, 0);
Vector3d pathNodeB = new Vector3d(100, 20, 0);

Vector3d closestPoint = VectorUtil.nearestPointOnSegment3d(entityPosition, pathNodeA, pathNodeB);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Attempting to create an instance via `new VectorUtil()` is incorrect. The class is designed for static access only.
- **Ignoring Mutating Behavior:** Calling methods like `rotateAroundAxis` or `bitShiftLeft` with the expectation that they return a new vector is a common error. These methods modify their arguments in-place, which can lead to subtle bugs if the original vector was expected to be preserved.
- **Concurrent Argument Modification:** Passing a vector to a VectorUtil method from one thread while another thread is actively changing its components is not safe and will lead to unpredictable results.

## Data Pipeline
VectorUtil does not participate in a data pipeline as a persistent stage; rather, it acts as a synchronous, on-demand processing function *within* a pipeline. It transforms or validates data but does not forward it.

> **Example Flow: Structure Placement**
>
> World Generator State -> Propose Structure Location (Vector3i) -> **VectorUtil.areasOverlap (Collision Check)** -> Placement Approved/Rejected -> World Data Update

