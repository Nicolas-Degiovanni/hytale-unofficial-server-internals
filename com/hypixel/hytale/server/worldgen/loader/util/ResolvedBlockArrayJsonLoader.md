---
description: Architectural reference for ResolvedBlockArrayJsonLoader
---

# ResolvedBlockArrayJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.util
**Type:** Transient

## Definition
```java
// Signature
public class ResolvedBlockArrayJsonLoader extends JsonLoader<SeedStringResource, ResolvedBlockArray> {
```

## Architecture & Concepts
The ResolvedBlockArrayJsonLoader is a specialized parser within the server's world generation framework. Its primary function is to act as a translation layer, converting human-readable block and fluid definitions from JSON configuration files into a compact, integer-indexed `ResolvedBlockArray` object. This resulting object serves as an efficient block palette for procedural generation algorithms.

This loader is a critical component for decoupling world generation logic from concrete asset definitions. It allows designers to specify block arrangements in a flexible JSON format—supporting single blocks, complex objects with fluids, or arrays of mixed types—while the engine consumes a standardized, performant data structure.

The loader operates by referencing two global, static asset registries:
1.  **BlockType.getAssetMap():** To resolve string block names (e.g., "hytale:stone") into unique integer IDs.
2.  **Fluid.getAssetMap():** To resolve string fluid names into their corresponding integer IDs.

A key performance feature is its interaction with a global static cache, `ResolvedBlockArray.RESOLVED_BLOCKS`. This cache stores and retrieves pre-resolved, non-rotated block arrays to avoid redundant parsing and allocation for commonly used single blocks.

### Lifecycle & Ownership
-   **Creation:** An instance is created on-demand by a higher-level system responsible for parsing world generation or prefab configuration files. It is constructed with the specific `JsonElement` it is tasked with processing.
-   **Scope:** Short-lived. The object's lifetime is intended to span a single `load` operation. It is a single-use worker object.
-   **Destruction:** The instance becomes eligible for garbage collection immediately after the `load` method returns its `ResolvedBlockArray` result. No external system is expected to hold a long-term reference to the loader itself.

## Internal State & Concurrency
-   **State:** An instance of ResolvedBlockArrayJsonLoader is effectively immutable after construction. It holds final references to the input `seed`, `dataFolder`, and `json`. The core `load` method is a transformation function that does not modify instance state. However, it reads from and writes to the global static cache located at `ResolvedBlockArray.RESOLVED_BLOCKS`.

-   **Thread Safety:** **This class is not thread-safe.** While individual instances are isolated, the static `loadSingleBlock` methods perform non-atomic check-then-act operations on the shared static cache.
    > **Warning:** Concurrent invocations attempting to load the same, previously uncached, non-rotated block will result in a race condition. Multiple threads may pass the cache check, independently create a new `ResolvedBlockArray`, and overwrite each other's entries in the cache. All calls to this loader from a multi-threaded context **must** be externally synchronized.

## API Surface
The public API is focused on the primary `load` method inherited from its parent and two static utility methods for handling single-block JSON structures.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ResolvedBlockArray | O(N) | Parses the `JsonElement` provided at construction. N is the number of entries in the JSON array. Throws `IllegalArgumentException` if any block or fluid keys are not found in the asset maps. |
| loadSingleBlock(String) | ResolvedBlockArray | O(1) | **Static.** Parses a single block name string. Interacts with the global cache for non-rotated blocks. |
| loadSingleBlock(JsonObject) | ResolvedBlockArray | O(1) | **Static.** Parses a single JSON object defining a block and/or fluid. Interacts with the global cache for non-rotated, non-fluid blocks. |

## Integration Patterns

### Standard Usage
This loader is intended to be instantiated and used immediately to process a JSON fragment during a larger configuration loading sequence.

```java
// Assume 'worldGenConfig' is a JsonObject loaded from a file
// and 'seed' is the current world generation seed.

JsonElement blockPaletteJson = worldGenConfig.get("surfaceLayerPalette");

// Create a loader for this specific JSON element and execute the load
ResolvedBlockArrayJsonLoader loader = new ResolvedBlockArrayJsonLoader(seed, dataPath, blockPaletteJson);
ResolvedBlockArray surfacePalette = loader.load();

// The 'surfacePalette' can now be used by world generation algorithms.
// The 'loader' instance is no longer needed.
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not retain an instance of ResolvedBlockArrayJsonLoader for later use. It is designed as a single-shot parser for the `JsonElement` it was constructed with. Create a new instance for each distinct parsing task.
-   **Unsynchronized Concurrent Loading:** Do not invoke `load` or the static `loadSingleBlock` methods from multiple threads without an external locking mechanism. This will lead to race conditions on the internal static cache, causing unpredictable behavior and potential memory leaks.

## Data Pipeline
The loader functions as a specific stage in the data pipeline that transforms raw configuration data into an engine-ready format for world generation.

> Flow:
> Worldgen Config File (JSON) -> GSON Parser -> `JsonElement` -> **ResolvedBlockArrayJsonLoader** -> `ResolvedBlockArray` (In-Memory Palette) -> Voxel Placement Routines

