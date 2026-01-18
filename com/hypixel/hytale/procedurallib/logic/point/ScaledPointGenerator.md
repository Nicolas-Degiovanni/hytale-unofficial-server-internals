---
description: Architectural reference for ScaledPointGenerator
---

# ScaledPointGenerator

**Package:** com.hypixel.hytale.procedurallib.logic.point
**Type:** Transient / Decorator

## Definition
```java
// Signature
public class ScaledPointGenerator implements IPointGenerator {
```

## Architecture & Concepts
The ScaledPointGenerator is an implementation of the **Decorator** design pattern. It does not generate points itself; instead, it wraps an existing PointGenerator instance and transforms the coordinate space before and after delegation. This component is fundamental to the procedural generation system, enabling the reuse of a single point distribution algorithm (e.g., Worley noise, grid) at various scales or resolutions across the game world.

Its primary function is to act as a coordinate space transformer. When a request for a point is made (e.g., nearest2D), the ScaledPointGenerator first multiplies the input coordinates by its internal scale factor. It then passes these scaled coordinates to the wrapped PointGenerator. The result, which is in the scaled coordinate space, is then transformed back by dividing its coordinates and distances by the same scale factor.

A critical detail is the handling of distance. The underlying generators often return squared distances for performance reasons (avoiding expensive square root operations). This class correctly accounts for this by taking the square root of the returned distance *before* dividing by the scale, ensuring the final distance value is linearly correct in the original coordinate space.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its public constructor: `new ScaledPointGenerator(generator, scale)`. It is typically created by higher-level procedural controllers, such as a biome or feature generator, that need to sample a noise field at a specific frequency.
- **Scope:** Transient. The lifetime of a ScaledPointGenerator is short and tied to the specific generation task for which it was created. It is not a long-lived service or singleton.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It is eligible for collection as soon as the orchestrating generator that created it goes out of scope.

## Internal State & Concurrency
- **State:** The internal state, consisting of the wrapped PointGenerator and the scale factor, is **immutable**. Both fields are declared final and are set only once during construction. The object itself is stateless after initialization.
- **Thread Safety:** This class contains no internal locks or synchronization primitives. Its thread safety is entirely dependent on the thread safety of the wrapped PointGenerator instance provided during construction. If the wrapped generator is thread-safe, this class can be safely used across multiple threads.

    **Warning:** Passing a non-thread-safe PointGenerator to this class and then sharing the ScaledPointGenerator instance across threads will lead to race conditions and undefined behavior. The caller is responsible for ensuring the thread safety of all composed objects.

## API Surface
The API mirrors the IPointGenerator interface, with each method performing a coordinate transformation before and after delegating to the wrapped instance.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nearest2D(seed, x, y) | ResultBuffer2d | O(N) | Finds the nearest point in 2D space, transforming coordinates. Complexity is determined by the wrapped generator. |
| nearest3D(seed, x, y, z) | ResultBuffer3d | O(N) | Finds the nearest point in 3D space, transforming coordinates. Complexity is determined by the wrapped generator. |
| transition2D(seed, x, y) | ResultBuffer2d | O(N) | Finds the two nearest points for cell boundary transitions. Complexity is determined by the wrapped generator. |
| transition3D(seed, x, y, z) | ResultBuffer3d | O(N) | Finds the two nearest points for cell boundary transitions. Complexity is determined by the wrapped generator. |
| getInterval() | double | O(1) | Returns the interval of the wrapped generator, adjusted by the scale factor. |
| collect(...) | void | O(N) | Collects all points within a given bounding box, transforming coordinates appropriately. |

## Integration Patterns

### Standard Usage
The primary use case is to wrap a base generator to change its feature size. This is common when layering different noise functions for terrain generation.

```java
// How a developer should normally use this
// 1. Create a base generator (e.g., Worley noise)
PointGenerator baseWorleyNoise = new PointGenerator(PointGenerator.WORLEY_2D);

// 2. Wrap it to create a larger-scale version for biome boundaries
IPointGenerator largeScaleFeatures = new ScaledPointGenerator(baseWorleyNoise, 0.01);

// 3. Wrap it again with a different scale for small details
IPointGenerator smallScaleDetails = new ScaledPointGenerator(baseWorleyNoise, 0.5);

// 4. Use the scaled generators to sample points
ResultBuffer.ResultBuffer2d biomeEdge = largeScaleFeatures.nearest2D(seed, x, y);
ResultBuffer.ResultBuffer2d rockPlacement = smallScaleDetails.nearest2D(seed, x, y);
```

### Anti-Patterns (Do NOT do this)
- **Chaining Scaled Generators:** Do not wrap a ScaledPointGenerator with another ScaledPointGenerator. This is inefficient and mathematically redundant. Instead, compose the scales into a single instance.
    - **BAD:** `new ScaledPointGenerator(new ScaledPointGenerator(gen, 0.5), 0.1);`
    - **GOOD:** `new ScaledPointGenerator(gen, 0.05);`
- **Invalid Scale Factor:** Providing a scale factor of zero will result in `DivisionByZeroException`. Negative scale factors are not meaningful and will produce inverted, undefined results. Always use a positive scale factor.
- **Redundant Scaling:** Using a scale factor of 1.0 creates a useless wrapper that adds minor performance overhead. If no scaling is needed, use the base PointGenerator directly.

## Data Pipeline
The flow of data for any query method (like nearest2D) follows a distinct "transform -> delegate -> inverse transform" pattern.

> Flow:
> Input World Coordinates -> **ScaledPointGenerator** (multiplies coords by scale) -> Wrapped PointGenerator -> ResultBuffer (in scaled space) -> **ScaledPointGenerator** (divides result coords and distance by scale) -> Output ResultBuffer (in world space)

