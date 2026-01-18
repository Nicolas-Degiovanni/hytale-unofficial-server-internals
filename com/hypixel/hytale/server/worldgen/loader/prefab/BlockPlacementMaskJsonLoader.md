---
description: Architectural reference for BlockPlacementMaskJsonLoader
---

# BlockPlacementMaskJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.prefab
**Type:** Transient

## Definition
```java
// Signature
public class BlockPlacementMaskJsonLoader extends JsonLoader<SeedStringResource, BlockPlacementMask> {
```

## Architecture & Concepts

The BlockPlacementMaskJsonLoader is a specialized parser within the server's procedural world generation framework. Its primary function is to translate a declarative JSON configuration into a highly optimized, in-memory BlockPlacementMask object. This mask defines the rules for which existing blocks can be replaced during the placement of prefabs or other generated structures.

Architecturally, this loader does not operate in isolation. It is a critical client of the **BlockPlacementMaskRegistry**, which acts as a central factory and cache. This integration is fundamental to the system's memory efficiency. The loader and registry collectively implement a **Flyweight pattern** for both the final BlockPlacementMask objects and their constituent rule entries. By interning and reusing identical mask configurations, the engine avoids storing thousands of duplicate rule sets in memory, which is essential for large-scale world generation.

The system employs a two-tier caching strategy:
1.  **File-Level Caching:** The `loadFileConstructor` method hooks into the registry's file cache. This prevents the same JSON file from being repeatedly read from disk and parsed into a JsonElement, reducing I/O overhead.
2.  **Object-Level Interning:** After parsing, the `load` method requests a mask from the registry via `retainOrAllocateMask`. The registry computes a hash of the mask's configuration and returns a shared, immutable instance if an identical one already exists.

This design ensures that even if hundreds of prefabs reference the same mask definition file, only one instance of the file is read from disk, and only one BlockPlacementMask object is held in memory.

### Lifecycle & Ownership
-   **Creation:** An instance is created on-demand by a higher-level component of the world generation pipeline, typically when a prefab definition or another procedural element references a block placement mask by its JSON definition. It is a short-lived worker object.
-   **Scope:** The loader's scope is confined to a single `load` operation. It is instantiated, its `load` method is called once, and it is then immediately eligible for garbage collection.
-   **Destruction:** The loader object is garbage collected after the `load` method returns and all references to it are dropped. The resulting BlockPlacementMask object it produces is **not** owned by the loader; its lifecycle is managed by the BlockPlacementMaskRegistry and persists as long as it is referenced by game assets.

## Internal State & Concurrency
-   **State:** This class is stateful. It maintains references to the input `seed`, `dataFolder`, and `json` element provided at construction. It also contains a mutable `fileName` field, which is set if the loader is initialized from a file path. This internal state is configured for a single, non-reentrant execution.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated and used within the context of a single asset-loading task. Concurrent access to the same instance will result in undefined behavior. The BlockPlacementMaskRegistry it interacts with is assumed to be thread-safe to support parallel asset loading.

## API Surface
The public API is minimal, centered on the constructor and the inherited `load` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BlockPlacementMaskJsonLoader(seed, dataFolder, json) | constructor | O(1) | Creates a new loader instance for a given JSON definition. |
| load() | BlockPlacementMask | O(N\*M) | Parses the JSON, interacts with the registry, and returns a cached or new BlockPlacementMask. Complexity is relative to N specific rules and M block variants per rule. |

## Integration Patterns

### Standard Usage
The loader is intended for immediate, single-shot use. It is constructed, used to load the mask, and then discarded. The resulting mask is then used by world generation algorithms.

```java
// How a developer should normally use this
// Assume 'seed', 'dataFolder', and 'maskJson' are provided by the engine.

BlockPlacementMaskJsonLoader loader = new BlockPlacementMaskJsonLoader(seed, dataFolder, maskJson);
BlockPlacementMask mask = loader.load();

// The 'mask' object is now ready for use in prefab placement logic.
// The 'loader' object is no longer needed.
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not retain an instance of the loader and call `load` multiple times. It is not designed for re-entrant use. Create a new instance for each JSON definition you need to parse.
-   **Bypassing the Registry:** Attempting to manually parse the JSON and construct a BlockPlacementMask object would defeat the entire memory optimization strategy. The loader's primary architectural value is its tight integration with the registry's flyweight cache.
-   **Concurrent Modification:** Do not access or use a loader instance from multiple threads. The internal state, particularly the `fileName` field, is not protected against race conditions.

## Data Pipeline
The loader transforms a raw JSON definition into a shared, memory-optimized game object through a clear, multi-stage pipeline involving the central registry.

> Flow:
> JSON File on Disk -> `BlockPlacementMaskRegistry` File Cache -> JsonElement in Memory -> **BlockPlacementMaskJsonLoader** (Parsing) -> Raw Mask Configuration -> `BlockPlacementMaskRegistry` (Interning) -> Shared BlockPlacementMask Object

