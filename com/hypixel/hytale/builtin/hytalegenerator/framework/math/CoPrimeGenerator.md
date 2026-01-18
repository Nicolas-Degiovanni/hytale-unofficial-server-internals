---
description: Architectural reference for CoPrimeGenerator
---

# CoPrimeGenerator

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Utility

## Definition
```java
// Signature
public class CoPrimeGenerator {
```

## Architecture & Concepts
The CoPrimeGenerator is a stateless, deterministic mathematical utility designed to produce arrays of large, relatively prime numbers. Its primary role within the engine is to supply foundational values for procedural content generation, particularly for multi-dimensional noise functions like Perlin or Simplex noise.

In procedural generation, using offsets that share common factors can lead to undesirable visual artifacts, such as grid-like patterns, aliasing, or obvious repetition. By generating a set of numbers that are products of distinct prime sets, this class provides the world generation system with robust coordinate offsets. These offsets ensure that when sampling noise across different dimensions (e.g., X, Y, Z), the sampling points are distributed in a way that avoids harmonic resonance and produces more natural, non-repeating patterns.

This class is a low-level, foundational component. It does not orchestrate generation but provides the mathematical primitives required by higher-level systems like the BiomeGenerator or TerrainVoxelizer.

### Lifecycle & Ownership
- **Creation:** The CoPrimeGenerator is a static utility class and is never instantiated. Its methods are accessed statically.
- **Scope:** Its scope is application-wide, available for use as soon as its class is loaded by the JVM.
- **Destruction:** Not applicable. The class is unloaded by the JVM during application shutdown.

## Internal State & Concurrency
- **State:** The CoPrimeGenerator is stateless. It maintains no internal fields or cached data between calls. Each invocation of its methods operates exclusively on the arguments provided.
- **Thread Safety:** This class is inherently thread-safe. All methods are static and do not modify any shared state. The primary method, generateCoPrimes, creates a local Random instance per call, ensuring that concurrent invocations with the same seed will produce identical results without interference. It is safe to call from any thread, including parallel world generation worker threads.

## API Surface
The public API consists of a primary generator function and its mathematical helpers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateCoPrimes(seed, bucketSize, buckets, floor) | long[] | High | Generates a deterministic array of large, relatively prime numbers. Computationally expensive and should not be called in performance-critical loops. |
| fillWithPrimes(int[] bucket) | void | High | Populates a given integer array with sequential prime numbers, starting from 2. Exposed for testing and specialized use cases. |
| isPrime(int number) | boolean | O(n) | Performs a naive primality test on the given integer. Its public visibility is for utility, but it is not optimized for large numbers. |

## Integration Patterns

### Standard Usage
The CoPrimeGenerator should be invoked once during the initialization phase of a world or dimension. The resulting array of offsets should be cached by the calling system and reused for all subsequent noise calculations within that world.

```java
// Invoked by a world generator to create distinct offsets for noise layers.
long worldSeed = 123456789L;
int dimensions = 3; // For X, Y, and Z coordinate noise
long minimumOffset = 100000L;

// Generate the offsets once and store them.
long[] noiseOffsets = CoPrimeGenerator.generateCoPrimes(worldSeed, 5, dimensions, minimumOffset);

// Later, in the terrain generation loop, use the cached offsets.
// float height = Noise.get(x * frequency + noiseOffsets[0], y * frequency + noiseOffsets[1]);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CoPrimeGenerator()`. This is a utility class with only static methods.
- **Frequent Invocation:** Calling generateCoPrimes repeatedly, especially inside a terrain generation loop, is a critical performance anti-pattern. The cost of prime number generation is extremely high. Generate once, cache forever.
- **Non-Deterministic Seeding:** Using a volatile seed like System.currentTimeMillis() for the `seed` parameter will produce a different world every time. For reproducible worlds, a fixed seed must be passed through the entire generation pipeline.

## Data Pipeline
The CoPrimeGenerator acts as a pure function that transforms a seed into a set of numerical constants for use by downstream systems.

> Flow:
> World Generation Seed -> **CoPrimeGenerator**.generateCoPrimes -> Array of Large Offsets -> Noise Function Seeding -> Voxel Terrain Generation

