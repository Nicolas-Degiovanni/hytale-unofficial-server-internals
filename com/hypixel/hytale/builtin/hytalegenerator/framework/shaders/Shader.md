---
description: Architectural reference for Shader
---

# Shader

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.shaders
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface Shader<T> {
   T shade(T var1, long var2);

   T shade(T var1, long var2, long var4);

   T shade(T var1, long var2, long var4, long var6);
}
```

## Architecture & Concepts
The Shader interface defines a fundamental contract within Hytale's procedural world generation framework. It represents a stateless, deterministic transformation function. Contrary to its name, it is **not** a graphics or rendering shader. Instead, it is a conceptual "shader" that modifies or "colors" an input value based on a set of coordinates or seeds.

This interface is a core building block for creating complex, emergent behavior from simple, repeatable rules. Implementations of Shader are used throughout the world generator to produce noise, select biomes, determine block types, and modify terrain features.

The generic type parameter **T** makes the interface highly versatile, allowing it to operate on any data type required by the generator, such as:
-   An integer representing a block ID.
-   A floating-point value representing terrain height.
-   A custom Biome object.

The `long` parameters in the `shade` methods are almost universally used to pass world coordinates (X, Y, Z) and the world seed. This ensures that for any given location in a specific world, the output of a Shader is always the same, which is critical for deterministic and reproducible world generation.

## Lifecycle & Ownership
As an interface, Shader itself has no lifecycle. The lifecycle described here pertains to its concrete implementations.

-   **Creation:** Implementations are typically instantiated and configured once by their parent system, such as a BiomeProvider or TerrainGenerator, during the engine's world-loading bootstrap sequence. They are often configured via data-driven systems.
-   **Scope:** An instance of a Shader implementation persists for the lifetime of the world generator it belongs to. If stateless, a single instance may be shared globally as a flyweight.
-   **Destruction:** Instances are eligible for garbage collection when the world they are associated with is unloaded.

## Internal State & Concurrency
-   **State:** The Shader contract **mandates** that all implementations be stateless and produce pure-functional output. The result of a `shade` call must depend exclusively on its input arguments. Caching of results based on inputs is permissible but must be managed in a thread-safe manner.

-   **Thread Safety:** Implementations **must be thread-safe**. The world generator heavily parallelizes chunk creation across multiple worker threads. Any Shader implementation could be invoked concurrently from any thread. Adherence to the stateless contract naturally confers thread safety.

    **WARNING:** Introducing mutable state into a Shader implementation will lead to severe and difficult-to-diagnose concurrency bugs, such as chunk boundary artifacts, non-deterministic generation, and race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shade(T value, long p1) | T | O(k) | Transforms the input value using a single coordinate or seed. |
| shade(T value, long p1, long p2) | T | O(k) | Transforms the input value using two coordinates or seeds, typically X and Z. |
| shade(T value, long p1, long p2, long p3) | T | O(k) | Transforms the input value using three coordinates or seeds, typically X, Y, and Z. |

*Complexity is implementation-defined but treated as a constant-time operation by callers.*

## Integration Patterns

### Standard Usage
A Shader is never used directly. Instead, a system retrieves a concrete implementation and invokes its `shade` method, typically within a tight loop during chunk column generation.

```java
// A system responsible for biome placement retrieves a configured Shader
// The generic type indicates this shader outputs Biome identifiers.
Shader<BiomeID> biomeShader = worldGenContext.getBiomeShader();

// World coordinates for the block being processed
long worldX = 1024;
long worldZ = -512;

// The initial value can be a default or a value from a previous stage
BiomeID baseBiome = BiomeID.PLAINS;

// The shader transforms the base biome based on world position
BiomeID finalBiome = biomeShader.shade(baseBiome, worldX, worldZ);

chunk.setBiomeAt(x, z, finalBiome);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Do not create implementations that store state between calls. The `shade` method must not have side effects.
-   **Non-Deterministic Logic:** Do not use non-deterministic functions like `Math.random()` or `System.currentTimeMillis()` inside a `shade` implementation. This will break world reproducibility. All randomness must be derived from the input `long` parameters.
-   **Interface Casting:** Do not attempt to cast a Shader to a specific implementation to access its internal methods. This violates the contract and couples the generator to a specific noise algorithm.

## Data Pipeline
The Shader interface is a key processing step in the world generation data pipeline. It acts as a transformation node that consumes coordinates and produces a modified value.

> Flow:
> World Coordinates (X, Y, Z) + World Seed -> **Shader<T>** -> Transformed Value T -> Higher-Level System (e.g., BiomePlacer, BlockSetter) -> Final Chunk Data

