---
description: Architectural reference for OrthogonalProjectionProvider
---

# OrthogonalProjectionProvider

**Package:** com.hypixel.hytale.math.hitdetection.projection
**Type:** Transient

## Definition
```java
// Signature
public class OrthogonalProjectionProvider implements MatrixProvider {
```

## Architecture & Concepts
The OrthogonalProjectionProvider is a data-driven component responsible for generating a 4x4 orthogonal projection matrix. In 3D graphics, an orthogonal projection maps 3D space onto a 2D plane without the perspective distortion seen in human vision, making it ideal for 2D-style games, user interfaces, or specific rendering effects like shadow mapping.

This class serves as a concrete implementation of the MatrixProvider interface, establishing it as a pluggable source for transformation matrices within the engine's rendering and physics systems. Its primary role is to translate a set of viewing volume parameters (left, right, top, bottom, near, far) and Euler rotation angles (yaw, pitch, roll) into a final, usable Matrix4d.

A key architectural feature is the static CODEC field. This exposes the class to Hytale's data serialization system, allowing instances to be defined declaratively in asset files (e.g., JSON) and loaded at runtime. This decouples rendering and camera logic from hard-coded values, empowering designers to configure visual properties without modifying source code.

## Lifecycle & Ownership
-   **Creation:** Instances are created through two primary pathways:
    1.  **Programmatically:** Via direct instantiation with `new OrthogonalProjectionProvider()`. This is common for temporary, runtime-calculated projections.
    2.  **Declaratively:** By the engine's asset loading system, which uses the static `CODEC` to deserialize an instance from a data file. This is the expected pattern for persistent entities like cameras or lights.
-   **Scope:** The object's lifetime is bound to its owner. It is a stateful, transient object, not a global singleton. An instance might exist for the duration of a scene if part of a camera, or for only a few milliseconds if used for a single hit-detection query.
-   **Destruction:** The object is managed by the Java Garbage Collector. There are no manual cleanup or `destroy` methods. It is reclaimed once all references to it are released.

## Internal State & Concurrency
-   **State:** This class is highly mutable. It maintains internal state for the six planes of its viewing frustum and three rotation angles. It also employs a lazy evaluation pattern, caching the computed Matrix4d and using a boolean `invalid` flag to track when the matrix needs to be recalculated. Any call to a `set` method will dirty the state and trigger a re-computation on the next call to getMatrix.
-   **Thread Safety:** **This class is not thread-safe.** All public mutator methods (`setLeft`, `setRotation`, etc.) and the `getMatrix` method access and modify shared internal fields without any synchronization mechanisms. Concurrent access from multiple threads will lead to race conditions, inconsistent state, and potentially corrupted matrix data.

    **WARNING:** Instances of this class must be confined to a single thread or protected by external locking. Do not share a mutable instance across the main game thread and a rendering thread without explicit synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setLeft(double left) | OrthogonalProjectionProvider | O(1) | Sets the left clipping plane and invalidates the cached matrix. |
| setRight(double right) | OrthogonalProjectionProvider | O(1) | Sets the right clipping plane and invalidates the cached matrix. |
| setBottom(double bottom) | OrthogonalProjectionProvider | O(1) | Sets the bottom clipping plane and invalidates the cached matrix. |
| setTop(double top) | OrthogonalProjectionProvider | O(1) | Sets the top clipping plane and invalidates the cached matrix. |
| setNear(double near) | OrthogonalProjectionProvider | O(1) | Sets the near clipping plane and invalidates the cached matrix. |
| setFar(double far) | OrthogonalProjectionProvider | O(1) | Sets the far clipping plane and invalidates the cached matrix. |
| setRotation(yaw, pitch, roll) | OrthogonalProjectionProvider | O(1) | Sets the Euler rotation angles for the projection. |
| getMatrix() | Matrix4d | O(k) / O(1) | Returns the projection matrix. Complexity is constant-time O(k) on first access after invalidation and O(1) for subsequent cached reads. |

## Integration Patterns

### Standard Usage
An instance is typically configured once and its matrix is then retrieved repeatedly within a render loop. The internal caching mechanism prevents redundant calculations.

```java
// Example: Setting up a projection for a UI layer
OrthogonalProjectionProvider uiProjection = new OrthogonalProjectionProvider();

// Configure the viewing volume to match screen dimensions
uiProjection.setLeft(0)
            .setRight(1920)
            .setBottom(0)
            .setTop(1080)
            .setNear(-1.0)
            .setFar(1.0);

// In the render loop, get the matrix to set the shader uniform
Matrix4d projectionMatrix = uiProjection.getMatrix();
shader.setUniform("projectionMatrix", projectionMatrix);
```

### Anti-Patterns (Do NOT do this)
-   **Inter-thread Sharing:** Never pass a mutable instance between threads without external synchronization. The lack of internal locking makes it inherently unsafe for concurrent modification.
-   **Modification in Tight Loops:** Avoid repeatedly calling setter methods within a performance-critical loop. Each call invalidates the cache, forcing a costly matrix recalculation on the next `getMatrix` call.

    ```java
    // BAD: Causes matrix recalculation every frame
    for (int i = 0; i < 1000; i++) {
        provider.setLeft(i); // Invalidates cache
        Matrix4d m = provider.getMatrix(); // Forces re-computation
        // ... use matrix m
    }
    ```

## Data Pipeline
This class acts as a transformation node in a data pipeline, converting declarative configuration into a low-level mathematical construct used by the graphics hardware.

> Flow:
> Asset File (JSON) -> Engine Deserializer (using CODEC) -> **OrthogonalProjectionProvider** instance -> `getMatrix()` call -> `Matrix4d` -> GPU Shader Uniform

