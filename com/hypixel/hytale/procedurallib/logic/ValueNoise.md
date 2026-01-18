---
description: Architectural reference for ValueNoise
---

# ValueNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Transient

## Definition
```java
// Signature
public class ValueNoise implements NoiseFunction {
```

## Architecture & Concepts
ValueNoise is a concrete implementation of the NoiseFunction interface, designed to generate classic "value noise". This algorithm forms a foundational layer within the procedural generation engine.

Architecturally, this class operates as a configurable noise provider. Its core design principle is the decoupling of the noise generation logic from the interpolation method. This is achieved through the **Strategy Pattern**, where an InterpolationFunction is injected via the constructor. This allows developers to switch between interpolation algorithms (e.g., linear, cubic, cosine) without altering the core noise generation code, thereby changing the visual characteristics of the resulting noise from sharp and blocky to smooth and organic.

ValueNoise works by generating pseudo-random values at integer coordinates on a grid and then smoothly interpolating between these points. It is a fundamental building block used by higher-level systems to create natural-looking features like terrain elevation, biome distribution, and cave systems.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its public constructor: `new ValueNoise(...)`. It is not managed by a dependency injection framework or a central registry. Typically, a higher-level generator, such as a WorldGenerator or BiomeProvider, will instantiate it during its own initialization phase.
- **Scope:** The lifetime of a ValueNoise object is bound to its owner. It is a lightweight, stateful-but-immutable object that persists as long as the parent generator requires it. It is not a global singleton.
- **Destruction:** The object is eligible for garbage collection once the owning generator is discarded and no other references remain. It does not manage any unmanaged resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state of ValueNoise is **Immutable**. Its only field, interpolationFunction, is final and is set exclusively at construction time. The class does not cache results or modify its state during method execution.
- **Thread Safety:** This class is unconditionally **Thread-Safe**. Its immutable nature and the fact that its `get` methods are pure functions (output depends only on input) guarantee that it can be safely shared across multiple threads without locks or synchronization. This is a critical feature for enabling parallelized and high-performance world generation.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ValueNoise(interpolationFunction) | constructor | O(1) | Constructs a new noise generator with the specified interpolation strategy. |
| get(seed, offsetSeed, x, y) | double | O(1) | Computes the 2D noise value at the given coordinates. The calculation is constant time. |
| get(seed, offsetSeed, x, y, z) | double | O(1) | Computes the 3D noise value at the given coordinates. The calculation is constant time. |
| getInterpolationFunction() | InterpolationFunction | O(1) | Returns the interpolation function configured at construction. |

## Integration Patterns

### Standard Usage
The class is intended to be instantiated once with a desired configuration and reused for all subsequent noise sampling.

```java
// 1. Select an interpolation strategy
GeneralNoise.InterpolationFunction interp = GeneralNoise.CUBIC;

// 2. Create the noise generator instance
NoiseFunction valueNoise = new ValueNoise(interp);

// 3. Sample the noise field at a specific coordinate
int worldSeed = 12345;
int featureOffset = 100;
double noiseValue = valueNoise.get(worldSeed, featureOffset, 150.75, -88.2);
```

### Anti-Patterns (Do NOT do this)
- **Re-instantiation in a Loop:** Avoid creating a new ValueNoise instance for every noise sample. While the object is lightweight, repeated instantiation inside a tight loop (e.g., a `for` loop iterating over world blocks) is inefficient and creates unnecessary garbage collector pressure.

  ```java
  // BAD: Creates thousands of objects
  for (int x = 0; x < 100; x++) {
      NoiseFunction noise = new ValueNoise(GeneralNoise.LINEAR);
      double value = noise.get(seed, 0, x, 0);
      // ...
  }
  ```

- **Ignoring Seed Management:** Using a hardcoded or constant seed for all noise generators will produce repetitive and predictable results across the world. Proper procedural generation relies on cascading seeds and offsets to create varied and unique features.

## Data Pipeline
ValueNoise acts as a source node in a larger procedural generation data pipeline. It transforms discrete inputs into a continuous, pseudo-random signal.

> Flow:
> (Integer Seed, Integer Offset, Double Coordinates) -> **ValueNoise.get()** -> [Double Noise Value, range -1.0 to 1.0] -> Higher-Level Consumer (e.g., TerrainGenerator, BiomeSelector)

