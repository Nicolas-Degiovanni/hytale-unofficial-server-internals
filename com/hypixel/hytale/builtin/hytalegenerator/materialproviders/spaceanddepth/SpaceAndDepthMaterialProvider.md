---
description: Architectural reference for SpaceAndDepthMaterialProvider
---

# SpaceAndDepthMaterialProvider<V>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth
**Type:** Transient / Configuration Object

## Definition
```java
// Signature
public class SpaceAndDepthMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts

The SpaceAndDepthMaterialProvider is a fundamental component of the procedural world generation engine. It is not a general-purpose material provider; its sole function is to translate vertical depth into a specific material, creating stratified layers of voxels like soil, dirt, clay, and stone beneath a surface.

This class operates as a strategic selector. It is configured with a sequence of layers, each with a specific thickness and material. When queried for a voxel at a given position, it measures the distance from a reference surface (either a floor or a ceiling) and iterates through its configured layers to determine which material is appropriate for that specific depth.

Its behavior is controlled by three primary inputs:
1.  **LayerContextType:** Determines whether depth is measured downwards from a floor or upwards from a ceiling.
2.  **Condition:** A predicate that acts as a gate. If the condition is not met for a given location, the provider does nothing, allowing other systems to act.
3.  **Layers:** An ordered list of strata, each defining its thickness and the material it contains. Thickness can be dynamic, allowing for non-uniform layers.

The generic parameter V indicates that this provider can return any voxel data type, from a simple identifier to a complex block state object.

### Lifecycle & Ownership
-   **Creation:** Instances are not created manually. They are instantiated by the world generator's configuration loader, typically by deserializing biome or feature definition files (e.g., JSON). The presence of a Codec in the LayerContextType enum is a strong indicator of this data-driven approach.
-   **Scope:** The object's lifetime is tied to a specific world generation task. It is configured once when a biome or feature is loaded and is then used repeatedly for all voxel placements governed by that feature. It is not a global singleton.
-   **Destruction:** The object is eligible for garbage collection once the world generation configuration it belongs to is unloaded or the generation task is complete.

## Internal State & Concurrency
-   **State:** The internal state, including the list of layers, the condition, and the context type, is set during construction and is **effectively immutable**. The provider does not modify its own state during operation.
-   **Thread Safety:** This class is **thread-safe** and designed for high-concurrency environments. The getVoxelTypeAt method is a pure function with respect to the provider's state; it only reads its immutable configuration and the provided Context object. This is critical for the performance of the world generator, which processes many chunks in parallel.

    **Warning:** While the provider itself is thread-safe, the contained Layer and Condition implementations must also be thread-safe for the system to be stable.

## API Surface

The public contract is focused exclusively on the voxel retrieval operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | V | O(N) | Resolves the voxel type for the given context. N is the number of layers. Returns null if the position is outside maxDistance, fails the condition, or falls into a gap between layers. |

## Integration Patterns

### Standard Usage

This provider is intended to be defined within data files and used by the world generation engine. A developer would typically define a list of layers in a biome configuration file, which the engine then deserializes into a SpaceAndDepthMaterialProvider instance.

```java
// Conceptual example of engine-level usage.
// Developers do not typically write this code.

// 1. Engine deserializes a biome's configuration into this provider.
SpaceAndDepthMaterialProvider<BlockState> groundProvider = loadProviderFromConfig("zone1_forest_floor.json");

// 2. During chunk generation, the engine calculates context for a specific block.
MaterialProvider.Context context = worldGenerator.calculateContextFor(x, y, z);

// 3. The engine invokes the provider to get the material.
BlockState state = groundProvider.getVoxelTypeAt(context);

if (state != null) {
    world.setBlockState(x, y, z, state);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SpaceAndDepthMaterialProvider()` in gameplay logic. This is a configuration object for world generation and should be defined in data assets.
-   **Misuse for Non-Layered Structures:** Attempting to use this provider for anything other than vertical, stratified material placement is an anti-pattern. For complex or sparse patterns, use a different provider like a NoiseMaterialProvider or a StructureProvider.
-   **Stateful Layers or Conditions:** Do not inject Layer or Condition implementations that rely on or modify mutable shared state. This will break thread safety and lead to severe, non-deterministic generation bugs.

## Data Pipeline

The provider acts as a filter and selector within the larger world generation data flow. It transforms spatial context into a concrete material choice.

> Flow:
> World Generator Request for (X,Y,Z) -> Context Calculation Engine -> MaterialProvider.Context -> **SpaceAndDepthMaterialProvider** -> Selects Matching Layer -> Layer's MaterialProvider -> Final Voxel Type (V)

