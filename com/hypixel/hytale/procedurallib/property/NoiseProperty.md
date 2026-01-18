---
description: Architectural reference for NoiseProperty
---

# NoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface NoiseProperty {
```

## Architecture & Concepts
The NoiseProperty interface defines a fundamental contract for retrieving noise values within Hytale's procedural generation framework. It serves as a critical abstraction layer, decoupling noise-consuming systems (e.g., terrain generators, biome placement logic, texture applicators) from the specific underlying noise generation algorithms (e.g., Perlin, Simplex, Worley).

This design follows the **Strategy Pattern**, allowing different noise generation strategies to be injected into procedural systems without altering the consumer's code. A consumer requests a noise value for a specific coordinate, and the concrete implementation of NoiseProperty is responsible for calculating and returning that value based on its internal algorithm and configuration (such as seed, frequency, and octaves).

This interface is central to creating deterministic, varied, and performant world generation.

### Lifecycle & Ownership
As an interface, NoiseProperty itself has no lifecycle. It is a stateless contract. The lifecycle and ownership concerns apply to the **concrete classes that implement this interface**.

- **Creation:** Instances of implementing classes are typically created and configured by a higher-level procedural context or a world generation manager during the initialization of a world generation pass. They are often part of a larger configuration object that defines the properties of a biome or terrain feature.
- **Scope:** The lifetime of a NoiseProperty implementation is tied to the scope of the generation task it serves. For world generation, an instance may live for the duration of a chunk generation process or be cached for the entire world session if its parameters are constant.
- **Destruction:** Implementations are subject to standard Java garbage collection. They are typically released once the procedural generation context or the world they belong to is unloaded.

## Internal State & Concurrency
The interface itself is stateless. However, any class implementing NoiseProperty is almost certain to be stateful.

- **State:** Concrete implementations will hold internal state, including the world seed, noise frequency, amplitude, number of octaves, lacunarity, and persistence. This state is typically configured at creation and is considered immutable for the duration of a single, deterministic generation task.
- **Thread Safety:** **This interface provides no thread-safety guarantees.** The responsibility for ensuring thread-safe access lies entirely with the implementing class. In a multi-threaded world generation environment, it is critical that implementations are either immutable or employ appropriate synchronization mechanisms if their internal state can be modified after construction. Most performant implementations are expected to be "effectively immutable" post-initialization, allowing safe concurrent read-only calls from multiple generator threads.

## API Surface
The public contract is minimal, focused exclusively on retrieving 2D or 3D noise values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int seed, double x, double y) | double | O(N) | Retrieves a 2D noise value for the given coordinates. The first parameter is a seed or seed-modifier. Complexity is O(N) where N is the number of octaves in the concrete implementation. |
| get(int seed, double x, double y, double z) | double | O(N) | Retrieves a 3D noise value for the given coordinates. The first parameter is a seed or seed-modifier. Complexity is O(N) where N is the number of octaves in the concrete implementation. |

**WARNING:** The output range of the get methods (e.g., [-1, 1] vs [0, 1]) is not defined by this contract. Consumers must not assume a specific range and should normalize the output if required.

## Integration Patterns

### Standard Usage
A system, such as a TerrainGenerator, receives a concrete implementation of NoiseProperty from a configuration or service registry. It then invokes the get method within its generation loop to determine terrain characteristics at specific world coordinates.

```java
// A terrain generator receives a pre-configured NoiseProperty
// for calculating elevation.
public class TerrainGenerator {
    private final NoiseProperty elevationNoise;

    public TerrainGenerator(NoiseProperty elevationNoise) {
        this.elevationNoise = elevationNoise;
    }

    public BlockType getBlockAt(int worldX, int worldY, int worldZ, int worldSeed) {
        // Use the interface to get a raw noise value
        double rawElevation = elevationNoise.get(worldSeed, worldX, worldZ);

        // Business logic uses the noise value
        if (worldY < (64 + rawElevation * 32)) {
            return BlockType.STONE;
        }
        return BlockType.AIR;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Assuming Implementation:** Do not cast a NoiseProperty to a specific concrete type (e.g., PerlinNoiseProperty) to access implementation-specific methods. This violates the abstraction and makes the system brittle.
- **Ignoring the Seed:** Failing to pass a consistent and correct seed to the get method will result in non-deterministic and unpredictable procedural generation.
- **Assuming Output Range:** Do not write logic that assumes the noise value will be within a specific range like [-1, 1]. This is not guaranteed by the contract and will lead to bugs when different noise implementations are used.

## Data Pipeline
The NoiseProperty interface acts as a source node in the procedural generation data pipeline. It transforms a coordinate and a seed into a scalar value that drives subsequent logic.

> Flow:
> World Generation Parameters (Seed, Frequency) -> **Concrete NoiseProperty Implementation** -> Raw Noise Value (double) -> Generator Logic (e.g., Terrain Height, Biome Blend Factor) -> Final World Data (Blocks, Entities)

