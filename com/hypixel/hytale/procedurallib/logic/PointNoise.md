---
description: Architectural reference for PointNoise
---

# PointNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Transient

## Definition
```java
// Signature
public class PointNoise implements NoiseFunction {
```

## Architecture & Concepts
PointNoise is a deterministic distance field generator that conforms to the NoiseFunction interface. Unlike traditional noise algorithms like Perlin or Simplex, it does not produce random-looking patterns. Instead, it generates a smooth, spherical gradient centered at a specific point in 3D space.

Its primary role in the procedural generation engine is to create zones of influence. It outputs a value that transitions from -1.0 (inside an *inner radius*) to +1.0 (outside an *outer radius*), with a linear interpolation in the shell between the two radii. This makes it a fundamental building block for features requiring a radial falloff, such as:
- Defining the boundaries of a spherical cave system.
- Creating a mask to blend one biome into another around a central point.
- Controlling the density of vegetation or other features based on proximity to a point of interest.

By implementing the NoiseFunction interface, instances of PointNoise can be seamlessly integrated into the larger procedural pipeline. They can be combined with other noise functions using operators (e.g., Add, Multiply, Select) to create highly complex and structured world features.

**WARNING:** The `seed` and `seedOffset` parameters required by the NoiseFunction interface are intentionally ignored. The output of PointNoise is entirely determined by its construction parameters and the input coordinates, making it fully deterministic.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor by higher-level procedural logic, such as a `ZoneGenerator` or a `FeaturePlacer`. It is not managed by a central service or registry.
- **Scope:** Short-lived. An instance of PointNoise typically exists only for the duration of a specific generation task (e.g., generating a single chunk or a specific world feature). It is a value object, not a persistent service.
- **Destruction:** Becomes eligible for garbage collection as soon as the procedural task that created it completes and drops its reference.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are final and are set exclusively within the constructor. Key values, such as the squared radii and the inverse range, are pre-calculated at instantiation to optimize the performance of the `get` methods.
- **Thread Safety:** **Fully thread-safe**. Its immutable nature guarantees that a single PointNoise instance can be safely accessed by multiple worker threads simultaneously without any external locking or synchronization. This is critical for the engine's parallelized world generation architecture.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PointNoise(x, y, z, inner, outer) | constructor | O(1) | Creates a new distance field generator. |
| get(seed, offset, x, y) | double | O(1) | Samples the 2D distance field. Ignores seed and offset. |
| get(seed, offset, x, y, z) | double | O(1) | Samples the 3D distance field. Ignores seed and offset. |

## Integration Patterns

### Standard Usage
PointNoise is used as a concrete implementation of the NoiseFunction interface, often to be combined with other noise sources.

```java
// Define a spherical zone of influence centered at (512, 64, 1024).
// The core (value -1.0) has a radius of 20 blocks.
// The influence fades to neutral (value 1.0) at a radius of 50 blocks.
NoiseFunction sphere = new PointNoise(512.0, 64.0, 1024.0, 20.0, 50.0);

// This 'sphere' object can now be passed to other procedural systems,
// such as a terrain carver or a biome selector, which operate on the
// generic NoiseFunction interface.
double value = sphere.get(0, 0, 530.0, 64.0, 1024.0);
```

### Anti-Patterns (Do NOT do this)
- **Expecting Randomness:** Do not use this class expecting pseudo-random noise. It is a geometric function. Attempting to get different results by changing the seed will have no effect. To create variation, you must instantiate a new PointNoise with different center coordinates or radii.
- **Zero-Width Falloff:** Constructing with `innerRadius` equal to `outerRadius` is technically valid but creates an infinitely sharp transition from -1.0 to 1.0. This hard edge can produce severe aliasing and undesirable visual artifacts in generated terrain. A falloff range greater than zero is almost always required.
- **Performance Misconception:** Do not re-instantiate PointNoise inside a tight loop (e.g., per-voxel). It is designed to be created once per feature and reused for all sampling points within that feature's scope.

## Data Pipeline
PointNoise acts as a generator node in the procedural data pipeline. It does not transform incoming data streams; it originates a new data stream based on spatial queries.

> Flow:
> Feature Configuration (Center, Radii) -> **new PointNoise()** -> World Generator calls `get(x,y,z)` -> **double value [-1.0, 1.0]** -> Downstream Noise Combiner / Voxel Carver

