---
description: Architectural reference for Vector4d
---

# Vector4d

**Package:** com.hypixel.hytale.math.vector
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Vector4d {
```

## Architecture & Concepts
Vector4d is a fundamental data structure representing a vector in 4-dimensional homogeneous coordinate space. It is a cornerstone of the 3D rendering pipeline, used to represent both positions and directional vectors in a way that simplifies complex geometric transformations like projection, rotation, and translation.

The fourth component, **w**, is the homogeneous coordinate. Its value fundamentally changes the vector's interpretation:
- **w = 1.0**: The vector represents a **position** in 3D space. It is a point that can be translated (moved).
- **w = 0.0**: The vector represents a **direction**. It has magnitude and orientation but no location in space. It is not affected by translation operations.

This distinction is critical for correct matrix mathematics within the engine. The class is designed for high performance and low garbage collection overhead, featuring public mutable fields and methods that modify state in-place. It is not a general-purpose container but a specialized mathematical tool.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor or, more commonly, through static factory methods like newPosition or newDirection. These factories correctly initialize the **w** component for the intended use case.
- **Scope:** Vector4d instances are ephemeral and typically have a narrow scope, existing only for the duration of a calculation within a single method or frame. They are frequently used as temporary variables on the stack or as fields within more complex objects like a Transform or a Camera.
- **Destruction:** As a simple Plain Old Java Object (POJO), it is managed by the Java Garbage Collector. There are no native resources or explicit cleanup steps required. Ownership is passed by reference, and the object is eligible for collection when no longer reachable.

## Internal State & Concurrency
- **State:** The state of a Vector4d is entirely defined by its four public double-precision floating-point fields: x, y, z, and w. The class is highly **mutable**, and its fields can be modified directly. Methods like assign, lerp, and perspectiveTransform modify the instance's state in-place to prevent unnecessary object allocation.

- **Thread Safety:** This class is **not thread-safe**. Its mutable nature and lack of internal synchronization mechanisms make it suitable only for single-threaded access. If a Vector4d instance must be shared between threads, all access must be synchronized externally. It is expected to be used primarily within the main game loop or the dedicated rendering thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newPosition(x, y, z) | static Vector4d | O(1) | Factory for creating a position vector (w=1.0). |
| newDirection(x, y, z) | static Vector4d | O(1) | Factory for creating a direction vector (w=0.0). |
| assign(v) | Vector4d | O(1) | Copies component values from another vector into this one. Modifies state. |
| lerp(dest, factor, target) | Vector4d | O(1) | Performs linear interpolation, storing the result in the provided target vector. |
| perspectiveTransform() | void | O(1) | Performs the perspective divide operation (dividing x, y, z by w). Critical for converting from clip space to NDC. Modifies state. |
| isInsideFrustum() | boolean | O(1) | Checks if the transformed point lies within the canonical view volume. Used for clipping. |

## Integration Patterns

### Standard Usage
The primary use case is performing vector-matrix calculations for rendering. Developers should use the static factories to ensure the **w** component is set correctly and leverage in-place modification methods for performance.

```java
// Example: Transforming a model vertex to clip space
Vector4d modelVertex = Vector4d.newPosition(10.0, 5.0, 2.0);
Matrix4d modelViewProjectionMatrix = camera.getMvpMatrix();

// The result of the transformation is stored in a new or reused vector
Vector4d clipSpaceVertex = modelViewProjectionMatrix.transform(modelVertex, new Vector4d());

// Perform perspective divide to get Normalized Device Coordinates
if (clipSpaceVertex.w != 0) {
    clipSpaceVertex.perspectiveTransform();
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never read or write to the same Vector4d instance from multiple threads without external locking. This will lead to race conditions and corrupted geometric data.
- **Ignoring the W Component:** Failing to distinguish between positions (w=1) and directions (w=0) will result in severe visual artifacts, as translations will be incorrectly applied to directional vectors like surface normals.
- **Unexpected Mutation:** Passing a Vector4d to a method may result in its value being changed. Be aware of which engine subsystems modify vectors in-place versus returning new instances. Do not assume immutability.

## Data Pipeline
Vector4d is a key data packet in the vertex transformation pipeline within the renderer.

> Flow:
> Vertex Buffer (Vector3d) -> **Vector4d.newPosition** -> Matrix Multiplication -> **Clip Space Vector4d** -> **perspectiveTransform()** -> Normalized Device Coordinates -> Viewport Mapping -> Fragment Shader

