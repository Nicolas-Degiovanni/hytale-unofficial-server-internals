---
description: Architectural reference for SimplexNoiseField
---

# SimplexNoiseField

**Package:** com.hypixel.hytale.builtin.hytalegenerator.fields.noise
**Type:** Transient

## Definition
```java
// Signature
public class SimplexNoiseField extends NoiseField {
```

## Architecture & Concepts
The SimplexNoiseField is a foundational component within the procedural world generation engine. It provides a concrete implementation of the NoiseField interface, using a multi-octave Simplex noise algorithm to produce natural-looking, pseudo-random patterns.

Architecturally, this class serves as a deterministic function that maps a set of input coordinates (1D, 2D, 3D, or 4D) to a normalized floating-point value. Its primary purpose is to generate coherent noise, which is essential for creating continuous and organic features like terrain elevation, biome distribution, temperature maps, and cave systems.

By layering multiple noise functions (octaves) at different frequencies and amplitudes, it produces fractal patterns with varying levels of detail. This layering is key to its utility; low-frequency octaves define the large-scale shapes (like continents or mountain ranges), while high-frequency octaves add fine-grained detail (like hills and surface roughness). The entire calculation is seeded, ensuring that for a given world seed, the generated world is identical every time.

### Lifecycle & Ownership
- **Creation:** Instances are created on-demand by higher-level world generation systems. The preferred and safest method of instantiation is via the provided static Builder. Direct constructor invocation is possible but discouraged.
- **Scope:** The lifetime of a SimplexNoiseField object is typically short and bound to a specific generation task. For example, a generator might create an instance to produce a heightmap for a single world zone, after which the instance is no longer referenced and becomes eligible for garbage collection. It is not a long-lived, session-wide service.
- **Destruction:** Managed entirely by the Java garbage collector. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
- **State:** The internal state of a SimplexNoiseField is **immutable** after construction. All configuration parameters, including the seed, octave count, and pre-calculated offset and amplitude arrays, are initialized once in the constructor and are never modified thereafter. This design guarantees deterministic and repeatable output.

- **Thread Safety:** This class is **inherently thread-safe**. Due to its immutable state, a single SimplexNoiseField instance can be safely shared and accessed by multiple worker threads simultaneously without the need for external locking or synchronization. This is a critical feature for enabling high-performance, parallelized world generation.

## API Surface
The public contract is focused on configuration via its builder and the core value retrieval methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| builder() | static Builder | O(1) | Returns a new builder instance for fluent configuration. |
| valueAt(double x, ...) | double | O(N) | Calculates the normalized noise value at the given coordinates. N is the number of octaves. Overloaded for 1D, 2D, 3D, and 4D. |
| getSeed() | long | O(1) | Returns the seed used to initialize this noise field. |

## Integration Patterns

### Standard Usage
The intended use is through the fluent Builder pattern. This allows for clear, readable, and safe configuration of the noise parameters before the object is constructed.

```java
// How a developer should normally use this
SimplexNoiseField terrainHeightNoise = SimplexNoiseField.builder()
    .withSeed(worldSeed)
    .withNumberOfOctaves(8)
    .withFrequencyMultiplier(2.0)
    .withAmplitudeMultiplier(0.5)
    .withScale(512.0)
    .build();

double height = terrainHeightNoise.valueAt(chunkX, chunkZ);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid using `new SimplexNoiseField(...)`. The constructor has a complex signature and provides no scaling configuration, which must be set on the parent class. The Builder handles the complete lifecycle correctly and is less error-prone.
- **State Modification:** Do not attempt to modify the state of a SimplexNoiseField instance after creation using reflection or other means. Its immutability is essential for determinism and thread safety. For different noise, create a new instance.
- **Ignoring Normalization:** The output of valueAt is a normalized value, typically between -1.0 and 1.0. Do not assume it falls within another range, such as 0.0 to 256.0. Always scale and offset the result explicitly to fit the target application (e.g., block height).

## Data Pipeline
SimplexNoiseField acts as a procedural data source within the world generation pipeline. It transforms a set of configuration parameters and spatial coordinates into a coherent noise value.

> Flow:
> World Generation Configuration -> **SimplexNoiseField.Builder** -> **SimplexNoiseField** -> valueAt(x, y, z) -> [Normalized Value] -> Biome/Terrain Logic -> Voxel Data

