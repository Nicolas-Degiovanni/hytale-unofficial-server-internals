---
description: Architectural reference for WeightedMaterialProvider
---

# WeightedMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient

## Definition
```java
// Signature
public class WeightedMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts
The WeightedMaterialProvider is a fundamental component within the procedural world generation framework. It functions as a **Composite Selector** that resolves a material type by choosing from a collection of other MaterialProvider instances based on a deterministic, weighted probability.

Its primary role is to introduce controlled, repeatable randomness into material placement. Rather than a simple uniform distribution, this provider allows designers to specify that, for example, a patch of ground should be 80% grass, 15% dirt, and 5% gravel. This is achieved by composing it with simpler providers (e.g., three separate SingleMaterialProviders for grass, dirt, and gravel) and assigning each a weight.

This class is a key enabler for creating complex and natural-looking biomes. It acts as a routing mechanism, delegating the final responsibility of providing a material to another, more specialized provider. Its behavior is entirely determined by its initial configuration and the world coordinate being sampled.

### Lifecycle & Ownership
- **Creation:** WeightedMaterialProvider is not a managed service. It is instantiated directly during the configuration phase of a world generator, typically when parsing biome or feature definition files. It is a data-driven object, constructed with its full configuration passed into its constructor.
- **Scope:** The lifetime of a WeightedMaterialProvider instance is tied to the world generation configuration it is part of. It persists as long as that configuration is active and is used whenever a chunk belonging to the relevant biome or layer is generated.
- **Destruction:** Instances are subject to standard Java garbage collection once the world generator configuration is discarded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state, consisting of the WeightedMap, SeedGenerator, and noneProbability, is established in the constructor and is **effectively immutable** thereafter. The class does not modify its own fields during its operational lifetime.
- **Thread Safety:** This class is **thread-safe** and designed for concurrent use. The primary method, getVoxelTypeAt, is re-entrant and does not rely on shared mutable state. It creates a new FastRandom instance for each invocation, seeded deterministically from the world coordinates. This design is critical for the performance of the world generator, which processes multiple world chunks in parallel across different threads.

## API Surface
The public contract is inherited from the base MaterialProvider. The core logic is encapsulated in a single method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | V | O(log N) | Calculates a deterministic random value based on the context position and seed. Selects a nested MaterialProvider from its weighted map and delegates the call. Returns null if the initial probability check fails or the map is empty. Complexity depends on the WeightedMap.pick implementation. |

## Integration Patterns

### Standard Usage
This provider is intended to be composed with other providers to create sophisticated material selection logic. The typical pattern involves defining a map of providers and their weights, then creating the WeightedMaterialProvider as part of a larger generator setup.

```java
// Example: Configuring a ground cover layer for a biome
MaterialProvider<VoxelType> grassProvider = new SingleMaterialProvider<>(GRASS);
MaterialProvider<VoxelType> dirtProvider = new SingleMaterialProvider<>(DIRT);

// Create a weighted map defining the material distribution
WeightedMap<MaterialProvider<VoxelType>> groundCoverMap = new WeightedMap<>();
groundCoverMap.put(grassProvider, 80.0);
groundCoverMap.put(dirtProvider, 20.0);

// SeedBox is provided by the world generator framework
SeedBox biomeSeedBox = worldGenerator.getSeedBoxFor("my_biome_ground");

// Instantiate the provider with a 5% chance to place nothing
double noneProbability = 0.05;
MaterialProvider<VoxelType> groundCover = new WeightedMaterialProvider<>(
    groundCoverMap,
    biomeSeedBox,
    noneProbability
);

// This 'groundCover' provider is now used by a generation layer
layer.setMaterial(groundCover);
```

### Anti-Patterns (Do NOT do this)
- **Misunderstanding Determinism:** Do not call getVoxelTypeAt multiple times for the same coordinates expecting a different result. The choice is pseudo-random but is deterministic for a given seed and position. To get different results, you must use a different SeedBox.
- **Empty Map:** Constructing this provider with an empty WeightedMap will cause it to always return null. While this may be intentional, it is often a configuration error.
- **Stateful Delegates:** Do not populate the WeightedMap with MaterialProvider implementations that are stateful or not thread-safe. This will break the concurrency guarantees of the world generation engine.

## Data Pipeline
The WeightedMaterialProvider acts as a conditional router in the data flow of material generation. It transforms a coordinate input into a delegated request.

> Flow:
> World Generator Request for (X,Y,Z) -> SeedGenerator.seedAt(X,Y,Z) -> FastRandom(seed) -> **WeightedMaterialProvider** -> WeightedMap.pick(random) -> Selected MaterialProvider.getVoxelTypeAt(context) -> Final Voxel Type (V) or null

