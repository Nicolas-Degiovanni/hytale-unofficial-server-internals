---
description: Architectural reference for RaycastAABB
---

# RaycastAABB

**Package:** com.hypixel.hytale.math.raycast
**Type:** Utility

## Definition
```java
// Signature
public class RaycastAABB {
```

## Architecture & Concepts
The **RaycastAABB** class is a fundamental, low-level mathematical utility that provides highly optimized intersection tests between a 3D ray and an Axis-Aligned Bounding Box (AABB). It is a core component of the physics and rendering engine, used for tasks such as player interaction, projectile collision detection, and AI line-of-sight calculations.

This class is designed as a stateless collection of static methods, ensuring maximum performance and thread safety. It does not represent a specific ray or AABB; rather, it provides the pure mathematical functions to operate on them.

The intersection algorithm is a variation of the slab test, which checks for intersections between the ray and the pairs of planes that define the AABB on each axis (X, Y, Z). It calculates the intersection distance, *t*, for each of the six faces of the box and determines the nearest valid intersection point in the direction of the ray.

A critical implementation detail is the use of the **EPSILON** constant. This small negative value is used in comparisons to prevent floating-point precision errors, particularly for rays originating on or very near the surface of an AABB. This ensures that such rays are correctly registered as intersections.

The API provides two distinct operational patterns:
1.  **Direct Return:** A simple variant of **intersect** returns the distance to the nearest intersection as a *double*. A value of **Double.POSITIVE_INFINITY** indicates no intersection occurred.
2.  **Consumer Callback:** Overloaded variants of **intersect** accept a functional interface (**RaycastConsumer** and its derivatives). This is a zero-allocation pattern designed for performance-critical code. Instead of returning a result object, it invokes the provided consumer with detailed hit information, including the intersection normal. The generic `Plus` variants allow for passing context-specific data (e.g., the entity or block being tested) through the calculation into the callback, avoiding the need for external state management in the calling loop.

## Lifecycle & Ownership
- **Creation:** As a static utility class, **RaycastAABB** is never instantiated. Its methods are accessed statically (e.g., **RaycastAABB.intersect(...)**).
- **Scope:** The class is loaded by the JVM and its methods are available for the entire application lifetime. It is globally accessible and has no instance-specific scope.
- **Destruction:** The class is unloaded by the ClassLoader when the application terminates. No manual cleanup is required.

## Internal State & Concurrency
- **State:** **RaycastAABB** is completely stateless. All required data (AABB bounds, ray origin, ray direction) is passed as method parameters. It performs calculations and returns a result without modifying any internal or external state.
- **Thread Safety:** This class is inherently thread-safe. Due to its stateless nature, any method can be called by any number of threads concurrently without risk of race conditions or data corruption. No synchronization mechanisms are required.

## API Surface
The public API consists exclusively of overloaded static **intersect** methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| intersect(min/max, origin, dir) | double | O(1) | Calculates the nearest intersection distance. Returns **Double.POSITIVE_INFINITY** if no hit occurs. |
| intersect(..., consumer) | void | O(1) | Performs intersection test and invokes the provided consumer with detailed results, including the surface normal. |
| intersect(..., consumer, obj1) | void | O(1) | Same as above, but passes one generic context object to the consumer. |
| intersect(..., consumer, obj1, obj2) | void | O(1) | Same as above, but passes two generic context objects to the consumer. |
| intersect(..., consumer, obj1, obj2, obj3) | void | O(1) | Same as above, but passes three generic context objects to the consumer. |

## Integration Patterns

### Standard Usage
The consumer-based pattern is the recommended approach for performance-sensitive systems like world traversal or physics queries, as it avoids heap allocations for result objects within a loop.

```java
// A physics system checking a ray against a specific entity's bounding box.
Entity targetEntity = ...;
AABB entityBounds = targetEntity.getAABB();
Ray playerLookRay = player.getLookRay();

// The lambda captures the entity to provide context on a successful hit.
RaycastAABB.intersect(
    entityBounds.minX, entityBounds.minY, entityBounds.minZ,
    entityBounds.maxX, entityBounds.maxY, entityBounds.maxZ,
    playerLookRay.originX, playerLookRay.originY, playerLookRay.originZ,
    playerLookRay.dirX, playerLookRay.dirY, playerLookRay.dirZ,
    (hit, ox, oy, oz, dx, dy, dz, t, nx, ny, nz, entity) -> {
        if (hit && t < nearestHitDistance) {
            // Logic to handle the hit on 'entity'
            // Update nearest hit, record normal, etc.
        }
    },
    targetEntity
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new RaycastAABB()`. This is a static utility class and cannot be instantiated.
- **Ignoring Infinity Return:** When using the variant that returns a *double*, failing to check for **Double.POSITIVE_INFINITY** will lead to incorrect behavior, as this value indicates a miss. Any subsequent calculations using this value will be invalid.
- **Assuming Hit from Callback:** The consumer is *always* called, even on a miss. The first parameter to the consumer is a boolean indicating whether an intersection occurred. Code inside the consumer must check this boolean before processing other result parameters.

## Data Pipeline
**RaycastAABB** acts as a pure processing function within a larger data flow. It does not manage data persistence or transport.

> Flow:
> System Input (e.g., Player Look Vector) -> Physics Engine -> **RaycastAABB.intersect** -> Consumer Callback -> Gameplay Logic (e.g., Damage Calculation, Block Interaction)

