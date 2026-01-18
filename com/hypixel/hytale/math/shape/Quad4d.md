---
description: Architectural reference for Quad4d
---

# Quad4d

**Package:** com.hypixel.hytale.math.shape
**Type:** Transient

## Definition
```java
// Signature
public class Quad4d {
```

## Architecture & Concepts
The Quad4d class is a fundamental geometric primitive within the Hytale math library. It represents a quadrilateral defined by four vertices in 4D homogeneous coordinate space. This structure is not a high-level game object but rather a low-level data container used extensively by the rendering engine and physics systems.

Its primary role is to group four Vector4d points, allowing them to be manipulated as a single geometric entity. This is critical for operations within the graphics pipeline, such as applying matrix transformations (e.g., model, view, projection), performing frustum culling, and executing perspective division. By operating on a Quad4d, the engine can efficiently process polygonal faces of 3D models.

## Lifecycle & Ownership
- **Creation:** Quad4d instances are created on-demand, typically in tight loops within performance-sensitive systems like the renderer's culling or transformation stages. They are instantiated to temporarily hold the transformed vertices of a model's face for a single frame's calculation.
- **Scope:** The lifetime of a Quad4d object is extremely short. It is intended to be a stack-local or short-lived heap object that exists only for the duration of a specific calculation, often within a single method scope.
- **Destruction:** Instances are managed by the Java Garbage Collector and are reclaimed as soon as they are no longer referenced. There are no manual cleanup or disposal requirements.

## Internal State & Concurrency
- **State:** The internal state is composed of four mutable Vector4d vertices. The class is designed for its state to be continuously and destructively modified by transformation operations. Methods like `multiply` and `perspectiveTransform` directly alter the vertex data.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. It is designed for use within a single thread, such as the main render thread. Concurrent access and modification from multiple threads will result in race conditions and unpredictable geometric corruption. All synchronization must be handled externally by the calling system.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isFullyInsideFrustum() | boolean | O(1) | Performs a culling check. Returns true only if all four vertices are within the view frustum. |
| multiply(matrix) | Quad4d | O(1) | Transforms all four vertices by the given matrix. This is a destructive operation on the current instance. |
| multiply(matrix, target) | Quad4d | O(1) | Transforms all four vertices, storing the result in the provided target instance to prevent new allocations. |
| perspectiveTransform() | void | O(1) | Applies perspective division to each vertex (dividing x, y, z by w). This is a destructive operation. |
| getMin(component) | double | O(1) | Calculates the minimum value across all four vertices for a given component index (0=x, 1=y, 2=z, 3=w). |
| getMax(component) | double | O(1) | Calculates the maximum value across all four vertices for a given component index (0=x, 1=y, 2=z, 3=w). |

## Integration Patterns

### Standard Usage
The most common pattern is to use Quad4d as a temporary container during the rendering pipeline. To optimize performance and reduce garbage collection pressure, a "target" object is often passed to transformation methods.

```java
// Example: Transforming a quad by a Model-View-Projection matrix
Matrix4d mvpMatrix = camera.getMvpMatrix();
Quad4d modelQuad = mesh.getFace(0); // Original quad in model space
Quad4d transformedQuad = new Quad4d(); // Pre-allocate target

// Transform vertices and store result in transformedQuad
modelQuad.multiply(mvpMatrix, transformedQuad);

// Perform perspective division for screen-space coordinates
transformedQuad.perspectiveTransform();

// Now use transformedQuad for culling or rasterization...
if (!transformedQuad.isFullyInsideFrustum()) {
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not reuse a Quad4d instance across multiple independent calculations without re-initializing its vertex data. This can lead to unintended cumulative transformations.
- **Concurrent Modification:** Never share and modify a single Quad4d instance across multiple threads without explicit external locking. This will lead to data corruption.
- **Ignoring Target-based Methods:** In performance-critical loops, avoid `multiply(matrix)` which allocates a new Quad4d. Always prefer the `multiply(matrix, target)` variant to control memory allocation.

## Data Pipeline
Quad4d serves as a data carrier at a specific stage of the 3D rendering pipeline. It represents a single face after it has been assembled from raw vertex data but before it is passed to the rasterizer.

> Flow:
> 3D Model Vertex Buffer -> **Quad4d** (Face Representation) -> Model-View-Projection Transformation -> **Quad4d** (Clip Space) -> Frustum Culling -> Perspective Division -> **Quad4d** (Normalized Device Coordinates) -> Viewport Transform -> Rasterizer

