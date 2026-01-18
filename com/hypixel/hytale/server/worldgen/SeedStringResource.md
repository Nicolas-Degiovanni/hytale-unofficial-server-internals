---
description: Architectural reference for SeedStringResource
---

# SeedStringResource

**Package:** com.hypixel.hytale.server.worldgen
**Type:** Transient

## Definition
```java
// Signature
public class SeedStringResource implements SeedResource {
```

## Architecture & Concepts
The SeedStringResource class serves as a central context object for a specific world generation session. It implements the SeedResource interface, signaling its role as a provider of essential resources and environmental information to the procedural generation library, procedurallib.

This class is not a data processor itself; rather, it is a stateful container that aggregates and provides access to critical world generation components. It acts as the bridge between the server's file system configuration (represented by a Path to a data folder) and the abstract procedural algorithms. Its primary responsibilities include:

1.  **Resource Loading:** It owns and manages the WorldGenPrefabLoader, which is responsible for loading prefab data from disk.
2.  **Condition Caching:** It holds registries for biome and block placement masks (FileMaskCache and BlockPlacementMaskRegistry). These are performance-critical caches that allow procedural rules to quickly evaluate placement conditions without repeated file I/O.
3.  **Buffer Access:** It delegates requests for temporary data buffers (ResultBuffer) to the active ChunkGenerator, tightly coupling its operational context to the chunk being generated.

In essence, an instance of SeedStringResource represents the complete environmental context required to execute any procedural generation task for a given world.

### Lifecycle & Ownership
-   **Creation:** An instance is created via its public constructor, typically by a higher-level world generation orchestrator at the beginning of a world creation or loading sequence. It requires a PrefabStoreRoot and a Path to the world's data folder, tying its existence to a specific world's assets.
-   **Scope:** The object's lifetime is bound to a single, active world generation session. While its internal state can be modified via setters, this is a destructive operation that re-initializes core components. It is not designed to be a long-lived, application-scoped service.
-   **Destruction:** The object holds no native resources and does not require explicit cleanup. It is eligible for garbage collection once the world generation orchestrator that created it releases its reference.

## Internal State & Concurrency
-   **State:** This class is highly **mutable**. Key internal fields, including the dataFolder and the WorldGenPrefabLoader, can be swapped out at runtime. The contained registries are also mutable, as they are populated on-demand to cache data loaded from disk.

-   **Thread Safety:** This class is **not thread-safe**. The public setters for prefabStore and dataFolder perform non-atomic re-initialization of the internal WorldGenPrefabLoader. Concurrent calls to these setters or simultaneous access to the loader while it is being replaced will result in unpredictable behavior and likely data corruption. All interactions with a SeedStringResource instance must be externally synchronized or confined to a single world generation thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SeedStringResource(store, path) | constructor | O(1) | Creates a new world generation context for a specific prefab store and data folder. |
| getLoader() | WorldGenPrefabLoader | O(1) | Returns the loader for accessing prefab data. |
| setPrefabStore(store) | void | O(1) | Re-initializes the internal prefab loader with a new store. **Warning:** Not a thread-safe operation. |
| setDataFolder(path) | void | O(1) | Re-initializes the internal prefab loader with a new data folder. **Warning:** Not a thread-safe operation. |
| localBounds2d() | ResultBuffer.Bounds2d | O(1) | Delegates to the active ChunkGenerator to retrieve the 2D bounds for the current operation. |
| localBuffer2d() | ResultBuffer.ResultBuffer2d | O(1) | Delegates to the active ChunkGenerator to retrieve the 2D buffer for the current operation. |
| localBuffer3d() | ResultBuffer.ResultBuffer3d | O(1) | Delegates to the active ChunkGenerator to retrieve the 3D buffer for the current operation. |
| getBiomeMaskRegistry() | FileMaskCache | O(1) | Returns the cache for biome placement conditions. |
| getBlockMaskRegistry() | BlockPlacementMaskRegistry | O(1) | Returns the registry for block placement masks. |

## Integration Patterns

### Standard Usage
A SeedStringResource should be instantiated once per world generation session and passed as a context object to procedural algorithms. These algorithms then use the instance to access loaders and registries as needed.

```java
// In a WorldGenerator or similar orchestrator
PrefabStoreRoot myPrefabStore = ...;
Path worldDataPath = Path.of("./world_data");

// Create a context for this specific world
SeedStringResource worldGenContext = new SeedStringResource(myPrefabStore, worldDataPath);

// Pass the context to procedural functions
BiomeGenerator.generate(worldGenContext, chunkPosition);
StructurePlacer.placeStructures(worldGenContext, region);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation During Generation:** Do not call setPrefabStore or setDataFolder while generation tasks are running. This will swap the underlying loader mid-operation, leading to inconsistent state and potential crashes. These methods should only be used during an explicit re-initialization phase.
-   **Concurrent Modification:** Do not access a SeedStringResource instance from multiple threads without external locking. The internal caches and loader are not designed for concurrent access.
-   **Global Singleton:** Do not treat this class as a global singleton. Each world or dimension with a distinct data folder or prefab set must have its own dedicated SeedStringResource instance.

## Data Pipeline
SeedStringResource does not process a stream of data itself. Instead, it serves as a central repository of resources that other pipeline stages query. It is a source of configuration and cached data, not a transformation stage.

> **Prefab Loading Flow:**
> Procedural Algorithm -> **SeedStringResource**.getLoader() -> WorldGenPrefabLoader -> Filesystem Read (Prefab JSON/Schematic) -> Prefab Object

> **Condition Checking Flow:**
> Placement Logic -> **SeedStringResource**.getBiomeMaskRegistry() -> FileMaskCache -> (Cache Miss) -> Filesystem Read (Mask Image) -> IIntCondition Object

