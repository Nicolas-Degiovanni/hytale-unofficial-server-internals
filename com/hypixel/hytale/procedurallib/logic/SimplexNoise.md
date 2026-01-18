---
description: Architectural reference for SimplexNoise
---

# SimplexNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Singleton / Utility

## Definition
```java
// Signature
public class SimplexNoise implements NoiseFunction {
```

## Architecture & Concepts
The SimplexNoise class is a concrete, high-performance implementation of the NoiseFunction interface. It serves as a foundational component within the procedural generation library, responsible for creating 2D and 3D gradient noise, commonly known as Simplex noise.

Architecturally, this class is designed as a pure, stateless computational utility. Its purpose is to transform a set of input coordinates and a seed into a deterministic, pseudo-random floating-point value. This output is continuous and repeatable, making it ideal for generating natural-looking textures, terrain features, and other procedural content.

It operates by dividing the input space into a grid of simplices (triangles in 2D, tetrahedrons in 3D) and interpolating values based on gradient vectors at the simplex vertices. This approach is computationally more efficient than classic Perlin noise, particularly in higher dimensions, and avoids most of its directional artifacts. The core mathematical constants and algorithmic steps are self-contained, but it delegates low-level hashing and gradient selection to the GeneralNoise utility class.

## Lifecycle & Ownership
- **Creation:** The SimplexNoise instance is created statically upon class loading by the Java Virtual Machine. The public static final field INSTANCE holds the single, globally accessible object. The constructor is private, strictly enforcing the Singleton pattern.
- **Scope:** Application-global. The INSTANCE field persists for the entire lifetime of the application.
- **Destruction:** The object is eligible for garbage collection only when the application's class loader is unloaded, typically during JVM shutdown.

## Internal State & Concurrency
- **State:** This class is entirely stateless and immutable. It contains no instance-level fields. All data required for computation is provided via method arguments. The only fields present are static final constants used for the noise algorithm, which are inherently immutable.
- **Thread Safety:** Inherently and unconditionally thread-safe. As a stateless object, the public INSTANCE can be safely accessed and its methods invoked by any number of concurrent threads without external synchronization or risk of race conditions. This is a critical design feature for parallelized world generation tasks.

## API Surface
The public contract is defined by the NoiseFunction interface, providing methods for 2D and 3D noise generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, offsetSeed, x, y) | double | O(1) | Calculates the 2D Simplex noise value for the given coordinates. |
| get(seed, offsetSeed, x, y, z) | double | O(1) | Calculates the 3D Simplex noise value for the given coordinates. |

## Integration Patterns

### Standard Usage
The class is intended to be used directly via its static INSTANCE field. It should be treated as a shared, global utility for any system requiring deterministic noise.

```java
// Standard access pattern for generating 2D noise
NoiseFunction noiseGenerator = SimplexNoise.INSTANCE;
double terrainHeight = noiseGenerator.get(worldSeed, 0, blockX, blockZ);

// Standard access pattern for generating 3D noise for caves
double caveValue = noiseGenerator.get(worldSeed, 1, blockX, blockY, blockZ);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Attempting to create a new instance using `new SimplexNoise()` will fail at compile time due to the private constructor. This design is intentional.
- **Reflection Abuse:** Bypassing the private constructor via reflection is a severe violation of the class's design contract. It offers no benefits and undermines the performance and memory guarantees of the singleton pattern.
- **Stateful Wrapping:** Do not wrap the SimplexNoise INSTANCE in another class that holds state related to the noise generation (e.g., storing the seed as a field). The noise function itself is pure; state should be managed by the calling system, such as a WorldGenerator.

## Data Pipeline
SimplexNoise acts as a data source or a function within a larger procedural generation pipeline. It does not process a stream of data itself but is invoked to generate a value at a specific point.

> Flow:
> Procedural Generator -> **SimplexNoise.INSTANCE.get(seed, coords)** -> Raw Noise Value [-1, 1] -> Post-Processing (Scaling, Clamping) -> Final Terrain/Biome Data

