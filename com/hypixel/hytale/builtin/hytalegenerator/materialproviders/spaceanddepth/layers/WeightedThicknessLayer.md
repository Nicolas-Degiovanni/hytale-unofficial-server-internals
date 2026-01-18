---
description: Architectural reference for WeightedThicknessLayer
---

# WeightedThicknessLayer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.layers
**Type:** Transient

## Definition
```java
// Signature
public class WeightedThicknessLayer<V> extends SpaceAndDepthMaterialProvider.Layer<V> {
```

## Architecture & Concepts
The WeightedThicknessLayer is a specialized implementation of a world generation Layer used within the SpaceAndDepthMaterialProvider framework. Its primary function is to determine the thickness of a procedural layer at a given world column using a probabilistic, weighted model.

Unlike layers that use noise functions like Perlin or Simplex to create smooth, continuous variations in thickness, this class selects a thickness value from a discrete, pre-defined set of possibilities. Each possible thickness has an associated weight, allowing designers to precisely control the probability distribution. For example, a designer can configure a layer to be 1 block thick 90% of the time, and 5 blocks thick the remaining 10% of the time.

This component strictly separates the concern of *layer thickness* from *layer material*. It calculates the thickness dimensionally, while the actual block or material type is delegated to an associated MaterialProvider. The use of a SeedGenerator ensures that the random selection is deterministic for any given world seed, producing repeatable and stable world generation results based on the (x, z) coordinate.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor, typically during the bootstrap phase of a parent SpaceAndDepthMaterialProvider. It is a configuration component, not a managed service.
- **Scope:** The object's lifetime is bound to the SpaceAndDepthMaterialProvider that contains it. It holds no global state and is intended to exist as part of a larger generator's configuration.
- **Destruction:** The object is eligible for garbage collection as soon as its parent provider is dereferenced. It does not manage any native resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state of the WeightedThicknessLayer is established at construction and is effectively immutable thereafter. The WeightedMap, SeedGenerator, and MaterialProvider references do not change during the object's lifetime.
- **Thread Safety:** This class is fully thread-safe. The core method, getThicknessAt, is pure and re-entrant. It instantiates a new FastRandom object for each invocation, seeded deterministically from the immutable SeedGenerator. All internal state is read-only. Consequently, a single WeightedThicknessLayer instance can be safely shared and accessed by multiple concurrent world generation threads without external locking.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getThicknessAt(...) | int | O(N) | Calculates the layer thickness for a given world column. The complexity is dependent on the WeightedMap.pick implementation, where N is the number of entries in the pool. Returns 0 if the thickness pool is empty. |
| getMaterialProvider() | MaterialProvider | O(1) | Returns the material provider associated with this layer. May return null if none was configured. |

## Integration Patterns

### Standard Usage
This class is designed to be composed within a larger world generation provider. The developer defines a weighted distribution of possible thickness values and provides it during construction.

```java
// Example: Configure a layer that is mostly thin, but sometimes thick.
WeightedMap<Integer> thicknessValues = new WeightedMap<>();
thicknessValues.add(1, 90.0); // 90% chance for a thickness of 1
thicknessValues.add(4, 10.0); // 10% chance for a thickness of 4

// Assume stoneProvider and worldSeedBox are already defined
MaterialProvider<Block> stoneProvider = ...;
SeedBox worldSeedBox = ...;

// Instantiate the layer
WeightedThicknessLayer<Block> stoneLayer = new WeightedThicknessLayer<>(
    thicknessValues,
    stoneProvider,
    worldSeedBox
);

// This layer would then be added to a SpaceAndDepthMaterialProvider instance.
```

### Anti-Patterns (Do NOT do this)
- **Empty Thickness Pool:** Constructing this class with an empty WeightedMap is a logical error. While the code handles this gracefully by always returning a thickness of 0, it results in a layer that will never generate any blocks. This is almost certainly not the intended behavior.
- **Stateful Material Providers:** Passing a non-thread-safe MaterialProvider to the constructor can break concurrency guarantees. Ensure any provided MaterialProvider is itself thread-safe, as it will be accessed by the same threads that call getThicknessAt.

## Data Pipeline
The data flow for this component is unidirectional, transforming a world coordinate into a discrete thickness value.

> Flow:
> World Generator requests thickness at (x, z) -> **WeightedThicknessLayer.getThicknessAt(x, z)** -> SeedGenerator computes a deterministic seed for the column -> A new FastRandom is created with the seed -> WeightedMap.pick selects an integer thickness based on the random value -> The integer thickness is returned to the World Generator.

