---
description: Architectural reference for Mirror
---

# Mirror

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations.transform
**Type:** Utility / Singleton

## Definition
```java
// Signature
public class Mirror implements Transform {
```

## Architecture & Concepts
The Mirror class is a concrete implementation of the Transform strategy interface, designed to perform an in-place reflection of a 3D integer vector across a specified cardinal axis (X, Y, or Z). It serves as a fundamental mathematical primitive within the Hytale Builder Tools framework.

Architecturally, Mirror represents a single, stateless, and atomic operation. It is not a service or a manager but a pure function encapsulated in an object. Its primary role is to be composed into more complex tool operations, such as brushes or selection modifiers, which need to apply transformations to large sets of coordinates. The class pre-allocates static instances for each axis (X, Y, Z), promoting reuse and preventing unnecessary object allocation during performance-critical build operations.

## Lifecycle & Ownership
- **Creation:** The three core instances (Mirror.X, Mirror.Y, Mirror.Z) are static final fields. They are instantiated once by the JVM during class loading and exist for the entire application lifecycle. The private constructor prevents any other part of the system from creating new Mirror instances.
- **Scope:** Application-scoped. The static instances are globally accessible and shared across all threads.
- **Destruction:** The instances are garbage collected only when the class loader unloads the Mirror class, which typically occurs at JVM shutdown.

## Internal State & Concurrency
- **State:** Instances of Mirror are **immutable**. The internal state consists of a single final field, *axis*, which is set at creation time and cannot be changed.
- **Thread Safety:** The class is inherently thread-safe due to its immutability. The core *apply* method operates on a mutable Vector3i argument passed into it. Callers are responsible for ensuring that concurrent modifications to the *same* Vector3i instance are properly synchronized. The Mirror object itself can be safely shared and used by any number of threads simultaneously without locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(Vector3i) | void | O(1) | Modifies the provided vector in-place, flipping its coordinate value along the Mirror's axis. |
| forAxis(BrushAxis) | Transform | O(1) | Static factory that returns one of the pre-allocated singleton instances (X, Y, Z) or the NONE transform. |
| forDirection(Vector3i) | Transform | O(1) | Static factory that infers the desired mirror axis from a direction vector. |

## Integration Patterns

### Standard Usage
The Mirror transform is typically obtained from one of its static factory methods or constants and then applied to a coordinate. It is not intended to be instantiated directly.

```java
// A position in the world
Vector3i blockPosition = new Vector3i(10, 20, 30);

// Get the pre-defined transform for the X axis
Transform mirrorOnX = Mirror.X;

// Apply the transform. The blockPosition object is modified directly.
mirrorOnX.apply(blockPosition);

// blockPosition is now (-10, 20, 30)
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to use reflection to create new instances. Always use the static constants (Mirror.X) or factory methods (Mirror.forAxis).
- **Ignoring In-Place Mutation:** The *apply* method has a side effect: it modifies the vector object it is given. Do not assume it returns a new vector. If you need to preserve the original, you must clone it first.
- **Incorrect Factory Usage:** Using `Mirror.forDirection` with a zero vector will not produce a predictable mirror axis and may result in the `NONE` transform being returned.

## Data Pipeline
The Mirror class acts as a processing node within a larger data flow, typically initiated by a player's action with a builder tool.

> Flow:
> Builder Tool Configuration -> **Mirror.forAxis()** -> Brush Operation Logic -> **Mirror.apply(position)** -> Voxel Engine Mutation Request

