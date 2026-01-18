---
description: Architectural reference for CoordinateRotator
---

# CoordinateRotator

**Package:** com.hypixel.hytale.procedurallib.random
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class CoordinateRotator implements ICoordinateRandomizer {
```

## Architecture & Concepts
The CoordinateRotator is a high-performance, immutable mathematical utility designed for applying deterministic 3D rotations to coordinate vectors. It serves as a foundational component within the procedural generation library, enabling the consistent orientation of world features, noise fields, and geometric structures.

Architecturally, its primary purpose is to pre-calculate and cache a 3x3 rotation matrix based on initial pitch and yaw angles. This one-time calculation during instantiation makes subsequent rotation operations extremely fast, reducing them to simple matrix-vector multiplications. This is a critical performance optimization for procedural systems that may need to transform millions of coordinates.

Despite implementing the ICoordinateRandomizer interface, this class is **not random**. It performs a deterministic transformation. The interface methods are implemented to satisfy a contract, but the input *seed* parameter is completely ignored. This design choice allows the CoordinateRotator to be used interchangeably with other, truly random coordinate modifiers within the same procedural framework.

The static constant CoordinateRotator.NONE represents an identity rotation (zero pitch, zero yaw), providing a convenient and efficient way to bypass rotation logic when no transformation is needed.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its public constructor: `new CoordinateRotator(pitch, yaw)`. It is typically created by higher-level procedural generation managers or configuration objects that define the orientation for a specific region or feature.
-   **Scope:** The lifetime of a CoordinateRotator instance is bound to the object that holds a reference to it. It is a lightweight value object and can be either short-lived (for a single operation) or long-lived (cached as part of a larger configuration).
-   **Destruction:** Managed by the Java Garbage Collector. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** The internal state consists of the initial pitch, yaw, and the derived rotation matrix. This state is **deeply immutable**. All fields are final, and the internal matrix array is never modified after its creation in the constructor.
-   **Thread Safety:** The CoordinateRotator is inherently **thread-safe** due to its immutability. A single instance can be safely shared and accessed by multiple threads simultaneously without any need for external synchronization or locks.

## API Surface
The public API provides methods for rotating 2D and 3D vectors. The ICoordinateRandomizer methods are simple delegates to these core rotation methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CoordinateRotator(pitch, yaw) | constructor | O(1) | Creates a new rotator, pre-calculating the internal rotation matrix. |
| rotateX(x, y, z) | double | O(1) | Calculates the rotated X component of a 3D vector. |
| rotateY(x, y, z) | double | O(1) | Calculates the rotated Y component of a 3D vector. |
| rotateZ(x, y, z) | double | O(1) | Calculates the rotated Z component of a 3D vector. |
| createRotationMatrix(pitch, yaw) | static double[] | O(1) | A static factory method to generate a 3x3 rotation matrix as a 9-element array. |

## Integration Patterns

### Standard Usage
A CoordinateRotator should be instantiated once with the desired orientation and then reused for all coordinate transformations within that context.

```java
// Example: Orienting points for a procedural structure
double structureYaw = Math.toRadians(45.0);
double structurePitch = Math.toRadians(15.0);

CoordinateRotator rotator = new CoordinateRotator(structurePitch, structureYaw);

// Define a point in local space
double localX = 10.0;
double localY = 0.0;
double localZ = 5.0;

// Transform the point into world space using the rotator
double worldX = rotator.rotateX(localX, localY, localZ);
double worldY = rotator.rotateY(localX, localY, localZ);
double worldZ = rotator.rotateZ(localX, localY, localZ);
```

### Anti-Patterns (Do NOT do this)
-   **Re-creation in Loops:** Avoid creating new CoordinateRotator instances inside tight loops for the same angles. The cost of recalculating the matrix repeatedly is unnecessary and inefficient. Cache the instance instead.
-   **Expecting Randomness:** Do not use this class with the expectation of random output. It is a deterministic transformer. The class name and its implementation of ICoordinateRandomizer can be misleading; the *seed* parameter in the interface methods has no effect.

## Data Pipeline
The data flow for this component is a simple mathematical transformation. It does not interact with I/O, networking, or other engine systems.

> Flow:
> (Pitch, Yaw) -> **Constructor** -> (Internal Rotation Matrix)
>
> (x, y, z) Coordinates -> **rotateX/Y/Z methods** -> (x', y', z') Rotated Coordinates

