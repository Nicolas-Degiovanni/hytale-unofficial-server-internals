---
description: Architectural reference for IPointGenerator
---

# IPointGenerator

**Package:** com.hypixel.hytale.procedurallib.logic.point
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IPointGenerator {
   ResultBuffer.ResultBuffer2d nearest2D(int var1, double var2, double var4);

   ResultBuffer.ResultBuffer3d nearest3D(int var1, double var2, double var4, double var6);

   ResultBuffer.ResultBuffer2d transition2D(int var1, double var2, double var4);

   ResultBuffer.ResultBuffer3d transition3D(int var1, double var2, double var4, double var6);

   void collect(int var1, double var2, double var4, double var6, double var8, IPointGenerator.PointConsumer2d var10);

   double getInterval();

   @FunctionalInterface
   public interface PointConsumer2d {
      void accept(double var1, double var3);
   }
}
```

## Architecture & Concepts

The IPointGenerator interface defines a contract for algorithms that produce spatially distributed sets of points. It is a foundational component within the procedural generation library, responsible for providing the raw coordinate data used by higher-level systems to place features, define region boundaries, and influence terrain morphology.

This interface abstracts the underlying point generation algorithm, which could range from simple grid-based patterns to more complex methods like Poisson disk sampling or Worley noise. The core architectural purpose is to decouple the *method* of point generation from the *consumption* of those points. For example, a BiomeFeaturePlacer system does not need to know *how* tree locations are calculated; it only needs to query an IPointGenerator for a set of valid coordinates.

The API design suggests two primary query patterns:
1.  **Nearest Point Queries:** Methods like nearest2D and nearest3D are used to find the closest generated point(s) to a given coordinate. This is essential for algorithms that create cellular patterns, such as Voronoi diagrams, which are commonly used for biome map generation.
2.  **Region-based Collection:** The collect method provides a highly efficient, callback-based mechanism for iterating over all points within a specified radius. This avoids large memory allocations associated with returning a list or array, which is critical for performance in dense point clouds.

## Lifecycle & Ownership

As an interface, IPointGenerator has no intrinsic lifecycle. The lifecycle and ownership semantics apply to its concrete implementations.

-   **Creation:** Implementations are typically instantiated by a world generation orchestrator or a biome-specific generator. They are often configured with a world seed and other parameters (like point density) to ensure deterministic and repeatable generation.
-   **Scope:** The scope of an IPointGenerator instance is almost always transient and task-specific. A new instance is typically created for a discrete unit of work, such as generating a single world chunk or a specific region. It is not intended to be a long-lived, session-wide service.
-   **Destruction:** The object is eligible for garbage collection as soon as the generation task that created it is complete. There are no explicit cleanup or disposal methods on the interface, as implementations are expected to be self-contained.

## Internal State & Concurrency

-   **State:** Any concrete implementation of IPointGenerator is expected to be highly stateful. The state will include the seed for the random number generator and potentially pre-computed data or spatial acceleration structures (e.g., a grid or k-d tree) to optimize nearest-point queries. The state is considered mutable during initialization but should be treated as immutable during querying.
-   **Thread Safety:** Implementations are **not** guaranteed to be thread-safe. Procedural generation is a highly parallelized process, but the standard pattern is to provide each worker thread with its own isolated IPointGenerator instance. Sharing a single instance across multiple threads without external locking will lead to race conditions and non-deterministic, corrupted world generation.

**WARNING:** Never share a single IPointGenerator instance across threads. Always create a new instance per generation task or per thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nearest2D(count, x, y) | ResultBuffer2d | O(log N) or O(1) | Finds the N nearest 2D points to the given coordinate. Complexity depends on the underlying spatial data structure. |
| nearest3D(count, x, y, z) | ResultBuffer3d | O(log N) or O(1) | Finds the N nearest 3D points to the given coordinate. |
| transition2D(seed, x, y) | ResultBuffer2d | O(log N) or O(1) | Finds points near a boundary or edge. Used for blending regions or creating transitional features. |
| transition3D(seed, x, y, z) | ResultBuffer3d | O(log N) or O(1) | Finds 3D points near a boundary or edge. |
| collect(...) | void | O(N) | Invokes a consumer for each point within a given radius. This is the preferred method for bulk processing. |
| getInterval() | double | O(1) | Returns the characteristic spacing or minimum distance between points, a key parameter of the generation algorithm. |

## Integration Patterns

### Standard Usage

The IPointGenerator is obtained from a factory or constructed as part of a larger generation process. It is then queried to guide the placement of world elements.

```java
// A higher-level system (e.g., a BiomeGenerator) uses the interface
// to place features like trees or rocks.
IPointGenerator pointSource = generatorFactory.createPoissonGenerator(worldSeed, chunkCoords);

// Use the efficient 'collect' method for bulk placement
pointSource.collect(worldSeed, centerX, centerZ, radius, 1.0, (x, z) -> {
    world.placeTreeAt(x, z);
});
```

### Anti-Patterns (Do NOT do this)

-   **Stateful Iteration:** Do not rely on the order of points returned by any method. The internal algorithms do not guarantee a consistent iteration order between calls.
-   **Cross-Thread Sharing:** As stated previously, do not access a single IPointGenerator instance from multiple threads. This will corrupt internal state and produce unpredictable results.
-   **Inefficient Querying:** Avoid calling nearest2D or nearest3D in a tight loop to gather points in an area. Use the collect method instead, as it is optimized for this exact purpose and avoids repeated spatial lookups.

## Data Pipeline

The IPointGenerator is a critical link in the procedural generation chain, transforming a seed into spatial coordinates.

> Flow:
> World Seed & Parameters -> **IPointGenerator (Implementation)** -> Point Coordinates (via ResultBuffer or Consumer) -> Feature Placer System -> Final World Voxel Data

