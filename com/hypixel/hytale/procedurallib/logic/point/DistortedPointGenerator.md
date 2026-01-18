---
description: Architectural reference for DistortedPointGenerator
---

# DistortedPointGenerator

**Package:** com.hypixel.hytale.procedurallib.logic.point
**Type:** Transient

## Definition
```java
// Signature
public class DistortedPointGenerator implements IPointGenerator {
```

## Architecture & Concepts
The DistortedPointGenerator is a concrete implementation of the **Decorator Pattern**. Its primary architectural function is to augment the behavior of an existing IPointGenerator instance without altering its class. It acts as a wrapper, intercepting coordinate-based queries and applying a spatial distortion before delegating the call to the underlying generator.

This component is fundamental to creating organic, non-uniform procedural structures. By composing a simple, predictable generator (e.g., a grid or Poisson-disc generator) with a DistortedPointGenerator, developers can introduce noise and randomness, breaking up repetitive patterns. The specific nature of the distortion is controlled by the injected ICoordinateRandomizer dependency, making the system highly modular.

For example, a uniform grid of points can be transformed into a warped, noisy point set, which can then be used as the basis for biome placement, feature distribution, or terrain variation.

### Lifecycle & Ownership
- **Creation:** Instantiated manually by a higher-level system responsible for configuring a procedural generation pipeline. It is constructed by providing a base IPointGenerator and an ICoordinateRandomizer. It is never managed by a dependency injection container.
- **Scope:** Its lifetime is typically bound to a single, specific generation task. It is a configuration object, not a long-lived service.
- **Destruction:** The object is short-lived and becomes eligible for garbage collection as soon as the reference from the parent generation process is dropped. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The DistortedPointGenerator is **immutable**. Its dependencies, pointGenerator and coordinateRandomizer, are final fields assigned at construction. It maintains no internal mutable state of its own.
- **Thread Safety:** This class is conditionally thread-safe. It introduces no synchronization mechanisms itself. Its thread safety is entirely inherited from the injected IPointGenerator and ICoordinateRandomizer instances. If the provided dependencies are thread-safe, then an instance of DistortedPointGenerator is also safe for concurrent use.

**WARNING:** Injecting stateful, non-thread-safe dependencies into a DistortedPointGenerator that will be shared across threads will lead to race conditions and non-deterministic output.

## API Surface
The public API mirrors the IPointGenerator interface. Key methods are intercepted to apply distortion, while others are passed through directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nearest2D(seed, x, y) | ResultBuffer.ResultBuffer2d | Inherited | Distorts the input (x, y) coordinates and delegates the call to the wrapped generator. |
| nearest3D(seed, x, y, z) | ResultBuffer.ResultBuffer3d | Inherited | Distorts the input (x, y, z) coordinates and delegates the call to the wrapped generator. |
| transition2D(seed, x, y) | ResultBuffer.ResultBuffer2d | Inherited | Distorts the input (x, y) coordinates and delegates the call to the wrapped generator. |
| transition3D(seed, x, y, z) | ResultBuffer.ResultBuffer3d | Inherited | Distorts the input (x, y, z) coordinates and delegates the call to the wrapped generator. |
| getInterval() | double | O(1) | Delegates directly to the wrapped generator without modification. |
| collect(...) | void | Inherited | Delegates directly to the wrapped generator. This method does *not* apply distortion. |

## Integration Patterns

### Standard Usage
The intended use is to compose generators during the setup phase of a procedural algorithm. A base generator is created first, then wrapped by the DistortedPointGenerator to add a layer of noise.

```java
// 1. Define a base point generator (e.g., a simple grid)
IPointGenerator grid = new GridPointGenerator(64.0);

// 2. Define a distortion strategy
ICoordinateRandomizer noise = new PerlinCoordinateRandomizer(0.05, 16.0);

// 3. Decorate the base generator with the distortion
IPointGenerator distortedGrid = new DistortedPointGenerator(grid, noise);

// 4. Use the final, composed generator in the algorithm
// The coordinates passed to nearest2D will be warped by Perlin noise
// before being evaluated against the base grid.
ResultBuffer.ResultBuffer2d result = distortedGrid.nearest2D(worldSeed, 350.5, -120.2);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Dependencies:** Do not inject an ICoordinateRandomizer that relies on mutable internal state if the resulting DistortedPointGenerator will be used concurrently. The behavior will be unpredictable.
- **Misordered Composition:** The order of decoration is critical. Wrapping a complex generator with distortion yields a different result than distorting a simple generator and then using it as a basis for a more complex one. Ensure the composition order matches the desired procedural outcome.

## Data Pipeline
The DistortedPointGenerator acts as a transformation step within a larger data flow. It modifies input coordinates before they reach the core point generation logic.

> Flow:
> Input Coordinates (x, y, z) -> **DistortedPointGenerator** -> ICoordinateRandomizer -> Distorted Coordinates -> Wrapped IPointGenerator -> Final ResultBuffer

