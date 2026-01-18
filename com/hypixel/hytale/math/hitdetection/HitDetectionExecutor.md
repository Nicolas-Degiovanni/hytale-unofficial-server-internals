---
description: Architectural reference for HitDetectionExecutor
---

# HitDetectionExecutor

**Package:** com.hypixel.hytale.math.hitdetection
**Type:** Transient

## Definition
```java
// Signature
public class HitDetectionExecutor {
```

## Architecture & Concepts

The HitDetectionExecutor is a high-performance, stateful calculator responsible for determining intersections between 3D models and a view ray. It is a core component of the engine's player interaction and rendering feedback systems, used for tasks like identifying the block or entity a player is looking at.

Architecturally, this class acts as a self-contained execution engine for a single, complex hit detection query. It is designed to be configured with the current rendering and world state—specifically camera matrices and line-of-sight data—and then executed against a target model.

Its primary function is to transform a model's geometry from model space into clip space, perform efficient culling and clipping against the view frustum, and then test the remaining visible geometry for line-of-sight and proximity. The core algorithm for frustum clipping is a variant of the Sutherland-Hodgman algorithm, which clips a polygon against the six planes of the canonical view volume.

This class is intentionally decoupled from the main game loop and rendering engine through the use of provider interfaces (MatrixProvider, LineOfSightProvider). This allows it to operate on data snapshots without holding direct references to volatile engine components like the Camera or World, making its behavior deterministic for a given set of inputs.

## Lifecycle & Ownership

-   **Creation:** An instance of HitDetectionExecutor is created on-demand by a system requiring a precise intersection test. It is instantiated directly using its constructor.
-   **Scope:** The lifetime of an instance is exceptionally short, typically confined to a single function call or, at most, a single frame update. It is designed to be configured, used once for a `test`, and then discarded.
-   **Destruction:** The object is managed by the Java garbage collector and becomes eligible for collection as soon as it falls out of scope. There are no manual cleanup or `close` methods.

## Internal State & Concurrency

-   **State:** The HitDetectionExecutor is highly mutable and stateful. It maintains internal instances of matrices (pvmMatrix, invPvMatrix), vectors, and a complex HitDetectionBuffer which serves as a scratchpad for intermediate calculations like transformed vertices and clipped polygons. This state is aggressively modified during each `test` operation.

-   **Thread Safety:** This class is **not thread-safe**. Its internal state is modified without any synchronization primitives. Concurrent calls to the `test` method on the same instance will result in data corruption, race conditions, and unpredictable behavior.

    **WARNING:** Each thread requiring hit detection logic must create and manage its own private instance of HitDetectionExecutor. Do not share instances across threads. The inclusion of the thread name in the exception logging is a deliberate indicator of this design constraint.

## API Surface

The public API is designed as a fluent builder for configuration, followed by a single execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setProjectionProvider(provider) | HitDetectionExecutor | O(1) | Sets the provider for the camera's projection matrix. |
| setViewProvider(provider) | HitDetectionExecutor | O(1) | Sets the provider for the camera's view matrix. |
| setLineOfSightProvider(provider) | HitDetectionExecutor | O(1) | Sets the callback used to verify if a clear path exists to a potential hit. |
| setMaxRayTests(count) | HitDetectionExecutor | O(1) | Caps the number of quads to test per model to prevent performance issues. |
| setOrigin(x, y, z) | HitDetectionExecutor | O(1) | Defines the world-space starting point of the ray (e.g., the player's eye position). |
| test(point, modelMatrix) | boolean | O(1) | Executes a hit test against a single world-space point. |
| test(model, modelMatrix) | boolean | O(N) | Executes a full hit test against a model's quads. N is `min(model.length, maxRayTests)`. |
| getHitLocation() | Vector4d | O(1) | Returns the world-space position of the closest successful hit. |

## Integration Patterns

### Standard Usage

The intended pattern is to create a new executor, configure it using the fluent setters, execute the test, and then retrieve the result. The executor should be discarded after use.

```java
// How a developer should normally use this
HitDetectionExecutor executor = new HitDetectionExecutor();

// Configure with current engine state
executor.setProjectionProvider(camera::getProjectionMatrix)
        .setViewProvider(camera::getViewMatrix)
        .setLineOfSightProvider(world::hasLineOfSight)
        .setOrigin(player.getEyePosition());

// Execute against a target entity's model
boolean hit = executor.test(entity.getModelQuads(), entity.getModelMatrix());

if (hit) {
    Vector4d location = executor.getHitLocation();
    // ... process the hit
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not cache and re-use a HitDetectionExecutor instance across multiple frames or distinct logical tests. Its internal state is not reset between `test` calls, which can lead to stale data influencing new calculations. Always create a fresh instance.
-   **Concurrent Access:** Never share an instance between threads. This will cause severe and difficult-to-debug state corruption.
-   **Incomplete Configuration:** Calling `test` without first providing the projection and view matrices via their setters will result in a NullPointerException.
-   **Premature Result Access:** Do not call `getHitLocation` before a `test` method has been invoked and returned `true`. The contents of the returned vector will be undefined.

## Data Pipeline

The `test(Quad4d[], Matrix4d)` method orchestrates a complex computational pipeline to transform and test model geometry.

> Flow:
> 1.  **Input:** Camera Matrices, Line of Sight Provider, Model Geometry, Model Transform, Origin Point.
> 2.  **Matrix Setup:** The provided Projection and View matrices are combined. The inverse of this PV matrix is calculated and cached for un-projection.
> 3.  **Per-Quad Loop:** The executor iterates through each quad of the input model.
> 4.  **MVP Transform:** The quad's vertices are transformed from model space to clip space using the full Model-View-Projection matrix.
> 5.  **Frustum Culling & Clipping:** The transformed quad is tested against the view frustum. If it is partially inside, its geometry is clipped to the frustum boundaries, producing a new, smaller polygon.
> 6.  **Hit Point Selection:** A random point is selected on the surface of the visible, clipped polygon.
> 7.  **Un-projection:** The selected clip-space hit point is transformed back into world space using the cached inverse PV matrix.
> 8.  **Line of Sight Test:** The external `LineOfSightProvider` is invoked to check for obstructions between the origin and the world-space hit point.
> 9.  **Proximity Check:** If the line of sight is clear, the distance to the hit is calculated. If it is the closest valid hit found so far, it is stored.
> 10. **Output:** The method returns `true` if any valid hit was found. The closest valid hit is then available via `getHitLocation`.

