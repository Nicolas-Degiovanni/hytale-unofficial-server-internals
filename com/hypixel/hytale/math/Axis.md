---
description: Architectural reference for Axis
---

# Axis

**Package:** com.hypixel.hytale.math
**Type:** Utility Enum

## Definition
```java
// Signature
public enum Axis {
```

## Architecture & Concepts
The Axis enum is a foundational component of the Hytale math library, representing the three primary Cartesian axes: X, Y, and Z. It embodies a behavioral enum pattern, where each constant not only represents a value but also encapsulates the logic for axis-aligned transformations.

This design centralizes complex rotational and reflectional mathematics, providing a stable, high-performance API for manipulating 3D vectors. Its primary role is to act as a transformer for spatial data, used extensively in physics, rendering, world generation, and entity orientation systems.

A key architectural decision is that all transformation methods operate **in-place** on the provided vector objects. This is a deliberate performance optimization to minimize object allocation and reduce garbage collector pressure during intensive, per-frame calculations. Callers must be aware that the vector instances they pass to methods like *rotate* or *flip* will be mutated directly.

## Lifecycle & Ownership
- **Creation:** The three instances, X, Y, and Z, are constructed and initialized by the Java Virtual Machine during class loading. They are compile-time constants.
- **Scope:** As static final members of the enum, they are application-scoped and persist for the entire lifetime of the engine. They are effectively global, immutable singletons.
- **Destruction:** The instances are garbage collected only when the JVM shuts down. There is no manual lifecycle management.

## Internal State & Concurrency
- **State:** The Axis enum itself is immutable. Each instance holds a private, final reference to a Vector3i representing its unit direction.
    - **WARNING:** While the reference to the internal direction vector is final, the Vector3i object itself is mutable. To prevent corruption of this internal state, the public getDirection method returns a defensive copy (a clone) of the vector. External modifications to the returned vector will not affect the enum's canonical state.
- **Thread Safety:** This class is unconditionally thread-safe. The enum instances hold no mutable state that is exposed to callers. All transformation methods operate exclusively on the arguments passed into them. The responsibility for ensuring the thread-safe use of vector objects lies entirely with the calling code. If two threads pass the same vector instance to an Axis method, a race condition will occur on the vector, not on the Axis enum.

## API Surface
The public contract focuses on in-place vector transformations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDirection() | Vector3i | O(1) | Returns a **new** Vector3i instance representing the axis direction. Safe from state corruption. |
| rotate(vector, angle) | void | O(N) | Rotates the given vector around the axis. N is angle / 90. Mutates the vector. |
| rotate(vector) | void | O(1) | Performs a single 90-degree right-handed rotation on the vector. Mutates the vector. |
| flip(vector) | void | O(1) | Flips or reflects the vector across the plane perpendicular to this axis. Mutates the vector. |
| flipRotation(rotation) | void | O(1) | Performs a specialized flip on a Vector3f representing yaw and pitch. Mutates the vector. |

## Integration Patterns

### Standard Usage
The primary pattern is to select a canonical axis and invoke a transformation method on a target vector. This is common in block placement logic or entity orientation updates.

```java
// A vector representing a block's orientation
Vector3i blockFacing = new Vector3i(0, 0, 1);

// Rotate the block 90 degrees around the world's Y-axis
Axis.Y.rotate(blockFacing);

// blockFacing is now (-1, 0, 0)
```

### Anti-Patterns (Do NOT do this)
- **Stateful Assumption:** Do not modify the vector returned by getDirection and expect it to alter the behavior of future rotations. The method provides a copy, not a live reference.
- **Incorrect Angle:** The `rotate(vector, angle)` methods are designed exclusively for multiples of 90 degrees. Passing other values (e.g., 45) will result in an incomplete and mathematically incorrect transformation, as the internal logic iterates in 90-degree steps.
- **Ignoring Mutation:** Passing a vector to any transformation method cedes control of its state. Do not read the vector from another thread during the operation. If the original state is needed, clone the vector *before* passing it to an Axis method.

## Data Pipeline
The Axis enum is not a pipeline stage itself but a **transformer tool** used within a larger data flow. It does not ingest or emit data; it modifies data that passes through it.

> **Example Flow: Block Placement**
> Player Input -> Placement Logic -> `Vector3i orientation = ...` -> **Axis.Y.rotate(orientation)** -> World State Update -> Render Packet Generation

