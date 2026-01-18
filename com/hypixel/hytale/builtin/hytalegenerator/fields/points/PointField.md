---
description: Architectural reference for PointField
---

# PointField

**Package:** com.hypixel.hytale.builtin.hytalegenerator.fields.points
**Type:** Transient (Abstract Base Class)

## Definition
```java
// Signature
public abstract class PointField implements PointProvider {
```

## Architecture & Concepts
PointField is an abstract base class that serves as a foundational component within the world generation system. It establishes a standardized contract for algorithms that distribute points within a given N-dimensional space.

Its primary architectural role is to decouple the specific point distribution logic (e.g., grid, random, Poisson disc) from the coordinate space transformation (scaling). Concrete subclasses, such as GridPointField or PoissonPointField, are responsible for implementing the core generation algorithm. This class provides the shared infrastructure for applying a uniform or non-uniform scale to the output coordinates.

By implementing the PointProvider interface, instances of PointField and its subclasses become interchangeable components within the broader generation pipeline, allowing different point distribution strategies to be plugged into feature placers or biome generators without altering the consuming code.

## Lifecycle & Ownership
- **Creation:** A concrete subclass of PointField is instantiated by a higher-level generator, such as a BiomeGenerator or FeaturePlacer, when it needs to determine locations for objects like trees, rocks, or structures. Direct instantiation of PointField is not possible as it is an abstract class.
- **Scope:** The lifetime of a PointField instance is strictly bound to a single, specific generation task. It is a short-lived object used to produce a set of coordinates and is then discarded.
- **Destruction:** The object becomes eligible for garbage collection as soon as the generation task completes and the returned list of points is consumed. It holds no persistent state and is not managed by any engine-level registry.

## Internal State & Concurrency
- **State:** PointField is a mutable, stateful class. Its primary state consists of the protected scaling factors: scaleX, scaleY, scaleZ, and scaleW. These values are directly modified by the public setScale methods.
- **Thread Safety:** **This class is not thread-safe.** The internal scale fields are accessed and modified without any synchronization mechanisms.

    **WARNING:** Sharing a single PointField instance across multiple threads for concurrent generation tasks will result in race conditions and non-deterministic output. Each worker thread in a parallelized world generator must create and manage its own distinct PointField instances.

## API Surface
The public API primarily consists of two categories: fluent scale configuration and point generation requests. The generation methods delegate the core logic to abstract methods that must be implemented by subclasses, while providing the boilerplate for list creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| points3i(min, max) | List<Vector3i> | O(N) | Generates a list of 3D integer points within a bounding box. Complexity depends on the subclass implementation, where N is the number of points. |
| points2d(min, max) | List<Vector2d> | O(N) | Generates a list of 2D double-precision points within a bounding box. |
| setScale(scale) | PointField | O(1) | Sets a uniform scale for all axes. Returns the instance for fluent chaining. |
| setScale(x, y, z, w) | PointField | O(1) | Sets a non-uniform scale for each axis. Returns the instance for fluent chaining. |

## Integration Patterns

### Standard Usage
The intended pattern is to instantiate a concrete subclass, configure its scale, and then request a list of points for a specific region. The instance is typically used once and then discarded.

```java
// Assume GridPointField is a concrete subclass
// 1. Instantiate a specific implementation
PointField pointField = new GridPointField();

// 2. Configure its state using the fluent API
pointField.setScale(16.0, 32.0, 16.0, 1.0);

// 3. Request points for a given chunk or region
Vector3i minBounds = new Vector3i(0, 0, 0);
Vector3i maxBounds = new Vector3i(16, 256, 16);
List<Vector3i> locations = pointField.points3i(minBounds, maxBounds);

// 4. Use the generated locations to place features
for (Vector3i loc : locations) {
    world.placeTree(loc);
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation During Generation:** Do not modify the scale of a PointField instance after it has been passed to a system that may be actively using it. This can lead to inconsistent point distributions within a single operation.
- **Cross-Thread Sharing:** Never share a single PointField instance between multiple worker threads. Each thread must have its own private instance to prevent data races on the internal scale fields.
- **Caching Instances:** Caching and reusing PointField instances is generally discouraged due to their mutable state. It is safer and clearer to create a new instance for each distinct generation task.

## Data Pipeline
PointField acts as a data source or *provider* in the world generation pipeline. It does not process incoming data; rather, it originates coordinate data based on its internal algorithm and configuration.

> Flow:
> Generator Configuration -> **Concrete PointField Subclass** (Instantiation & Scaling) -> `pointsXi()` call -> List of Vectors -> Feature Placer / Voxel Engine

