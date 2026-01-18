---
description: Architectural reference for Matrix4d
---

# Matrix4d

**Package:** com.hypixel.hytale.math.matrix
**Type:** Transient

## Definition
```java
// Signature
public class Matrix4d {
```

## Architecture & Concepts
The **Matrix4d** class is a fundamental data structure within the Hytale engine's mathematics library, representing a 4x4 matrix of double-precision floating-point values. It is the cornerstone for all 3D spatial transformations, including translation, rotation, scaling, and camera projections.

Architecturally, it is designed as a high-performance, mutable object with a fluent API. Most transformation methods modify the matrix's internal state directly and return a reference to the same instance, allowing for efficient chaining of operations.

The internal data is stored in a flat, 16-element array organized in **column-major order**. This layout is critical for direct compatibility with modern graphics APIs like OpenGL, which expect matrix data in this format for shader uniforms. This design choice avoids costly data transposition when communicating with the GPU.

## Lifecycle & Ownership
- **Creation:** Instances are created directly by the caller using `new Matrix4d()`. There is no factory or manager responsible for its instantiation. It is treated as a primitive data type.
- **Scope:** The lifecycle of a **Matrix4d** object is typically very short. They are often created on the stack or as temporary fields within a method, used for a specific set of calculations within a single frame (e.g., calculating a model-view-projection matrix), and then immediately become eligible for garbage collection.
- **Destruction:** Cleanup is handled entirely by the Java Garbage Collector. There are no native resources or explicit disposal methods.

## Internal State & Concurrency
- **State:** The **Matrix4d** class is highly **mutable**. Its core state is a `double[16]` array whose elements are modified by nearly all public methods. The design prioritizes performance and memory efficiency over immutability, allowing for in-place calculations that avoid repeated object allocation.

- **Thread Safety:** This class is **not thread-safe**. Concurrent modification of a single **Matrix4d** instance from multiple threads will result in a corrupted state and unpredictable behavior. Synchronization is not provided internally to avoid performance overhead.

    **WARNING:** Any **Matrix4d** instance shared between threads must be protected by external synchronization mechanisms, such as locks or concurrent data structures. It is strongly recommended to provide each thread with its own instance of a matrix for calculations.

## API Surface
The API is designed as a fluent interface for chained transformations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| identity() | Matrix4d | O(1) | Resets the matrix to the identity matrix. |
| assign(Matrix4d other) | Matrix4d | O(1) | Copies the values from another matrix into this one. |
| translate(x, y, z) | Matrix4d | O(1) | Applies a translation transformation to the matrix. |
| scale(x, y, z) | Matrix4d | O(1) | Applies a scaling transformation to the matrix. |
| multiply(Matrix4d other) | Matrix4d | O(1) | Multiplies this matrix by another, storing the result in this instance. |
| multiplyPosition(Vector3d vec) | Vector3d | O(1) | Transforms a 3D vector as a position, applying perspective division. |
| multiplyDirection(Vector3d vec) | Vector3d | O(1) | Transforms a 3D vector as a direction, ignoring translation. |
| invert() | boolean | O(1) | Inverts the matrix in-place. Returns false if the matrix is singular. |
| projectionCone(...) | Matrix4d | O(1) | Configures the matrix as a perspective projection matrix. |
| viewTarget(...) | Matrix4d | O(1) | Configures the matrix as a view (camera) matrix. |

## Integration Patterns

### Standard Usage
**Matrix4d** is intended to be used in a chain to build a final transformation matrix, which is then used to transform vertices or passed to a shader.

```java
// Example: Creating a Model-View-Projection (MVP) matrix for rendering
Matrix4d modelMatrix = new Matrix4d().identity().translate(10, 0, 5);
Matrix4d viewMatrix = new Matrix4d().viewTarget(eyeX, eyeY, eyeZ, targetX, ...);
Matrix4d projectionMatrix = new Matrix4d().projectionCone(fov, aspect, near, far);

// The order of multiplication is critical
Matrix4d mvpMatrix = new Matrix4d(projectionMatrix).multiply(viewMatrix).multiply(modelMatrix);

// mvpMatrix is now ready to be sent to the GPU
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Reusing a matrix across different logical calculations without resetting it. This leads to cumulative transformations that are extremely difficult to debug. Always call **identity()** or **assign()** before starting a new, independent transformation chain.
- **Concurrent Modification:** Sharing and modifying a single **Matrix4d** instance across multiple threads without external locking. This will lead to race conditions and corrupt matrix data.
- **External Array Modification:** Retrieving the internal array via **getData()** and modifying it externally. While possible, this breaks encapsulation and can lead to an inconsistent matrix state if not handled with extreme care.

## Data Pipeline
**Matrix4d** is a key component in the rendering data pipeline, used to transform geometric data from model space to screen space.

> Flow:
> Entity Transform (Position, Rotation) -> **Model Matrix** -> **View Matrix** -> **Projection Matrix** -> Final MVP Matrix -> GPU Shader -> Transformed Vertex Position

