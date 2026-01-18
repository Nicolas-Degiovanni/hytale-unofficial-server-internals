---
description: Architectural reference for PointProvider
---

# PointProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.fields.points
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface PointProvider {
```

## Architecture & Concepts
The PointProvider interface defines a formal contract for algorithms that generate collections of coordinates within a specified N-dimensional boundary. It serves as a fundamental abstraction within the world generation pipeline, decoupling feature placement logic from the specific strategy used to determine placement locations.

This abstraction allows world generation systems, such as biome populators or structure placers, to request a set of points without needing to know *how* those points are generated. Implementations can range from simple grid-based layouts (GridPointProvider) to complex procedural distributions like Poisson disk sampling (PoissonPointProvider), all conforming to this single interface.

A key architectural feature is the dual-method pattern for point retrieval:
1.  **List-based retrieval:** Methods returning a List are provided for convenience and are suitable for scenarios where the number of generated points is small and known to be within reasonable memory limits.
2.  **Stream-based retrieval:** Methods accepting a Consumer are designed for high-performance, large-scale generation. This pattern avoids allocating a potentially massive list in memory, instead streaming each point directly to the caller's processing logic. This is the preferred approach for performance-critical code paths like chunk generation.

The interface provides comprehensive support for 1D, 2D, and 3D spaces with both integer (Vector3i, Vector2i) and double-precision (Vector3d, Vector2d) vector types.

## Lifecycle & Ownership
As an interface, PointProvider itself has no lifecycle. The following applies to its concrete implementations.

-   **Creation:** Implementations are typically instantiated by higher-level world generation services or field functions. They are often created on-the-fly to fulfill a specific generation task. For example, a BiomePopulator might create a RandomPointProvider configured with a specific seed for a single chunk.
-   **Scope:** The lifetime of a PointProvider implementation is almost always transient. It exists only for the duration of the operation that requires it, such as populating a single chunk or region. It is not intended to be a long-lived service.
-   **Destruction:** Instances are eligible for garbage collection as soon as the generation task completes and all references to the object are released. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** This interface defines no state. However, concrete implementations are expected to be stateful. For instance, a provider for random points would encapsulate a Random instance and its seed. The contract implies that for a given input and a given state, the output should be deterministic.
-   **Thread Safety:** **Not thread-safe.** The PointProvider contract makes no guarantees about thread safety. Implementations are assumed to be stateful and unsafe for concurrent use. Callers are responsible for ensuring that a single instance is not accessed from multiple threads without external synchronization.

    **WARNING:** Using a single PointProvider instance across multiple world generation worker threads will lead to race conditions and non-deterministic, corrupted output. Each thread or task should use its own dedicated instance.

## API Surface
The API is composed of pairs of methods for each dimensionality and data type. One method in each pair returns a fully materialized List, while the other uses a Consumer for memory-efficient streaming.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| points3i(min, max) | List<Vector3i> | O(N) | Returns a list of all 3D integer points within the bounding box. **WARNING:** High memory cost for large volumes. |
| points3i(min, max, consumer) | void | O(N) | Streams all 3D integer points to the provided consumer. Preferred for performance. |
| points2i(min, max) | List<Vector2i> | O(N) | Returns a list of all 2D integer points within the bounding box. |
| points2i(min, max, consumer) | void | O(N) | Streams all 2D integer points to the provided consumer. |
| points1i(min, max) | List<Integer> | O(N) | Returns a list of all 1D integer points within the range. |
| points1i(min, max, consumer) | void | O(N) | Streams all 1D integer points to the provided consumer. |
| points3d(...) | List<Vector3d> | O(N) | Provides double-precision 3D points. Overloads exist for List and Consumer patterns. |
| points2d(...) | List<Vector2d> | O(N) | Provides double-precision 2D points. Overloads exist for List and Consumer patterns. |
| points1d(...) | List<Double> | O(N) | Provides double-precision 1D points. Overloads exist for List and Consumer patterns. |

*N represents the total number of points generated by the specific implementation within the given bounds.*

## Integration Patterns

### Standard Usage
The primary use case is within a world generation function to acquire locations for placing features. The high-performance Consumer pattern is strongly recommended for any non-trivial generation task.

```java
// A FieldFunction obtains a specific provider implementation
PointProvider pointProvider = getGridPointProviderForChunk(chunkPos);
Vector3i min = chunkPos.toWorldCoords();
Vector3i max = min.add(CHUNK_SIZE, CHUNK_SIZE, CHUNK_SIZE);

// Use the memory-efficient streaming pattern to place features
pointProvider.points3i(min, max, (point) -> {
    if (canPlaceTreeAt(point)) {
        world.setBlock(point, Block.TREE_TRUNK);
    }
});
```

### Anti-Patterns (Do NOT do this)
-   **Requesting Large Lists:** Avoid calling the List-returning variants for large volumes. This can easily cause an OutOfMemoryError and stall the world generator. Always favor the Consumer-based methods for chunk-scale or larger operations.
    ```java
    // BAD: This might allocate a list with millions of vectors.
    List<Vector3i> points = provider.points3i(regionMin, regionMax);
    ```
-   **Assuming Distribution:** Do not write logic that relies on the specific distribution pattern of an underlying implementation (e.g., assuming points are always sorted or on a grid). The purpose of the interface is to abstract this away.
-   **Shared Instances Across Threads:** Never share a single PointProvider instance between multiple worker threads without explicit locking. This will result in corrupted state and unpredictable generation.

## Data Pipeline
PointProvider acts as a source node in a localized data flow during world generation. It does not process incoming data but rather originates it based on configuration and spatial bounds.

> Flow:
> World Generator Task -> Requests points for a BoundingBox -> **PointProvider Implementation** (e.g., GridPointProvider) -> Generates and streams Vector3i coordinates -> Consumer Lambda (Feature Placer) -> Mutates World Voxel Buffer

