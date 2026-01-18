---
description: Architectural reference for the NoiseFunction interface, the core abstraction for procedural generation.
---

# NoiseFunction

**Package:** com.hypixel.hytale.procedurallib
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface NoiseFunction extends NoiseFunction2d, NoiseFunction3d {
```

## Architecture & Concepts
The NoiseFunction interface is the fundamental contract for all procedural noise algorithms within Hytale's world generation engine. It serves as a unified abstraction for sampling a deterministic, pseudo-random value at any given point in 2D or 3D space.

By extending both NoiseFunction2d and NoiseFunction3d, this interface ensures that any noise implementation can be used interchangeably by higher-level systems, regardless of whether the context is 2D (e.g., biome mapping) or 3D (e.g., cave generation, density fields). This polymorphic design is critical for creating a modular and composable world generation pipeline, where different noise algorithms (Perlin, Simplex, Worley, etc.) can be swapped, chained, or modified without altering the systems that consume them.

It sits at the lowest level of the procedural generation stack, providing the raw data that is later interpreted by systems like TerrainShapers, BiomeSelectors, and OreVeinGenerators.

## Lifecycle & Ownership
As an interface, NoiseFunction itself does not have a lifecycle. However, its concrete implementations follow a strict and predictable pattern.

- **Creation:** Implementations are almost exclusively instantiated and configured during the world generation bootstrap phase. They are typically defined in worldgen asset files (e.g., JSON or HOCON) and loaded by a central ProceduralManager or a world-specific context. Direct instantiation in game logic is heavily discouraged.
- **Scope:** A configured NoiseFunction instance is long-lived. It persists for the entire duration of a world generation session for a given dimension or context. They are designed to be stateless or have immutable configuration (seed, frequency, octaves) to ensure deterministic and repeatable results.
- **Destruction:** Instances are garbage collected when the world generation context is destroyed, for example, when a server shuts down or a different world is loaded. There is no manual destruction or cleanup required.

## Internal State & Concurrency
The contract of NoiseFunction implicitly demands specific behaviors regarding state and threading.

- **State:** Implementations are expected to be effectively immutable after their initial configuration. All parameters required to calculate a noise value (seed, frequency, permutation tables) should be finalized upon creation. Caching of intermediate values, such as permutation tables, is common for performance but must not affect the deterministic nature of the output.
- **Thread Safety:** **CRITICAL:** All implementations of NoiseFunction **must be** thread-safe. World generation is a massively parallel process, with multiple chunks being generated simultaneously across many worker threads. A single NoiseFunction instance will be called concurrently from these threads. Implementations must not contain mutable state that is modified during a `get` call. All internal data structures must be read-only after initialization.

## API Surface
The API is inherited from its parent interfaces and represents the core sampling contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(double x, double y) | double | O(1) | Samples the 2D noise field at the given coordinates. Returns a value typically in the range [-1.0, 1.0]. |
| get(double x, double y, double z) | double | O(1) | Samples the 3D noise field at the given coordinates. Returns a value typically in the range [-1.0, 1.0]. |

## Integration Patterns

### Standard Usage
A system, such as a terrain generator, retrieves a pre-configured noise function from a registry or context and uses it to determine terrain density at a specific world coordinate.

```java
// Retrieve a pre-configured noise function for continentalness
NoiseFunction continentalnessNoise = worldgenContext.getNoise("continentalness");

// Sample the noise at a specific world position
double rawValue = continentalnessNoise.get(worldX, worldZ);

// Use the value to influence biome or terrain height
if (rawValue > 0.5) {
    // Generate mountain terrain
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create a new noise function instance inside a tight loop (e.g., per-block or per-chunk). This is extremely inefficient and defeats the purpose of a configurable, reusable system. Always retrieve shared instances from a central context.
- **Implementation Casting:** Avoid casting a NoiseFunction to a specific implementation (e.g., PerlinNoise). The power of the system lies in its abstraction. Rely only on the methods defined by the interface contract.
- **Ignoring Determinism:** Do not implement a NoiseFunction that relies on non-deterministic sources like `Math.random()` or the system clock. The entire world generation system depends on the output being identical for the same seed and input coordinates.

## Data Pipeline
NoiseFunction is the starting point for transforming spatial coordinates into meaningful procedural data.

> Flow:
> World Coordinates (x, y, z) -> **NoiseFunction** -> Raw Noise Value [-1, 1] -> Modifier Chain (e.g., scale, bias) -> Domain Warping -> Final Value -> Consumed by BiomeSelector, TerrainGenerator, etc.

