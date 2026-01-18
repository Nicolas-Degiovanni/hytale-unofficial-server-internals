---
description: Architectural reference for Box
---

# Box

**Package:** com.hypixel.hytale.math.shape
**Type:** Transient

## Definition
```java
// Signature
public class Box implements Shape {
```

## Architecture & Concepts
The Box class is a fundamental geometric primitive representing an Axis-Aligned Bounding Box (AABB). It is a core data structure within the engine's math and physics systems, used extensively for collision detection, spatial queries, culling, and defining the physical volume of entities and world geometry.

A Box is defined by two three-dimensional vectors, **min** and **max**, which represent the coordinates of its two opposing corners. By design, its edges are always parallel to the coordinate system axes. This constraint simplifies intersection and containment calculations, making it computationally cheaper than more complex shapes like Oriented Bounding Boxes (OBB) or arbitrary convex hulls.

A critical architectural concept is the handling of transformations. Operations like rotation do not produce a true rotated box. Instead, they compute a new, larger, axis-aligned Box that fully encloses the volume of the original Box after the rotation. This ensures the object remains an AABB, preserving its computational advantages at the cost of spatial precision.

The class implements a fluent interface, where most modification methods return the instance itself to allow for chained operations.

## Lifecycle & Ownership
- **Creation:** A Box is a transient data object. It is instantiated on-demand whenever a spatial calculation is required. This can be done via its constructors, static factory methods like centeredCube, or by cloning an existing instance. It is not managed by a central registry or dependency injection framework.
- **Scope:** The lifetime of a Box instance is typically very short. They are often created as temporary variables within a single method's scope, used for a calculation, and then discarded. They are not intended to be long-lived stateful objects.
- **Destruction:** Instances are managed by the Java Garbage Collector and are eligible for cleanup as soon as they are no longer referenced. There is no manual destruction or pooling mechanism.

## Internal State & Concurrency
- **State:** The Box class is **highly mutable**. Its primary state consists of the public final **min** and **max** Vector3d fields. While the field references themselves are final, the Vector3d objects they point to are mutable. Nearly all transformative methods (e.g., offset, scale, union) modify the internal state of the instance directly.

- **Thread Safety:** This class is **not thread-safe**. Its mutable nature and the absence of any synchronization mechanisms make it unsafe for concurrent access. If a Box instance is shared between threads, all access must be synchronized externally. Failure to do so will result in race conditions and unpredictable geometric state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| union(Box bb) | Box | O(1) | Expands this box to contain both its original volume and the volume of the provided box. Modifies the instance in-place. |
| offset(double x, double y, double z) | Box | O(1) | Translates the box by the given vector. Modifies the instance in-place. |
| isIntersecting(Box other) | boolean | O(1) | Performs a fast check to determine if this box overlaps with another. |
| containsPosition(double x, double y, double z) | boolean | O(1) | Checks if a given 3D point is inside the volume of the box. |
| forEachBlock(...) | boolean | O(W*H*D) | Iterates over every integer block coordinate that intersects the box's volume, executing a consumer predicate for each. |
| normalize() | Box | O(1) | Ensures that each component of the **min** vector is less than or equal to the corresponding component of the **max** vector by swapping them if necessary. |

## Integration Patterns

### Standard Usage
A Box is typically used for transient calculations within game logic, such as physics updates or interaction checks. The fluent API is leveraged for concise transformations.

```java
// Example: Check if an expanded player bounding box intersects with a static zone.
Box playerBounds = new Box(player.getPosition(), player.getPosition()).extend(0.5, 1.8, 0.5);
Box interactionZone = new Box(10, 0, 10, 20, 5, 20);

// Move the player box and check for intersection
playerBounds.offset(player.getVelocity());
if (playerBounds.isIntersecting(interactionZone)) {
    // Handle interaction
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never share and modify a single Box instance across multiple threads without external locking. This is the most common and severe misuse of this class.
- **Misinterpreting Rotation:** Do not assume that methods like rotateX produce an Oriented Bounding Box. The result is a new, larger AABB that contains the rotated shape. This loss of precision is intentional. If you need a true rotated box, you must use a different data structure.
- **Unintended State Mutation:** Because methods modify the instance in-place, be cautious when passing a Box as an argument. If the receiving method modifies the Box, the change will be reflected in the calling scope. If the original state is needed, always pass a clone.
    ```java
    // BAD: The original box is modified by the method
    Box original = new Box(0,0,0,1,1,1);
    someMethodThatOffsets(original); // original is now changed

    // GOOD: Pass a copy to preserve the original
    Box original = new Box(0,0,0,1,1,1);
    someMethodThatOffsets(original.clone()); // original is unchanged
    ```

## Data Pipeline
The Box is not a pipeline stage itself, but rather a data payload that flows through various engine systems. It represents a key piece of spatial information used to make decisions at different stages of a process, such as rendering or physics simulation.

> **Physics Collision Pipeline Example:**
> Entity Transform -> **Calculate AABB (Box)** -> Broad-phase System (using Box intersections) -> Narrow-phase System (for precise collision) -> Collision Resolution

> **Rendering Culling Pipeline Example:**
> Camera Frustum -> World Octree Query -> **Retrieve Object AABBs (Box)** -> Frustum-Box Intersection Test -> Render Visible Objects

