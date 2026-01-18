---
description: Architectural reference for TerrainDensityMaterialProvider
---

# TerrainDensityMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient

## Definition
```java
// Signature
public class TerrainDensityMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts
The TerrainDensityMaterialProvider is a fundamental component within the procedural world generation pipeline. It functions as a specialized implementation of the *Strategy Pattern*, designed to select a final voxel material based on a continuous density value. This value is typically sourced from a 3D noise field, such as Perlin or Simplex noise, which defines the raw shape of the terrain.

This class acts as a **conditional router** or a **switch**. Its primary role is to translate a single floating-point density value into a specific material layer. For example, it can be configured to return stone for high density values (deep underground), dirt for medium values (near the surface), and grass for a narrow band at the top.

It achieves this by maintaining a list of **FieldDelimiter** objects, each defining a density range (`bottom` to `top`) and associating it with another MaterialProvider. When queried, it finds the appropriate range and delegates the final material resolution to the corresponding provider. This design enables a highly data-driven and composable approach to defining complex, layered terrain for different biomes without hardcoding material logic.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level world generator or biome configuration system during the initialization phase of world generation. It is configured with a list of FieldDelimiter objects, which are typically loaded from world generation asset files (e.g., JSON definitions).
- **Scope:** The object's lifetime is bound to the world generator instance that created it. It is not a global singleton and persists as long as its parent generator is in use for generating chunks.
- **Destruction:** The object is eligible for garbage collection when the world generator that owns it is discarded. It does not manage any native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state consists of the `fieldDelimiters` array. This state is **effectively immutable**. The constructor performs a defensive copy of the incoming list into a private final array, preventing any external modification after instantiation.
- **Thread Safety:** This class is **thread-safe** and designed for high-concurrency environments. Its immutable state guarantees that multiple world generation worker threads can safely call `getVoxelTypeAt` on the same instance without locks or race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TerrainDensityMaterialProvider(delimiters) | constructor | O(N) | Constructs the provider. Throws IllegalArgumentException if the list contains nulls. |
| getVoxelTypeAt(context) | V | O(N) | Resolves a voxel type by finding the first delimiter whose range contains the context density. Returns null if no range matches. |

*N = number of configured FieldDelimiter objects.*

## Integration Patterns

### Standard Usage
This provider is typically assembled by a factory or builder responsible for constructing the complete material generation pipeline for a biome. It is configured once and then used repeatedly by worker threads.

```java
// Example configuration for a simple terrain profile
List<TerrainDensityMaterialProvider.FieldDelimiter<BlockState>> delimiters = List.of(
    // Deep underground: high density -> stone
    new FieldDelimiter<>(new ConstantMaterialProvider<>(STONE), 0.5, 1.0),
    // Sub-surface: medium density -> dirt
    new FieldDelimiter<>(new ConstantMaterialProvider<>(DIRT), 0.0, 0.5),
    // Surface layer: low density -> grass
    new FieldDelimiter<>(new ConstantMaterialProvider<>(GRASS), -0.1, 0.0)
);

// The provider is created once per generator configuration
MaterialProvider<BlockState> densityProvider = new TerrainDensityMaterialProvider<>(delimiters);

// In the generation loop, for each voxel:
// context.density is calculated from a noise function
BlockState state = densityProvider.getVoxelTypeAt(context);
```

### Anti-Patterns (Do NOT do this)
- **Overlapping Ranges:** Configuring the provider with `FieldDelimiter` objects whose density ranges overlap is a critical error. The system uses a simple linear scan and will always return the material from the *first* matching range it finds. This can lead to non-deterministic or incorrect material layers. Ranges must be validated to be contiguous and non-overlapping.
- **Gaps in Ranges:** Leaving gaps between the density ranges of the delimiters will cause the `getVoxelTypeAt` method to return null for any density value that falls within a gap. This can result in holes or empty voxels in the generated world.

## Data Pipeline
The provider sits in the middle of the material selection pipeline, acting as a crucial decision point based on terrain shape.

> Flow:
> 3D Noise Function -> Density Value -> `MaterialProvider.Context` -> **TerrainDensityMaterialProvider** -> Delegate `MaterialProvider` -> Final Voxel Type `V`

