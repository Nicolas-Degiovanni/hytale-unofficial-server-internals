---
description: Architectural reference for CollisionMath
---

# CollisionMath

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Utility

## Definition
```java
// Signature
public class CollisionMath {
```

## Architecture & Concepts
The CollisionMath class is a foundational, low-level utility providing a suite of high-performance, static mathematical functions for collision detection. It serves as a core toolbox for the server-side physics engine, enabling efficient determination of intersections between geometric primitives.

This class is not a stateful system or manager; it is a stateless collection of pure functions designed for raw computational throughput. Its primary responsibilities include:

*   **Ray-Casting:** Calculating intersections between a ray (or a finite vector) and an Axis-Aligned Bounding Box (AABB). This is fundamental for mechanics like projectile travel or line-of-sight checks.
*   **Continuous Collision Detection:** Implementing swept-volume tests (specifically Swept AABB) to determine if two moving objects will collide between physics ticks. This prevents high-velocity objects from "tunneling" through thin obstacles.
*   **Static Overlap Testing:** Checking for simple intersection between two static AABBs and classifying the nature of the intersection using a bitmask system.

The use of integer bitmasks (e.g., TOUCH_X, OVERLAP_Y) is a critical performance optimization. It allows the engine to encode rich intersection data into a single integer, which can be evaluated rapidly using bitwise operations instead of more expensive conditional logic.

## Lifecycle & Ownership
As a static utility class, CollisionMath does not follow a traditional object lifecycle.

*   **Creation:** This class is never instantiated. The constructor is private and will throw an `IllegalStateException` if invoked via reflection. It exists only as a static container for its methods.
*   **Scope:** The class and its static methods are available globally as soon as the `CollisionMath` class is loaded by the Java ClassLoader. This scope persists for the entire lifetime of the server application.
*   **Destruction:** The class is unloaded from memory only when its ClassLoader is garbage collected, which typically occurs during a full server shutdown.

## Internal State & Concurrency
*   **State:** The CollisionMath class is designed to be stateless. It maintains no instance-level state and its static fields are final constants, with one critical exception for performance optimization.

*   **Thread Safety:** This class is fully thread-safe. Concurrency is managed through a `ThreadLocal` field named MIN_MAX. This field provides each executing thread with its own private instance of a `Vector2d` object. This object is used as a mutable "scratchpad" for intermediate calculations within intersection methods, primarily to return the near and far intersection times (t-values).

    **WARNING:** This `ThreadLocal` pattern is an intentional performance optimization. It avoids the significant overhead of heap-allocating a new `Vector2d` object for every collision check, which can occur thousands of time per second. Because each thread has its own copy, there are no race conditions or data visibility issues between threads calling these methods concurrently.

## API Surface
The public API consists entirely of static methods for performing geometric queries.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| intersectRayAABB(...) | boolean / double | O(1) | Calculates the intersection of a ray with an AABB. Overloads return either a boolean result or the distance to the nearest intersection point. |
| intersectVectorAABB(...) | boolean | O(1) | A specialized version of ray intersection where the ray's length is 1.0, effectively testing a directional vector. |
| intersectSweptAABBs(...) | boolean | O(1) | Performs a continuous collision detection test between a moving AABB and a static AABB. |
| intersectAABBs(...) | int | O(1) | Determines the static overlap state of two AABBs, returning a bitmask that describes the intersection on each axis. |
| isDisjoint(code) | boolean | O(1) | Helper function to interpret the bitmask from intersectAABBs. Returns true if the boxes do not overlap or touch. |
| isOverlapping(code) | boolean | O(1) | Helper function to interpret the bitmask. Returns true if the boxes are fully overlapping on all three axes. |
| isTouching(code) | boolean | O(1) | Helper function to interpret the bitmask. Returns true if the boxes' boundaries are touching on any axis. |

## Integration Patterns

### Standard Usage
CollisionMath methods should be called directly from higher-level game logic, such as a physics simulation loop or an entity movement system, to query for potential collisions.

```java
// Example from a hypothetical EntityMovementSystem
Vector3d currentPosition = entity.getPosition();
Vector3d velocity = entity.getVelocity(); // Per-tick velocity
Box entityBounds = entity.getCollisionBox();
Box worldBlockBounds = world.getBlockCollisionBox(targetPos);

// Use a thread-local scratchpad for the result
Vector2d intersectionTime = new Vector2d(); 
Box minkowskiBox = new Box(); // Temporary box for calculation

// Check for collision in the next tick
boolean willCollide = CollisionMath.intersectSweptAABBs(
    currentPosition,
    velocity,
    entityBounds,
    targetPos,
    worldBlockBounds,
    intersectionTime,
    minkowskiBox
);

if (willCollide) {
    // Respond to the impending collision
    // The 'intersectionTime.x' value can be used for precise resolution
}
```

### Anti-Patterns (Do NOT do this)
*   **Direct Instantiation:** Never attempt to create an instance of this class. It is a static utility and will throw an exception.
    ```java
    // ANTI-PATTERN: Throws IllegalStateException
    CollisionMath math = new CollisionMath();
    ```
*   **External State Management:** Do not attempt to access or manipulate the internal `MIN_MAX` `ThreadLocal` variable. It is an implementation detail and relying on it will lead to brittle code. Pass in your own `Vector2d` result objects where the method signature requires it.

## Data Pipeline
CollisionMath does not operate as a stage in a data pipeline; rather, it is a functional component *used by* stages in a pipeline. It takes geometric data as input and produces a boolean or numeric result, which then informs the logic of the calling system.

> **Physics Tick Pipeline Example:**
>
> Entity State (Position, Velocity) -> Physics System -> **CollisionMath.intersectSweptAABBs** -> Collision Resolution Logic -> Updated Entity State

