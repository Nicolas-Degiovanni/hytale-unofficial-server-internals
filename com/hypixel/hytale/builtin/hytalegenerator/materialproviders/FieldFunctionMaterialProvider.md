---
description: Architectural reference for FieldFunctionMaterialProvider
---

# FieldFunctionMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient

## Definition
```java
// Signature
public class FieldFunctionMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts
The FieldFunctionMaterialProvider is a fundamental component in the procedural world generation pipeline. It acts as a conditional router or a selector, translating a continuous scalar field into discrete material choices. Its primary function is to map ranges of values from a Density function (such as Perlin noise, elevation, or temperature) to specific, subordinate MaterialProviders.

This class enables designers to create stratified and layered environments. For example, a Density function representing altitude can be partitioned by this provider to assign different materials: water below a certain threshold, sand near the threshold, and grass or rock at higher values. It forms a bridge between abstract mathematical functions and concrete in-game materials.

It implements a form of the **Strategy** pattern, where the core algorithm (calculating a density value) is used to select another algorithm (a nested MaterialProvider) to execute.

## Lifecycle & Ownership
- **Creation:** FieldFunctionMaterialProvider is not a managed service. It is instantiated directly by higher-level configuration code, typically during the parsing and construction of a world generator's definition from an asset file (e.g., JSON).
- **Scope:** The object's lifetime is tied to the world generator configuration it is part of. As it is stateless beyond its initial configuration, a single instance is used for all voxel calculations within its designated generative context.
- **Destruction:** The object and its dependencies are eligible for garbage collection when the parent world generator is unloaded or replaced. It holds no persistent resources that require manual cleanup.

## Internal State & Concurrency
- **State:** The internal state, consisting of the Density function and the array of FieldDelimiters, is **effectively immutable**. The constructor creates a defensive copy of the incoming delimiter list, ensuring that the provider's configuration cannot be modified after instantiation.
- **Thread Safety:** This class is **inherently thread-safe**. Its immutable state and the re-entrant nature of the getVoxelTypeAt method allow it to be safely shared and executed by multiple world generation threads simultaneously.

    **WARNING:** While the class itself is thread-safe, the overall thread safety of a generation operation depends on the injected dependencies. The provided Density function and all nested MaterialProviders within the FieldDelimiters **must also be thread-safe**.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FieldFunctionMaterialProvider(density, delimiters) | constructor | O(N) | Constructs the provider. Complexity is linear to the number of delimiters due to list-to-array conversion. |
| getVoxelTypeAt(context) | V | O(N) | Calculates the density value and iterates through N delimiters to find a match. Returns the material from the first matching nested provider. |
| FieldDelimiter(materialProvider, bottom, top) | constructor | O(1) | Constructs a delimiter defining the half-open range [bottom, top). |

## Integration Patterns

### Standard Usage
This class is intended to be configured and composed during the setup of a world generator. The typical pattern involves defining a set of material layers and their corresponding value ranges, then instantiating the provider with those definitions.

```java
// Example: Creating a simple stratified provider based on a noise field
Density elevationNoise = new PerlinNoise(...);

List<FieldDelimiter<Block>> layers = List.of(
    // If noise is in [-1.0, 0.0), use Water
    new FieldDelimiter<>(new StaticMaterialProvider(WATER_BLOCK), -1.0, 0.0),
    // If noise is in [0.0, 0.1), use Sand
    new FieldDelimiter<>(new StaticMaterialProvider(SAND_BLOCK), 0.0, 0.1),
    // If noise is in [0.1, 1.0), use Grass
    new FieldDelimiter<>(new StaticMaterialProvider(GRASS_BLOCK), 0.1, 1.0)
);

MaterialProvider<Block> stratifiedProvider = new FieldFunctionMaterialProvider<>(elevationNoise, layers);

// This provider is now ready to be used in a larger generator
Block block = stratifiedProvider.getVoxelTypeAt(new MaterialProvider.Context(x, y, z));
```

### Anti-Patterns (Do NOT do this)
- **Overlapping Delimiters:** The provider uses a linear scan and returns on the first match. If delimiters have overlapping ranges, the one that appears earlier in the list will always be chosen, potentially making subsequent layers unreachable. Ensure ranges are distinct and ordered logically.
- **Gaps in Delimiters:** If the calculated density value falls into a gap not covered by any FieldDelimiter, getVoxelTypeAt will return null. This can lead to holes in the world or unexpected behavior in downstream systems. It is critical to ensure the delimiters cover the entire expected output range of the Density function.
- **Dynamic Modification:** Do not attempt to modify the list of delimiters after passing it to the constructor. The class makes an internal copy for stability and thread safety; any external changes will be ignored.

## Data Pipeline
The flow of data through this component is a clear, multi-stage process that transforms a world coordinate into a specific material.

> Flow:
> MaterialProvider.Context (x,y,z) -> Density.process() -> double (scalar value) -> **FieldFunctionMaterialProvider** (linear scan for range match) -> Nested MaterialProvider.getVoxelTypeAt() -> V (Final Voxel Material)

