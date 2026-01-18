---
description: Architectural reference for Triangle4d
---

# Triangle4d

**Package:** com.hypixel.hytale.math.shape
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Triangle4d {
```

## Architecture & Concepts
The Triangle4d class is a fundamental geometric primitive within the Hytale rendering engine. It represents a single, planar triangle defined by three vertices in 4D homogeneous coordinate space. This is the canonical representation for geometry that has passed through the model, view, and projection transformation stages of the 3D rendering pipeline.

Its primary role is to serve as a container for triangle data during the intermediate stages of rendering, specifically within *clip space*. The fourth component of each vertex, *w*, is essential for handling perspective distortion. Operations performed on a Triangle4d, such as matrix multiplication and perspective transformation, are core to the process of projecting a 3D world onto a 2D screen.

This class is not intended for long-term storage of mesh data, but rather as a high-performance, mutable, "working" object used during the per-frame rendering loop.

## Lifecycle & Ownership
- **Creation:** Triangle4d instances are typically created on-demand by systems responsible for processing geometry, such as a MeshProcessor or a Rasterizer. They are often instantiated to hold the transformed vertices of a single triangle from a larger 3D model. The design, which includes methods that accept a target object, strongly encourages object reuse to mitigate garbage collection pressure in performance-critical code.

- **Scope:** The lifetime of a Triangle4d object is expected to be extremely short, often confined to the processing of a single triangle within a single frame. It is a transient object by design.

- **Destruction:** Instances are managed by the Java Garbage Collector. In high-throughput rendering scenarios, these objects are prime candidates for object pooling patterns to avoid the performance cost of frequent allocation and deallocation.

## Internal State & Concurrency
- **State:** The internal state is **highly mutable**. It consists of three Vector4d objects, which are themselves mutable. Methods like `assign`, `multiply`, and `perspectiveTransform` directly modify the vertices of the triangle in-place. This mutability is a deliberate performance choice, allowing for efficient, zero-allocation transformations.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. Concurrent modification from multiple threads will result in data corruption and undefined behavior. A Triangle4d instance must be confined to a single thread, typically the main render thread, or protected by external locking if multithreaded geometry processing is required.

## API Surface
The public API is designed for high-performance geometric manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(v1, v2, v3) | Triangle4d | O(1) | Re-initializes the triangle's vertices from the provided vectors. Allows for object reuse. |
| multiply(matrix) | Triangle4d | O(1) | Transforms the triangle by the given matrix, modifying its vertices in-place. |
| multiply(matrix, target) | Triangle4d | O(1) | Transforms this triangle, storing the result in the `target` object without modifying this instance. |
| perspectiveTransform() | Triangle4d | O(1) | Applies the perspective divide (x/w, y/w, z/w) to each vertex. This is a destructive, in-place operation. |
| getMin(component) | double | O(1) | Returns the minimum value for a given component (0=x, 1=y, 2=z, 3=w) across all three vertices. |
| getMax(component) | double | O(1) | Returns the maximum value for a given component across all three vertices. |
| to2d(target) | Triangle2d | O(1) | Projects the triangle into 2D space by discarding the z and w components. |

## Integration Patterns

### Standard Usage
The intended use is within a rendering loop where a triangle's vertices are transformed and prepared for rasterization. Reusing a pre-allocated Triangle4d object is the preferred pattern.

```java
// Pre-allocate working objects outside the main loop
Triangle4d worldTriangle = ... // From a mesh
Triangle4d clipSpaceTriangle = new Triangle4d();
Matrix4d viewProjectionMatrix = camera.getViewProjectionMatrix();

// Inside the render loop for a single triangle
worldTriangle.multiply(viewProjectionMatrix, clipSpaceTriangle);
clipSpaceTriangle.perspectiveTransform();

// ... proceed with clipping and rasterization using clipSpaceTriangle
```

### Anti-Patterns (Do NOT do this)
- **Allocation in Loops:** Avoid calling `new Triangle4d()` inside a tight loop that processes thousands of triangles per frame. This will cause significant GC pressure. Use the standard usage pattern above.
- **Shared Instances:** Do not share a single Triangle4d instance across multiple threads without explicit and robust external synchronization. It is not designed for concurrent access.
- **Premature Transformation:** Do not call `perspectiveTransform` before the triangle has been fully transformed into clip space and clipped against the view frustum. The *w* component is critical for correct clipping.

## Data Pipeline
Triangle4d acts as a critical data container in the vertex processing stage of the graphics pipeline. It holds the geometric data after it has been projected into homogeneous clip space and before it is converted to screen coordinates.

> Flow:
> 3D Mesh Data (Vector3d) -> Model-View-Projection Transform -> **Triangle4d** (Clip Space) -> Frustum Clipping -> `perspectiveTransform()` -> Viewport Transform -> Rasterization

