---
description: Architectural reference for HorizontalMaterialProvider
---

# HorizontalMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient Component

## Definition
```java
// Signature
public class HorizontalMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts
The HorizontalMaterialProvider is a specialized implementation of the MaterialProvider contract that functions as a **decorator** or **filter** within the world generation data pipeline. Its primary architectural role is to constrain the output of a wrapped MaterialProvider to a specific vertical slice of the world, defined by dynamic upper and lower Y-boundaries.

This component is a fundamental building block for creating layered geology and biomes. Unlike a simple conditional check, the boundaries are not static values but are calculated by DoubleFunctionXZ functions. This allows the horizontal "slab" to follow complex, non-flat surfaces, such as terrain noise, water tables, or biome-specific strata.

For example, a HorizontalMaterialProvider can be used to ensure a layer of dirt only appears between a noise-defined grass surface and a fixed-level stone base. By chaining multiple providers, complex geological cross-sections can be constructed.

### Lifecycle & Ownership
- **Creation:** Instantiated during the configuration phase of a world generator. It is not created by the core engine bootstrap but is instead defined and composed by world generation scripts to form a material generation graph.
- **Scope:** The lifetime of a HorizontalMaterialProvider instance is bound to the world generator configuration it is part of. It persists as long as that generator is active for a given world.
- **Destruction:** The object is marked for garbage collection when its parent world generator configuration is discarded. This typically occurs on server shutdown or when loading a new world with a different generator.

## Internal State & Concurrency
- **State:** This class is **effectively immutable**. Its internal fields, which consist of the wrapped provider and the boundary functions, are final and are assigned exclusively during construction. It performs no internal caching and holds no mutable state.
- **Thread Safety:** **Thread-safe**. Due to its immutable nature, an instance can be safely accessed by multiple world generation worker threads concurrently without external locking or synchronization.

    **WARNING:** While the HorizontalMaterialProvider itself is thread-safe, the overall thread safety of a call to getVoxelTypeAt depends on the thread safety of the wrapped MaterialProvider and the provided DoubleFunctionXZ implementations. These dependencies must also be thread-safe.

## API Surface
The public contract is minimal, consisting only of the inherited method it overrides from the MaterialProvider base class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | V (Nullable) | O(C) | Calculates top and bottom Y-boundaries using the provided functions. If the context's Y-position is within this range, it delegates to the wrapped provider; otherwise, it returns null, vetoing material placement. Complexity is dependent on the boundary functions. |

## Integration Patterns

### Standard Usage
This component is designed to be composed with other providers. The standard pattern is to wrap a base material provider to constrain its placement to a specific vertical region.

```java
// Standard Usage: Create a flat layer of stone between Y=10 and Y=40.

// 1. Define the base material to be placed.
MaterialProvider<VoxelType> stoneProvider = new ConstantMaterialProvider<>(VoxelType.STONE);

// 2. Define the top and bottom boundaries as simple lambda functions.
DoubleFunctionXZ topBoundary = (x, z) -> 40.0;
DoubleFunctionXZ bottomBoundary = (x, z) -> 10.0;

// 3. Wrap the stone provider with the HorizontalMaterialProvider.
MaterialProvider<VoxelType> stoneLayer = new HorizontalMaterialProvider<>(
    stoneProvider,
    topBoundary,
    bottomBoundary
);

// 4. The 'stoneLayer' provider can now be used by the world generator.
// It will only return STONE for coordinates where 10 <= Y < 40.
```

### Anti-Patterns (Do NOT do this)
- **Inverted Boundaries:** Configuring the provider where the `bottomY` function consistently returns values greater than the `topY` function. This creates a logically impossible state where the provider will never return a material, effectively creating a void. The system does not guard against this logical error.
- **High-Frequency Boundary Functions:** Using computationally expensive noise functions or complex calculations for the boundaries can create significant performance bottlenecks, as these functions are executed for every single voxel column check.

## Data Pipeline
The HorizontalMaterialProvider acts as a conditional gate in the material selection flow. If its conditions are not met, it terminates the chain for that position by returning null.

> Flow:
> World Generator Request (X,Y,Z) -> MaterialProvider.Context -> **HorizontalMaterialProvider** (Boundary Check) -> [Delegate if Y is in-bounds] -> Wrapped MaterialProvider -> Voxel Type (or null) -> Voxel Chunk Buffer

