---
description: Architectural reference for Normalizer
---

# Normalizer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Utility

## Definition
```java
// Signature
public class Normalizer {
```

## Architecture & Concepts
The Normalizer is a stateless, pure-functional mathematics utility class. It serves a foundational role within the procedural world generation pipeline. Its primary responsibility is to remap numerical values from an arbitrary input range to a standardized output range, most commonly 0.0 to 1.0.

This process is critical for consuming raw output from noise functions (e.g., Perlin, Simplex), which typically produce values in an unbounded or [-1.0, 1.0] range. By transforming these values into a predictable, normalized space, subsequent generation stages—such as biome selection, terrain height modulation, and material distribution—can operate on a consistent and well-understood data set.

As a utility, it has no dependencies and produces no side effects, making it a highly reliable and reusable component throughout the engine's mathematical frameworks.

### Lifecycle & Ownership
- **Creation:** The Normalizer class is never instantiated. It consists exclusively of static methods and is loaded by the JVM ClassLoader upon its first reference. A private constructor should be assumed to prevent instantiation.
- **Scope:** The class and its static methods are available for the entire application lifetime once loaded.
- **Destruction:** The class is unloaded from memory when the JVM shuts down. There is no instance-level state to manage or clean up.

## Internal State & Concurrency
- **State:** The Normalizer is completely stateless. It maintains no internal fields, caches, or configuration. Each method call's output is determined solely by its input arguments.
- **Thread Safety:** This class is inherently thread-safe. Its stateless and pure-functional nature guarantees that its methods can be invoked concurrently from multiple threads without any risk of race conditions or data corruption. No external synchronization is required.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| normalizeNoise(double input) | double | O(1) | Remaps a value from the standard noise range of [-1.0, 1.0] to the normalized range of [0.0, 1.0]. |
| normalize(fromMin, fromMax, toMin, toMax, input) | double | O(1) | Remaps a value from a specified source range to a target range. Throws IllegalArgumentException if any min value is greater than its corresponding max. |

## Integration Patterns

### Standard Usage
The Normalizer is almost always used immediately after sampling a noise function to prepare the value for other systems.

```java
// Example within a terrain generation module
NoiseGenerator noise = worldContext.getNoiseGenerator("terrain_elevation");
double rawValue = noise.get(x, z); // Produces a value, e.g., -0.78

// Normalize the raw value to a 0-1 range for heightmap application
double normalizedHeight = Normalizer.normalizeNoise(rawValue);

terrainMap.setHeight(x, z, normalizedHeight);
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** Do not attempt to create an instance of this class. It is a static utility. `new Normalizer()` is invalid and will fail if the class has a private constructor as is best practice.
- **Invalid Range Arguments:** Providing a `fromMin` greater than `fromMax` (or `toMin` > `toMax`) will result in a runtime crash via an IllegalArgumentException. All range inputs must be validated by the caller if they are not compile-time constants.

## Data Pipeline
The Normalizer acts as a simple, stateless transformation step in a larger data processing pipeline, typically for procedural generation.

> Flow:
> Noise Function -> Raw Value (e.g., -1.0 to 1.0) -> **Normalizer.normalizeNoise()** -> Normalized Value (0.0 to 1.0) -> Biome/Height/Material Logic

