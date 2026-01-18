---
description: Architectural reference for MaterialCache
---

# MaterialCache

**Package:** com.hypixel.hytale.builtin.hytalegenerator.material
**Type:** Scoped Service / Factory

## Definition
```java
// Signature
public class MaterialCache {
```

## Architecture & Concepts
The MaterialCache is a specialized factory and cache system designed to manage the lifecycle of material objects within the world generation pipeline. Its primary architectural purpose is to implement the **Flyweight design pattern**. By ensuring that only one unique instance of any given material (e.g., stone, dirt, water at level 7) exists in memory, it dramatically reduces the memory footprint required for world generation and storage.

This class acts as the single source of truth for `Material`, `SolidMaterial`, and `FluidMaterial` objects during a generation task. Instead of generators instantiating new material objects for every block, they request them from the cache. The cache uses a hashing strategy based on material properties (block ID, rotation, fluid level, etc.) to either retrieve a pre-existing instance from its internal maps or create, store, and return a new one.

The public final fields like `EMPTY_AIR` and `ROCK_STONE` serve as pre-warmed, high-frequency access points, bootstrapping the cache with the most common materials required by nearly all world generators.

## Lifecycle & Ownership
- **Creation:** An instance of MaterialCache is created via its public constructor, typically by a high-level world generation orchestrator or context object at the beginning of a generation session. During construction, the cache is immediately populated with a set of common, default materials.
- **Scope:** The object's lifetime is tightly coupled to a single world generation task. It is **not a global singleton**. A new MaterialCache should be created for each distinct and isolated generation process.
- **Destruction:** The object holds no native resources and has no explicit destruction method. It becomes eligible for garbage collection as soon as the world generation task completes and the orchestrator that created it releases its reference.

## Internal State & Concurrency
- **State:** The internal state is highly **mutable**. It consists of three primary `HashMap` collections that store the canonical instances of solid, fluid, and combined materials. These maps grow dynamically throughout the world generation lifecycle as new, unique material variants are requested.

- **Thread Safety:** **CRITICAL WARNING:** This class is **not thread-safe**. The internal use of `HashMap` without any synchronization primitives makes it vulnerable to race conditions and data corruption if accessed concurrently. All method calls on a single MaterialCache instance **must** be externally synchronized or confined to a single thread. Failure to do so will lead to unpredictable behavior, including `ConcurrentModificationException` and inconsistent material states.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMaterial(solid, fluid) | Material | O(1) | Returns a canonical Material instance for the given solid and fluid combination. |
| getFluidMaterial(fluidString) | FluidMaterial | O(1) | Returns a canonical FluidMaterial from a string asset name. Defaults to UNKNOWN_FLUID on failure. |
| getFluidMaterial(fluidId, level) | FluidMaterial | O(1) | Returns a canonical FluidMaterial from a numeric ID and fluid level. |
| getSolidMaterial(solidString) | SolidMaterial | O(1) | Returns a canonical, non-rotated SolidMaterial from a string asset name. |
| getSolidMaterial(id, support, rot, filler, holder) | SolidMaterial | O(1) | The core factory method for retrieving any canonical SolidMaterial based on all its properties. |
| getSolidMaterialRotatedY(solid, rotation) | SolidMaterial | O(1) | Returns a new or cached SolidMaterial that represents the input material rotated along the Y-axis. |

## Integration Patterns

### Standard Usage
The MaterialCache is intended to be instantiated once per generation context and passed down to various generator components.

```java
// In a world generator orchestrator
MaterialCache materialCache = new MaterialCache();

// In a terrain generation pass
public void generateTerrain(WorldGenContext context, MaterialCache cache) {
    // Request a common, pre-cached material
    Material stone = cache.getMaterial(cache.ROCK_STONE, cache.EMPTY_FLUID);
    context.setBlock(x, y, z, stone);

    // Request a more complex, dynamically-created material
    FluidMaterial deepWater = cache.getFluidMaterial("Fluid_Water", (byte) 8);
    Material waterBlock = cache.getMaterial(cache.EMPTY_AIR, deepWater);
    context.setBlock(x, y+1, z, waterBlock);
}
```

### Anti-Patterns (Do NOT do this)
- **Global Singleton:** Do not store MaterialCache in a static global field. This will cause state from one world generation to leak into another, leading to severe and hard-to-debug errors. Each generation task requires a clean, isolated cache.
- **Concurrent Access:** Do not share a single MaterialCache instance across multiple threads without implementing an external locking mechanism. The internal maps are not thread-safe.

```java
// BAD: Sharing a cache across parallel tasks without locking
MaterialCache sharedCache = new MaterialCache();
ExecutorService executor = Executors.newFixedThreadPool(4);

for (int i = 0; i < 4; i++) {
    // This will cause race conditions inside the cache's HashMaps
    executor.submit(() -> generateChunk(sharedCache));
}
```

## Data Pipeline
The primary data flow is a request-response cycle where descriptive properties are converted into a canonical, memory-efficient object instance.

> Flow:
> World Generator Request (e.g., "Rock_Stone", rotation=0) -> `getSolidMaterial()` -> Asset Map Lookup (String to Block ID) -> **MaterialCache** (Hash Calculation & Map Lookup) -> Return Cached or New `SolidMaterial` Instance

