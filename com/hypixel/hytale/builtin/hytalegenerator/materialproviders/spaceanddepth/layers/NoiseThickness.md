---
description: Architectural reference for NoiseThickness
---

# NoiseThickness

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.layers
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class NoiseThickness<V> extends SpaceAndDepthMaterialProvider.Layer<V> {
```

## Architecture & Concepts
The NoiseThickness class is a concrete implementation of the **Layer** strategy, designed to operate within the `SpaceAndDepthMaterialProvider` world generation system. Its primary function is to define the vertical thickness of a geological stratum at any given world coordinate by consulting a procedural noise function.

Architecturally, this class acts as a bridge between a generic `Density` function and the layer-based material placement logic. It effectively decouples the *shape* and *form* of a layer from its constituent *material*. By composing a `Density` object (for shape) with an optional `MaterialProvider` (for substance), world designers can create complex, organic-looking strata like undulating layers of stone, variable-thickness dirt, or patchy ore deposits.

This component is a fundamental building block for procedural terrain generation, enabling varied and non-uniform geological features without requiring pre-built models or manual placement.

### Lifecycle & Ownership
-   **Creation:** Instances are not created dynamically during gameplay. They are instantiated once during the world generator's configuration and loading phase, typically defined within a static data file (e.g., JSON) or assembled by a world generation preset builder.
-   **Scope:** The object's lifetime is bound to its parent `SpaceAndDepthMaterialProvider`. It persists in memory as part of the immutable world generation configuration for the duration of a server session or until a new world with a different configuration is loaded.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection when its parent `SpaceAndDepthMaterialProvider` is discarded.

## Internal State & Concurrency
-   **State:** The NoiseThickness class is **immutable**. Its two fields, `density` and `materialProvider`, are assigned only once via the constructor. It holds no mutable state and does not cache results.

-   **Thread Safety:** This class is **conditionally thread-safe**. Its own implementation is stateless and safe for concurrent access. However, its overall thread safety is entirely dependent on the injected `Density` and `MaterialProvider` dependencies.
    -   **WARNING:** The injected `Density` object **must** be thread-safe and produce deterministic output for a given input. Stateful or non-thread-safe density functions will corrupt parallel chunk generation, leading to visual artifacts and race conditions. The `getThicknessAt` method mitigates some risk by creating a new `Density.Context` per call, preventing state leakage between threads at this level.

## API Surface
The public API is minimal, exposing only the contract required by the parent `SpaceAndDepthMaterialProvider`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getThicknessAt(...) | int | O(D) | Calculates the layer thickness by invoking the injected `Density` function. Complexity is dependent on the `Density` implementation. |
| getMaterialProvider() | MaterialProvider<V> | O(1) | Returns the provider for the material that should fill this layer. May return null if this layer only defines space. |

## Integration Patterns

### Standard Usage
NoiseThickness is not intended for direct invocation. It is configured and added as a layer to a `SpaceAndDepthMaterialProvider`, which then manages its lifecycle and invocation during world generation.

```java
// Simplified configuration example
// Assume 'stoneProvider' and 'dirtProvider' are MaterialProvider instances
// Assume 'surfaceNoise' and 'baseNoise' are Density instances

SpaceAndDepthMaterialProvider<BlockState> terrainProvider = new SpaceAndDepthMaterialProvider<>();

// Create a layer of dirt whose thickness is controlled by a noise function
NoiseThickness<BlockState> dirtLayer = new NoiseThickness<>(surfaceNoise, dirtProvider);

// Create a deeper layer of stone
NoiseThickness<BlockState> stoneLayer = new NoiseThickness<>(baseNoise, stoneProvider);

// The provider's configuration holds the layers
terrainProvider.addLayer(dirtLayer);
terrainProvider.addLayer(stoneLayer);

// The engine will later use 'terrainProvider' to generate chunks.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Never call `getThicknessAt` from game logic or other systems. This method requires a very specific set of contextual parameters that are only correctly supplied by the parent `SpaceAndDepthMaterialProvider` during its generation pass. Bypassing the parent will yield incorrect and meaningless results.
-   **Stateful Dependencies:** Do not inject a `Density` object that relies on mutable state across calls. This will break the determinism of world generation and cause severe issues when chunks are generated by multiple threads.

## Data Pipeline
The NoiseThickness class is a single processing step within the larger `SpaceAndDepthMaterialProvider` data flow. It is responsible for converting spatial context into a scalar thickness value.

> Flow:
> World Generator requests block at (X, Y, Z) -> `SpaceAndDepthMaterialProvider` iterates its configured layers -> **NoiseThickness.getThicknessAt** is called with world coordinates and depth context -> The `Density` dependency is processed -> A raw `double` is returned -> **NoiseThickness** casts it to an `int` thickness -> The parent `SpaceAndDepthMaterialProvider` uses this thickness to determine if the block at (X, Y, Z) falls within this layer.

