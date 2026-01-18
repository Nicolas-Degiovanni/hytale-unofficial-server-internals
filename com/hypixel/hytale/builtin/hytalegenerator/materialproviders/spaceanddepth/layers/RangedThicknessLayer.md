---
description: Architectural reference for RangedThicknessLayer
---

# RangedThicknessLayer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.layers
**Type:** Transient Component

## Definition
```java
// Signature
public class RangedThicknessLayer<V> extends SpaceAndDepthMaterialProvider.Layer<V> {
```

## Architecture & Concepts
The RangedThicknessLayer is a concrete implementation of the Layer strategy, designed for use within the SpaceAndDepthMaterialProvider system. Its fundamental purpose is to define a horizontal stratum of material whose thickness varies randomly but deterministically across the world.

This component is a foundational element in procedural world generation, enabling the creation of natural-looking geological layers like topsoil, clay, or stone that are not perfectly uniform. The system's determinism is critical: by deriving a random seed from world coordinates (x, z), it guarantees that any given vertical column will have the same layer thickness every time the world is generated with the same master seed.

Architecturally, it decouples the logic for *thickness calculation* from the logic of *material type*. It calculates a thickness value, and the parent SpaceAndDepthMaterialProvider uses this information, along with the associated MaterialProvider, to place blocks during chunk generation.

### Lifecycle & Ownership
- **Creation:** Instantiated programmatically during the configuration phase of a world generator. It is typically constructed as part of a biome or zone definition, where a list of layers is assembled to define the vertical material profile of the terrain.
- **Scope:** The lifetime of a RangedThicknessLayer instance is bound to its parent SpaceAndDepthMaterialProvider. It is a configuration object that holds no state beyond its initial setup.
- **Destruction:** The object is self-contained and requires no explicit cleanup. It becomes eligible for garbage collection as soon as its parent provider is no longer in scope.

## Internal State & Concurrency
- **State:** This class is **effectively immutable**. All configuration fields (min, max, seedGenerator, materialProvider) are set in the constructor and never modified. The class does not cache results or maintain any state related to previous invocations.

- **Thread Safety:** RangedThicknessLayer is **fully thread-safe** and designed for high-concurrency environments, such as a parallelized world generation engine. The primary computation method, getThicknessAt, is a pure function of its inputs and the immutable internal state. It instantiates a new FastRandom object for each call, preventing any potential for shared state conflicts between threads generating adjacent world chunks.

## API Surface
The public API is minimal, focusing exclusively on fulfilling its contract as a Layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getThicknessAt(...) | int | O(1) | Calculates the deterministic, pseudo-random thickness for a given world column (x, z). The result is guaranteed to be between the min and max values set at construction. |
| getMaterialProvider() | MaterialProvider | O(1) | Returns the material provider associated with this layer, which defines what blocks the layer is made of. May return null. |

## Integration Patterns

### Standard Usage
A RangedThicknessLayer is not used in isolation. It is constructed and passed into a system that composes multiple layers, such as the SpaceAndDepthMaterialProvider.

```java
// Example: Defining a 2-to-5 block thick layer of dirt
// A SeedBox is retrieved from the broader world generation context.
SeedBox worldSeedBox = generatorContext.getSeedBox();

// The material for the layer is defined separately.
MaterialProvider<Block> dirtProvider = new SingleMaterialProvider<>(DIRT_BLOCK);

// The layer is instantiated with its configuration.
RangedThicknessLayer<Block> dirtLayer = new RangedThicknessLayer<>(
    2,                          // min thickness
    5,                          // max thickness
    worldSeedBox,               // for deterministic randomness
    dirtProvider                // the material to use
);

// The configured layer is then added to the primary provider.
spaceAndDepthProvider.addLayer(dirtLayer);
```

### Anti-Patterns (Do NOT do this)
- **Stateful SeedBox:** Do not provide a SeedBox that returns a different seed on subsequent calls within the same generation process. The SeedGenerator relies on a stable base seed to produce deterministic results.
- **Invalid Range:** Constructing a layer with a minimum thickness greater than the maximum thickness will throw an IllegalArgumentException. This is a configuration error and must be avoided.

## Data Pipeline
This component acts as a computational function within the larger world generation pipeline. It does not transform data streams but rather answers a specific question: "How thick should this layer be at this location?"

> Flow:
> World Generator (requests block at x,y,z) -> SpaceAndDepthMaterialProvider (evaluates layers) -> **RangedThicknessLayer.getThicknessAt(x,z)** -> Returns Thickness (int) -> Parent Provider places blocks using the associated MaterialProvider

