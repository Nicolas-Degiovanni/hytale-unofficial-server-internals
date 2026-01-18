---
description: Architectural reference for Rotate
---

# Rotate

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations.transform
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class Rotate implements Transform {
```

## Architecture & Concepts
The Rotate class is a concrete implementation of the Transform interface, representing a discrete, axis-aligned mathematical rotation. It serves as a fundamental component within the Builder Tools framework, providing the logic to manipulate the orientation of objects, selections, and brushes within the game world.

Architecturally, this class embodies the Command Pattern. Each Rotate instance is an immutable command object that encapsulates a specific rotation (e.g., 90 degrees around the Y-axis). This design allows the broader Builder Tools system to treat various transformations—such as rotation, translation, or scaling—polymorphically through the shared Transform interface.

A key design choice is the pre-allocation of static instances for common 90, 180, and 270-degree rotations around the cardinal axes (e.g., X_90, Y_180). This is a performance optimization that significantly reduces object allocation and garbage collection pressure during intensive building operations, where these rotations are used frequently. The composition of these transforms, seen in constants like FACING_SOUTH, further demonstrates a commitment to building complex operations from simple, reusable, and efficient primitives.

## Lifecycle & Ownership
- **Creation:** Rotate instances are primarily intended to be retrieved via static factory methods like forDirection or forAxisAndAngle, which map high-level concepts like player facing or brush settings to a specific Transform. Direct instantiation is possible but is often superseded by the use of the numerous static final instances for common rotations.
- **Scope:** These objects are typically transient and short-lived. A Rotate instance is created or retrieved, used to transform one or more vectors, and then becomes eligible for garbage collection. The static instances, however, are scoped to the application's lifetime.
- **Destruction:** Managed entirely by the Java Garbage Collector. As immutable value objects, they hold no external resources and require no manual cleanup.

## Internal State & Concurrency
- **State:** Immutable. The internal fields, axis and rotations, are final and are set exclusively within the constructor. Once a Rotate object is created, its defined transformation is fixed and cannot be altered.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. A single Rotate instance can be safely shared and used across any number of threads without external synchronization. The apply method mutates its *argument*, the Vector3i, but never its own internal state, making it safe for concurrent execution.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(Vector3i) | void | O(1) | Applies the rotation to the given vector, modifying it in-place. The operation is constant time as the loop runs at most 3 times. |
| forDirection(Vector3i, Rotation) | static Transform | O(1) | Factory method that returns a pre-defined Transform to orient an object to face a specific direction with an additional rotation. |
| forAxisAndAngle(BrushAxis, Rotation) | static Transform | O(1) | Factory method that returns a pre-defined Transform for a given brush axis and rotation angle. |

## Integration Patterns

### Standard Usage
The most common use case involves obtaining a pre-defined static transform and applying it to a vector. This is highly efficient and avoids unnecessary object creation.

```java
// A developer needs to rotate a position vector 90 degrees around the Y axis.
Vector3i myPosition = new Vector3i(10, 0, 5);

// Retrieve the efficient, pre-allocated transform.
Transform yRotation = Rotate.Y_90;

// Apply the transform. The myPosition vector is modified directly.
yRotation.apply(myPosition);

// myPosition is now (5, 0, -10)
```

### Anti-Patterns (Do NOT do this)
- **Ignoring In-Place Mutation:** The apply method modifies the vector passed to it. If the original vector is needed later, you must clone it before applying the transform. Failure to do so is a common source of bugs.
  ```java
  // BAD: The original vector is lost.
  Vector3i original = new Vector3i(1, 2, 3);
  Rotate.X_90.apply(original); // original is now mutated.

  // GOOD: Preserve the original by operating on a copy.
  Vector3i original = new Vector3i(1, 2, 3);
  Vector3i rotated = original.clone();
  Rotate.X_90.apply(rotated);
  ```
- **Unnecessary Instantiation:** Avoid creating new instances for common rotations. The static final fields are heavily optimized for this purpose.
  ```java
  // BAD: Creates a new object, causing unnecessary GC pressure.
  Transform rotation = new Rotate(Axis.X, 90);

  // GOOD: Reuses a single, static, application-wide instance.
  Transform rotation = Rotate.X_90;
  ```

## Data Pipeline
The Rotate class acts as a processing node within a larger data flow, typically initiated by a user action in the Builder Tools. It does not manage a pipeline itself but is a critical step within one.

> Flow:
> User Input (e.g., "Rotate selection 90 degrees on Y-axis") -> Builder Tool UI Event -> ToolOperation Dispatcher -> **Rotate.forAxisAndAngle(Y, Ninety)** -> **Rotate instance** is applied to each Vector3i in the selection -> World State Update -> Render Engine displays changes

