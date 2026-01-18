---
description: Architectural reference for CellPointFunction
---

# CellPointFunction

**Package:** com.hypixel.hytale.procedurallib.logic.cell
**Type:** Functional Interface / Strategy

## Definition
```java
// Signature
public interface CellPointFunction {
```

## Architecture & Concepts
The CellPointFunction interface defines a behavioral contract for algorithms that generate and position feature points within a grid cell. It is a core component of the procedural generation library, specifically for cellular noise and Voronoi-based patterns.

This interface embodies the **Strategy Pattern**. It decouples the high-level cellular noise algorithm from the specific mathematical rules used to determine point locations within each cell. By providing different implementations of CellPointFunction, the engine can produce a wide variety of visual patterns (e.g., regular grids, jittered grids, random distributions) using the same underlying noise evaluation logic.

Its primary role is to translate an integer grid coordinate and a seed into a deterministic, pseudo-random hash and a set of spatial offsets. This forms the basis for calculating distances to the nearest feature point, which is the fundamental operation of cellular noise.

## Lifecycle & Ownership
As an interface, CellPointFunction itself has no lifecycle. The following applies to its concrete implementations.

- **Creation:** Implementations are typically stateless and are often instantiated as static singletons or constants within a noise utility class. For example, a `JitteredCellPointFunction` might be created once and reused across all procedural generation tasks. They are passed as configuration parameters to noise generators.
- **Scope:** The lifetime of an implementation is tied to the procedural generation context that uses it. For stateless, shared instances, the scope is the entire application lifetime.
- **Destruction:** Implementations are subject to standard Java garbage collection when they are no longer referenced by any noise generator or configuration object.

## Internal State & Concurrency
- **State:** The contract strongly implies that all implementations should be **stateless**. All methods accept the necessary input context as parameters, allowing them to operate as pure functions. Storing mutable state within an implementation is a severe anti-pattern.
- **Thread Safety:** Implementations **must be thread-safe**. The procedural generation engine is heavily parallelized, and a single CellPointFunction instance will be invoked concurrently from multiple worker threads during world generation. The stateless, pure-function nature of the contract is designed to guarantee this safety.

**WARNING:** Introducing any mutable state or synchronization into an implementation will create a significant performance bottleneck and is likely to cause severe concurrency bugs.

## API Surface
The public contract consists of mathematical transformations and point generation functions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scale(value) | double | O(1) | Default method. Applies a scaling factor to an input value. Can be overridden for custom scaling logic. |
| normalize(value) | double | O(1) | Default method. Normalizes a value, typically to a range like [0, 1]. Can be overridden. |
| getHash(x, y, z) | int | O(1) | **Critical Path.** Calculates a deterministic, pseudo-random integer hash based on 3D integer coordinates. |
| getX(hash, offset) | double | O(1) | Calculates the X-coordinate offset for a feature point within a cell, using a hash and an offset value. |
| getY(hash, offset) | double | O(1) | Calculates the Y-coordinate offset for a feature point within a cell, using a hash and an offset value. |
| getOffsets(hash) | DoubleArray.Double2 | O(1) | Generates a pair of offset values from a hash, which are then typically passed to getX and getY. |

## Integration Patterns

### Standard Usage
A concrete implementation of CellPointFunction is provided to a cellular noise generator during its construction. The generator then invokes the interface methods internally during noise evaluation.

```java
// A hypothetical noise generator receiving the strategy
// Note: This is conceptual; actual API may differ.

// 1. Obtain a specific implementation (e.g., one that creates a jittered grid)
CellPointFunction jitteredFunction = NoiseStrategies.getJitteredCellFunction();

// 2. Configure a cellular noise instance with the chosen strategy
CellularNoiseGenerator noise = new CellularNoiseGenerator(seed, jitteredFunction);

// 3. The generator uses the function internally when computing noise values
double value = noise.getValue(10.5, 20.3);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create implementations that store per-call state in member variables. This will break thread safety and lead to unpredictable generation results.
- **Expensive Computations:** Do not implement methods like getHash or getOffsets with computationally expensive logic (e.g., loops, complex allocations). These methods are called millions of times in tight loops and must be highly performant.
- **Non-Deterministic Logic:** Do not use non-deterministic sources like `Math.random()` or system time within implementations. The entire procedural pipeline relies on deterministic output for a given seed.

## Data Pipeline
The CellPointFunction acts as a transformation stage within the larger cellular noise data pipeline. It converts discrete grid coordinates into continuous-space feature point locations.

> Flow:
> Integer Grid Coordinates (x, y, z) -> **CellPointFunction.getHash** -> Integer Hash -> **CellPointFunction.getOffsets/getX/getY** -> Feature Point Offsets (dx, dy) -> Cellular Noise Algorithm -> Final Noise Value

