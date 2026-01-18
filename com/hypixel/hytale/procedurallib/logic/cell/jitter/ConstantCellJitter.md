---
description: Architectural reference for ConstantCellJitter
---

# ConstantCellJitter

**Package:** com.hypixel.hytale.procedurallib.logic.cell.jitter
**Type:** Transient

## Definition
```java
// Signature
public class ConstantCellJitter implements CellJitter {
```

## Architecture & Concepts
ConstantCellJitter is a specific, deterministic implementation of the CellJitter strategy interface. Its primary role within the procedural generation library is to apply a uniform, non-random displacement to a point within a grid cell.

This component is fundamental for algorithms that require predictable and consistent point placement, such as creating evenly spaced grids or patterns. Unlike stochastic jittering strategies that introduce randomness, ConstantCellJitter scales a point's position by a fixed, predetermined factor. This ensures that for a given input vector, the output displacement is always identical.

It operates as a simple data-transformer, encapsulating the jittering logic and configuration (the jitter factors) into a reusable, stateless object. This design decouples the core procedural algorithms from the specific method of point displacement, allowing different jittering behaviors to be swapped in via the CellJitter interface.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its public constructor: `new ConstantCellJitter(...)`. It is typically instantiated by a higher-level configuration or factory class responsible for setting up a procedural generation pipeline.
- **Scope:** The object is lightweight and intended to be transient. Its lifetime is bound to the parent process that requires its specific jittering behavior, such as a single world generation task or the configuration of a specific biome generator.
- **Destruction:** As a simple POJO with no external resources, it is managed entirely by the Java garbage collector. It becomes eligible for collection as soon as it is no longer referenced.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state consists of the `jitterX`, `jitterY`, and `jitterZ` values. These are declared as `final` and are set only once during object construction. The object's state cannot be modified after it has been created.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a single ConstantCellJitter instance can be safely shared and accessed by multiple threads concurrently without any risk of data corruption or race conditions. No synchronization mechanisms are required. All of its methods are pure functions, meaning their output depends solely on their inputs.

## API Surface
The public API provides methods to calculate displaced coordinates based on an initial integer cell coordinate and a normalized direction vector.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPointX(int, DoubleArray.Double2/3) | double | O(1) | Calculates the final X coordinate by applying the configured jitterX factor to the input vector. |
| getPointY(int, DoubleArray.Double2/3) | double | O(1) | Calculates the final Y coordinate by applying the configured jitterY factor to the input vector. |
| getPointZ(int, DoubleArray.Double3) | double | O(1) | Calculates the final Z coordinate by applying the configured jitterZ factor to the input vector. |
| getMaxX() | double | O(1) | Returns the maximum possible displacement along the X-axis. |
| getMaxY() | double | O(1) | Returns the maximum possible displacement along the Y-axis. |
| getMaxZ() | double | O(1) | Returns the maximum possible displacement along the Z-axis. |

## Integration Patterns

### Standard Usage
ConstantCellJitter is used as a strategy object. It is instantiated with the desired jitter factors and passed to a consumer that operates on the CellJitter interface. This is a classic example of the Strategy design pattern.

```java
// Define a jitter strategy that displaces points by up to 0.5 units
// from their base cell coordinate along each axis.
CellJitter fixedJitter = new ConstantCellJitter(0.5, 0.5, 0.5);

// This strategy is then injected into a service that places points,
// ensuring predictable and uniform displacement.
PointPlacerService placer = new PointPlacerService(fixedJitter);

// The service can now use the strategy without knowing its implementation details.
Double3 finalPosition = placer.getJitteredPosition(10, 20, 30);
```

### Anti-Patterns (Do NOT do this)
- **Misuse for Randomness:** Do not use this class when random or pseudo-random point displacement is required. Its behavior is deterministic. Using it in a context expecting variety will result in rigid, grid-like patterns. Use a noise-based or random jitter implementation instead.
- **Redundant Instantiation:** Because the object is immutable and thread-safe, there is no need to create a new instance for every calculation if the jitter factors are the same. A single instance can be created once and reused throughout a process.

## Data Pipeline
This class acts as a transformation step within a larger data generation pipeline. It does not manage a pipeline itself but is a critical component within one.

> Flow:
> Integer Grid Coordinate + Normalized Vector -> **ConstantCellJitter.getPoint()** -> Final Floating-Point World Coordinate

