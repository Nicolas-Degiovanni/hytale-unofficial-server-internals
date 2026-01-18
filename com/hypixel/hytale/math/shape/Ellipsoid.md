---
description: Architectural reference for Ellipsoid
---

# Ellipsoid

**Package:** com.hypixel.hytale.math.shape
**Type:** Transient

## Definition
```java
// Signature
public class Ellipsoid implements Shape {
```

## Architecture & Concepts
The Ellipsoid class is a fundamental mathematical primitive within the Hytale engine's geometry systems. It represents a 3D ellipsoid defined by three distinct radii along the X, Y, and Z axes. Unlike managed services, this class is a simple, mutable data structure designed for high-performance, short-lived calculations.

Its primary role is to provide a concrete implementation of the Shape interface, enabling systems like physics engines, spell casting logic, and world modification tools to perform spatial queries and volumetric operations. Its design prioritizes performance and low memory allocation overhead by allowing direct field manipulation and in-place modification, trading safety for speed in performance-critical code paths.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via the `new` keyword. There is no factory or manager responsible for their creation. They are intended to be instantiated on the stack or as temporary fields within other objects that perform geometric calculations.
- **Scope:** The lifecycle of an Ellipsoid instance is typically very short. It is scoped to the lifetime of the method or operation that requires it, such as a single collision detection check or a single area-of-effect calculation.
- **Destruction:** Instances are managed by the Java garbage collector. They are eligible for collection as soon as they are no longer referenced. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** The state of an Ellipsoid is entirely defined by its three public double fields: radiusX, radiusY, and radiusZ. This state is highly mutable and can be modified directly or through methods like `assign` and `expand`. This design avoids the overhead of creating new objects for every modification.

- **Thread Safety:** **This class is not thread-safe.** Its public, mutable fields make it susceptible to race conditions and inconsistent state if shared and modified across multiple threads without external synchronization.

    **WARNING:** An Ellipsoid instance should be confined to a single thread. If concurrent access is unavoidable, all reads and writes to its radii must be protected by explicit locks. Failure to do so will result in unpredictable geometric calculations.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Ellipsoid(radius) | constructor | O(1) | Creates a sphere (an ellipsoid with equal radii). |
| Ellipsoid(rX, rY, rZ) | constructor | O(1) | Creates an ellipsoid with specified radii. |
| assign(radius) | Ellipsoid | O(1) | Modifies the instance in-place to become a sphere. |
| getBox(x, y, z) | Box | O(1) | Computes the axis-aligned bounding box for the ellipsoid at a given center. |
| containsPosition(x, y, z) | boolean | O(1) | Performs a point-in-volume check. The coordinates are relative to the ellipsoid's center. |
| expand(radius) | void | O(1) | Increases each radius by the given amount, modifying the instance in-place. |
| forEachBlock(...) | boolean | O(rX * rY * rZ) | Iterates over every integer block coordinate within the ellipsoid's volume. Delegates to BlockSphereUtil. |

## Integration Patterns

### Standard Usage
The Ellipsoid is intended for direct instantiation and use in localized, single-threaded calculations. It is commonly used to define the area of effect for a game event.

```java
// Define the shape for a magical explosion
Ellipsoid explosionShape = new Ellipsoid(5.0, 3.0, 5.0);

// Check if a specific entity is within the blast radius
// (Assuming explosion is centered at 0,0,0 for this check)
boolean isHit = explosionShape.containsPosition(entity.x, entity.y, entity.z);

// Or, iterate through all blocks to change their type
explosionShape.forEachBlock(centerX, centerY, centerZ, 0.25, (x, y, z) -> {
    world.setBlock(x, y, z, Block.AIR);
    return true; // Continue iteration
});
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never share and modify a single Ellipsoid instance across multiple threads without explicit locking. This is the most common source of errors when using this class.

- **Stateful Reuse without Reset:** Reusing an Ellipsoid instance for multiple, unrelated calculations without re-initializing its radii can lead to subtle and hard-to-debug errors.

    ```java
    // BAD: The second calculation is contaminated by the first
    Ellipsoid shape = new Ellipsoid(5.0);
    shape.expand(2.0); // Now has radius 7.0
    
    // This next check incorrectly uses a radius of 7.0 instead of the intended 5.0
    boolean result = shape.containsPosition(somePos); 
    ```

## Data Pipeline
The most significant data flow involving the Ellipsoid is its use in volumetric block iteration via `forEachBlock`. This process transforms a continuous mathematical shape into a discrete set of block coordinates for world manipulation.

> Flow:
> 1. An **Ellipsoid** instance is created with defined radii.
> 2. A system calls `forEachBlock`, providing a world-space center and a consumer function.
> 3. The call is delegated to the low-level `BlockSphereUtil`.
> 4. `BlockSphereUtil` calculates an integer-based bounding box around the shape.
> 5. It iterates through every block coordinate within this bounding box.
> 6. For each coordinate, it performs a point-in-ellipsoid check.
> 7. If the check passes, the provided `consumer` function is invoked with the block's (x, y, z) coordinates.
> 8. The return value of the consumer determines if the iteration should continue or halt.

