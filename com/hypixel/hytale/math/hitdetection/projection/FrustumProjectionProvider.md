---
description: Architectural reference for FrustumProjectionProvider
---

# FrustumProjectionProvider

**Package:** com.hypixel.hytale.math.hitdetection.projection
**Type:** Transient

## Definition
```java
// Signature
public class FrustumProjectionProvider implements MatrixProvider {
```

## Architecture & Concepts
The FrustumProjectionProvider is a stateful component responsible for generating a 4x4 projection matrix (Matrix4d) that defines a 3D viewing frustum. In 3D graphics and physics, a frustum is a truncated pyramid that represents the visible or collidable volume of space from a specific viewpoint.

This class serves as a concrete implementation of the MatrixProvider interface, abstracting the complex mathematics of frustum projection into a configurable object. Its primary role is to translate a set of dimensional and rotational parameters (left, right, top, bottom, near, far, yaw, pitch, roll) into a single matrix that can be consumed by the rendering pipeline or physics engine.

A key architectural feature is its use of a dirty-flag optimization. The internal matrix is only recalculated when one of its dimensional parameters changes, signaled by the internal *invalid* flag. This prevents redundant, computationally expensive matrix operations on every request, making it efficient for use in performance-critical loops.

Furthermore, the class includes a static CODEC field, indicating deep integration with Hytale's serialization system. This allows frustum configurations to be defined in external data files (e.g., JSON) and loaded at runtime, decoupling camera and projection settings from game logic.

## Lifecycle & Ownership
-   **Creation:** An instance is created either directly via its constructor (`new FrustumProjectionProvider()`) for programmatic setup, or more commonly, it is instantiated by the Hytale Codec system when deserializing engine or game assets.
-   **Scope:** The lifetime of a FrustumProjectionProvider is bound to its owner. It is typically a component of a higher-level object, such as a Camera or a Culling Volume, and persists as long as its owner does. It is not a global singleton.
-   **Destruction:** The object is managed by the Java garbage collector and is reclaimed when all references to it are released. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This class is highly mutable. It maintains the six planes of the frustum, three rotation angles, and a boolean *invalid* flag. Its most critical state is the cached Matrix4d, which is lazily recomputed when the object is marked as invalid.
-   **Thread Safety:** **This class is not thread-safe.** Access to an instance must be externally synchronized or confined to a single thread. The `getMatrix` method performs a check-then-act operation on the *invalid* flag while modifying internal state, which creates a classic race condition. Concurrent calls to setters and `getMatrix` will result in unpredictable behavior, including corrupted matrices and rendering artifacts.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setLeft(double left) | FrustumProjectionProvider | O(1) | Sets the left clipping plane coordinate and invalidates the cached matrix. |
| setRight(double right) | FrustumProjectionProvider | O(1) | Sets the right clipping plane coordinate and invalidates the cached matrix. |
| setBottom(double bottom) | FrustumProjectionProvider | O(1) | Sets the bottom clipping plane coordinate and invalidates the cached matrix. |
| setTop(double top) | FrustumProjectionProvider | O(1) | Sets the top clipping plane coordinate and invalidates the cached matrix. |
| setNear(double near) | FrustumProjectionProvider | O(1) | Sets the near clipping plane coordinate and invalidates the cached matrix. |
| setFar(double far) | FrustumProjectionProvider | O(1) | Sets the far clipping plane coordinate and invalidates the cached matrix. |
| setRotation(double yaw, double pitch, double roll) | FrustumProjectionProvider | O(1) | Sets the Euler rotation angles. Does not invalidate the matrix directly. |
| getMatrix() | Matrix4d | O(N) / O(1) | Returns the projection matrix. Complexity is O(N) if the matrix is invalid (triggering recalculation) and O(1) if it is cached. |

## Integration Patterns

### Standard Usage
The intended use is to configure the provider once, or infrequently, and then call `getMatrix` repeatedly within a render or physics loop.

```java
// Typically obtained as a component of a camera or scene object
FrustumProjectionProvider projection = camera.getProjectionProvider();

// Configure the viewing volume, for example, based on screen aspect ratio
projection.setLeft(-aspectRatio)
          .setRight(aspectRatio)
          .setBottom(-1.0)
          .setTop(1.0)
          .setNear(0.1)
          .setFar(1000.0);

// In the render loop, retrieve the matrix to pass to a shader
Matrix4d projMatrix = projection.getMatrix();
shader.setUniform("projectionMatrix", projMatrix);
```

### Anti-Patterns (Do NOT do this)
-   **External Matrix Modification:** The `getMatrix` method returns a direct reference to the internal, cached matrix. Modifying this returned object externally will corrupt the provider's state and lead to severe visual bugs. If you need to modify the matrix, create a copy.
-   **Concurrent Access:** Do not share a single FrustumProjectionProvider instance across multiple threads without explicit locking. Calling a setter from a logic thread while the render thread is calling `getMatrix` is an unsupported and dangerous operation.
-   **Redundant Invalidation:** Avoid calling setters with the same values repeatedly within a loop. While the `getMatrix` call is optimized, each setter call marks the object as invalid, forcing an unnecessary recalculation on the next `getMatrix` call.

## Data Pipeline
This class acts as a data source or generator, not a pipeline stage. It transforms configuration parameters into a complex data structure (Matrix4d) for consumption by other systems.

> Flow:
> Configuration (Setter calls OR Deserialization via CODEC) -> **FrustumProjectionProvider** (Internal State) -> `getMatrix()` -> Matrix4d -> Render Pipeline / Physics Engine

