---
description: Architectural reference for SeedGenerator
---

# SeedGenerator

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Transient

## Definition
```java
// Signature
public class SeedGenerator {
```

## Architecture & Concepts
The SeedGenerator is a foundational component within the procedural world generation framework. Its primary function is to transform a set of spatial coordinates into a single, deterministic, and uniformly distributed seed value. This allows any subsequent pseudo-random operation, such as noise generation or feature placement, to be perfectly reproducible for a given world seed and location.

Architecturally, this class acts as a high-performance spatial hashing function. It is designed to produce a high degree of output change for small input changes (an avalanche effect), which is critical for preventing visually obvious patterns or grid-like artifacts in the generated world.

The core of its algorithm relies on a set of pre-calculated co-prime numbers derived from an initial master seed. Each coordinate axis (x, y, z, etc.) is multiplied by a unique co-prime number. The sum of these products, taken modulo another co-prime, produces the final seed. This mathematical approach ensures a robust distribution of output seeds across the input coordinate space. The responsibility for generating these co-primes is delegated to the CoPrimeGenerator utility, ensuring a clean separation of concerns.

## Lifecycle & Ownership
- **Creation:** An instance of SeedGenerator is created by a higher-level system responsible for a specific procedural generation task, such as a BiomeGenerator or FeaturePlacer. It is initialized with a master seed, which is typically the world seed or a derivative of it.

- **Scope:** The lifetime of a SeedGenerator is scoped to the generation task it serves. It is not a global singleton. A world generation process may create many short-lived SeedGenerator instances, each configured with a slightly different initial seed to represent a different "random stream" for a specific feature (e.g., one for trees, one for ores).

- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and is eligible for collection as soon as the owning generation process completes and releases its reference.

## Internal State & Concurrency
- **State:** The internal state consists of a `long` array of co-prime numbers. This state is **immutable**. It is populated once during construction and is never modified thereafter. The `final` keyword on the field enforces this immutability at the instance level.

- **Thread Safety:** This class is **unconditionally thread-safe**. Due to its immutable internal state, all public methods are pure functions whose output depends solely on their input arguments. A single SeedGenerator instance can be safely shared and accessed by multiple worker threads in a parallelized world generation engine without any need for external locking or synchronization. This is a critical design feature for performance.

## API Surface
The public API consists of a family of overloaded `seedAt` methods for converting coordinates into a seed.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| seedAt(long... coords) | long | O(1) | Computes a deterministic seed from 2 to 6 integer coordinates. The number of arguments determines the dimensionality of the input space. |
| seedAt(double... coords, double resolution) | long | O(1) | Computes a seed from 2 to 6 floating-point coordinates. The coordinates are first discretized by multiplying by the resolution parameter. |

## Integration Patterns

### Standard Usage
A SeedGenerator should be instantiated once per major generation task. Its purpose is to provide seeds for initializing more complex random number generators or noise functions at specific locations.

```java
// In a world generation orchestrator, create a generator for a specific feature.
// Using a derivative of the master seed ensures this feature's randomness
// is independent from other features.
long masterWorldSeed = world.getSeed();
SeedGenerator treeSeedGenerator = new SeedGenerator(masterWorldSeed + 100);

// Inside a loop generating a chunk at chunkPos
for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        long worldX = chunkPos.x * 16 + x;
        long worldZ = chunkPos.z * 16 + z;

        // Get a unique, deterministic seed for this specific column
        long columnSeed = treeSeedGenerator.seedAt(worldX, worldZ);

        // Use the seed to drive further logic, like initializing a new Random instance
        Random columnRandom = new Random(columnSeed);
        if (columnRandom.nextFloat() < 0.05) {
            // placeTreeAt(worldX, y, worldZ);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instantiation in a Loop:** Never create a new SeedGenerator inside a tight loop (e.g., per-block). The constructor performs a non-trivial co-prime calculation. This is highly inefficient and defeats the purpose of the class.

    ```java
    // BAD: Excessive object creation and recalculation
    for (int x = 0; x < 100; x++) {
        // This recalculates co-primes on every iteration!
        SeedGenerator badGenerator = new SeedGenerator(worldSeed);
        long seed = badGenerator.seedAt(x, 0);
    }
    ```

- **Using a Single Global Instance:** While thread-safe, using one single SeedGenerator for all unrelated features (e.g., caves, trees, ores) can create subtle and undesirable correlations between them. The standard pattern is to create separate instances with different initial seeds to establish independent random streams for each feature.

## Data Pipeline
The flow of data for creating and using a SeedGenerator is linear and deterministic.

> Flow:
> Master World Seed -> **CoPrimeGenerator** -> Co-prime Array -> **SeedGenerator (Constructor)**
>
> Spatial Coordinates (x, y, z) -> **SeedGenerator.seedAt()** -> Deterministic Local Seed -> Random Number Generator or Noise Function

