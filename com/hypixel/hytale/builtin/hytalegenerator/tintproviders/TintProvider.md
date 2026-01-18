---
description: Architectural reference for TintProvider
---

# TintProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.tintproviders
**Type:** Base Class / Strategy

## Definition
```java
// Signature
public abstract class TintProvider {
```

## Architecture & Concepts
The TintProvider is an abstract base class that defines a strategy for calculating a color tint for a specific location within the world. It serves as a fundamental component in the procedural world generation pipeline, allowing for dynamic and data-driven environmental coloring, most notably for biomes.

This class embodies the Strategy design pattern. Concrete implementations, such as a `SwampTintProvider` or a `MountainTintProvider`, encapsulate specific coloring algorithms. The world generator selects and invokes the appropriate provider based on biome data or other contextual world-generation rules for a given block coordinate.

The inclusion of a `WorkerIndexer.Id` within its `Context` object indicates that this system is designed to operate within a highly concurrent, multi-threaded world generation environment. Each worker thread processes different parts of the world, and the TintProvider must be able to supply a deterministic color value based solely on the provided context.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of TintProvider are typically instantiated and configured during the loading of world generation assets, such as biome definition files. A factory or registry system is responsible for creating and managing these provider instances. The static factory method `noTintProvider` creates a default, constant-value provider.
- **Scope:** The lifespan of a TintProvider instance is generally tied to the server session. They are loaded once and held in memory to be queried repeatedly by world generation workers. The nested `Context` and `Result` objects, however, are transient and have a very short lifecycle, existing only for the duration of a single `getValue` call.
- **Destruction:** Provider instances are garbage collected when the server shuts down or when world generation assets are reloaded.

## Internal State & Concurrency
- **State:** The abstract TintProvider class is stateless. Concrete implementations should be designed to be immutable or effectively immutable. Any internal state, such as pre-calculated noise maps or color palettes, should be read-only after initialization.
- **Thread Safety:** This class is designed for a multi-threaded environment. The public contract, specifically the `getValue` method, **must be thread-safe**. Implementations must not rely on mutable shared state that could be accessed by multiple worker threads simultaneously without proper synchronization. The ideal implementation is a pure function of its input `Context`.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue(Context) | Result | Varies | Abstract method. Calculates the tint for a given world position and worker context. |
| noTintProvider() | TintProvider | O(1) | Static factory method that returns a default provider which always supplies a standard, non-tinted value. |

## Integration Patterns

### Standard Usage
A developer implementing custom world generation features will create a concrete subclass of TintProvider. This class will be registered with a higher-level system, like a BiomeManager, which will invoke it during chunk generation.

```java
// Example of a concrete implementation
public class ForestTintProvider extends TintProvider {
    // Assume noise is initialized elsewhere and is thread-safe
    private final FastNoiseLite noise;

    public ForestTintProvider(long seed) {
        this.noise = new FastNoiseLite(seed);
        this.noise.SetNoiseType(FastNoiseLite.NoiseType.Perlin);
    }

    @Override
    public TintProvider.Result getValue(@Nonnull TintProvider.Context context) {
        float noiseValue = noise.GetNoise(context.position.x, context.position.z);
        int greenComponent = 90 + (int)(noiseValue * 30);
        int color = Color.rgb(60, greenComponent, 30);
        return new TintProvider.Result(color);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Mutable State:** Do not add mutable fields to a TintProvider subclass that are modified within the `getValue` method. This will create severe race conditions when used by multiple world generation threads.
- **Blocking Operations:** The `getValue` method is on a hot path during world generation. Avoid any file I/O, network requests, or other long-running blocking operations within this method. All necessary data should be loaded and cached at initialization.

## Data Pipeline
The TintProvider acts as a functional step in the block-coloring stage of the world generation pipeline.

> Flow:
> World Generator requests chunk -> Generator iterates block positions -> Biome lookup for position -> **TintProvider.getValue(position)** -> Resulting tint integer -> Applied to block mesh data -> Sent to client for rendering

