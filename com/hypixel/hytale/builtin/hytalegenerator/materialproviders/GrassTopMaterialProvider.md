---
description: Architectural reference for GrassTopMaterialProvider
---

# GrassTopMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient

## Definition
```java
// Signature
public class GrassTopMaterialProvider extends MaterialProvider<SolidMaterial> {
```

## Architecture & Concepts
The GrassTopMaterialProvider is a concrete implementation of the *Strategy* pattern, providing a specific algorithm for voxel material selection within the procedural world generation framework. Its sole responsibility is to define the vertical layering of materials for the top surface of the world, such as the transition from grass to dirt to stone.

This class is a fundamental building block for biome definition. It encapsulates the simple but common rule of "one layer of grass, a few layers of dirt, then stone". By abstracting this logic, different biomes can compose their surfaces by creating instances of this provider with different materials (e.g., sand instead of grass for a desert, mycelium for a mushroom biome). It acts as a function that maps a vertical depth coordinate to a specific material.

## Lifecycle & Ownership
- **Creation:** Instantiated directly by a higher-level world generation component, typically a Biome or Terrain generator. It is configured at creation time with the specific materials it will provide. It is not managed by a dependency injection container or service locator.
- **Scope:** The object's lifetime is tied to the parent generator that created it. It is a short-lived, configuration-based object used during the voxel-filling phase of chunk generation.
- **Destruction:** Eligible for garbage collection as soon as the reference held by the parent generator is released. There is no explicit cleanup or disposal method.

## Internal State & Concurrency
- **State:** The internal state is **immutable**. The four SolidMaterial fields (grass, dirt, stone, empty) are final and set exclusively via the constructor. The object acts as a read-only configuration holder after instantiation.
- **Thread Safety:** This class is unconditionally **thread-safe**. Its primary method, getVoxelTypeAt, is a pure function that depends only on its immutable internal state and the provided Context argument. It contains no locks, synchronization, or mutable state, making it safe for parallel execution by multiple world generation worker threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | SolidMaterial | O(1) | Returns the appropriate SolidMaterial based on the depthIntoFloor field of the context. This is the core operational method. |

## Integration Patterns

### Standard Usage
This provider is intended to be constructed with pre-resolved materials and used by a terrain generator to populate a vertical column of voxels.

```java
// In a Biome or Terrain Generator during initialization
SolidMaterial grassBlock = materialRegistry.get("core:grass");
SolidMaterial dirtBlock = materialRegistry.get("core:dirt");
SolidMaterial stoneBlock = materialRegistry.get("core:stone");
SolidMaterial airBlock = materialRegistry.get("core:air");

MaterialProvider surfaceProvider = new GrassTopMaterialProvider(grassBlock, dirtBlock, stoneBlock, airBlock);

// During chunk generation loop
MaterialProvider.Context generationContext = new MaterialProvider.Context(currentY, depth);
SolidMaterial materialToPlace = surfaceProvider.getVoxelTypeAt(generationContext);
chunk.setVoxel(x, y, z, materialToPlace);
```

### Anti-Patterns (Do NOT do this)
- **Null Injection:** Do not construct this provider with null material arguments. The constructor is annotated with Nonnull, and doing so will cause a NullPointerException deep within the world generation pipeline, which can be difficult to debug.
- **Over-Complication:** Do not attempt to subclass this provider to add complex logic. If more sophisticated layering is required (e.g., based on noise, moisture), a different, more capable MaterialProvider implementation should be used or created. This class is designed for a simple, fixed-depth transition.

## Data Pipeline
The GrassTopMaterialProvider is a single, simple step in the larger voxel generation pipeline. It consumes a context object and produces a material.

> Flow:
> Terrain Generator -> Calculates Voxel Position and Depth -> Creates MaterialProvider.Context -> **GrassTopMaterialProvider.getVoxelTypeAt()** -> Returns SolidMaterial -> Voxel Data written to Chunk Buffer

