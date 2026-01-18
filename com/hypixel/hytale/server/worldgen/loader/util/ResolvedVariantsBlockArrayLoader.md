---
description: Architectural reference for ResolvedVariantsBlockArrayLoader
---

# ResolvedVariantsBlockArrayLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.util
**Type:** Transient

## Definition
```java
// Signature
public class ResolvedVariantsBlockArrayLoader extends JsonLoader<SeedStringResource, ResolvedBlockArray> {
```

## Architecture & Concepts

The ResolvedVariantsBlockArrayLoader is a specialized, low-level component within the server's world generation framework. It acts as a deserializer, translating abstract block definitions from JSON configuration files into a concrete, runtime-optimized data structure: the ResolvedBlockArray.

Its primary architectural function is to bridge the gap between human-readable worldgen definitions and the engine's internal block ID system. The loader is responsible for a critical optimization feature: **variant expansion**. A worldgen designer can specify a base block name, such as "stone", in a JSON file. This loader queries the engine's central asset registry, the BlockTypeAssetMap, to find all registered variants of "stone" (e.g., stone stairs, stone slabs, polished stone) and consolidates them into a single array.

This component operates as part of the larger `procedurallib.json` loading pattern, inheriting from JsonLoader. It makes heavy use of a global, static cache to avoid redundant variant resolution, which is critical for server startup performance and reducing memory churn during world generation.

## Lifecycle & Ownership

-   **Creation:** An instance is created by the procedural generation framework whenever it encounters a JSON structure that needs to be interpreted as a collection of block types. It is not intended for direct instantiation by gameplay systems.
-   **Scope:** The lifecycle of a ResolvedVariantsBlockArrayLoader instance is extremely short. It is created for the sole purpose of executing the `load` method. Once the method returns its ResolvedBlockArray, the loader instance has no further purpose and is eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java garbage collector. No manual cleanup is necessary.

## Internal State & Concurrency

-   **State:** An instance of this class holds immutable references to the `seed`, `dataFolder`, and the source `json` element provided during construction. The more significant state it interacts with is the static, mutable, and globally shared cache located at `ResolvedBlockArray.RESOLVED_BLOCKS_WITH_VARIANTS`. This cache persists for the lifetime of the server process.

-   **Thread Safety:** This class is **not thread-safe**. The instance method `load` and the static methods `loadSingleBlock` both read from and write to a shared static cache without any explicit locking.

    **WARNING:** Invoking these methods from multiple threads concurrently can lead to race conditions and unpredictable caching behavior. Any system using this loader in a multi-threaded context, such as a parallelized world generation pre-computation step, must provide its own external synchronization around calls to this loader.

## API Surface

The public API consists of one instance method for fulfilling the JsonLoader contract and several static utility methods for direct, ad-hoc loading.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ResolvedBlockArray | O(N * V) | Parses the JSON provided at construction. N is the number of block entries in the JSON array; V is the average number of variants per block. |
| loadSingleBlock(String) | ResolvedBlockArray | O(V) | *Static*. Resolves a single block name and its variants. Complexity becomes O(1) if the result is already cached. |
| loadSingleBlock(JsonObject) | ResolvedBlockArray | O(V) | *Static*. Resolves a block and optional fluid from a JSON object. Complexity becomes O(1) if the result is cached. |
| resolveBlockArrayWithVariants(...) | BlockFluidEntry[] | O(V) | *Static*. Core logic for querying the asset map for all variants of a base block name. |

## Integration Patterns

### Standard Usage

This class is designed to be used internally by the world generation asset loading system. A developer will typically define a block palette in a JSON file, which the engine then passes to this loader behind the scenes.

```java
// Hypothetical framework code that uses the loader
JsonElement blockPaletteJson = parse("[\"stone\", \"grass\"]");
SeedString<SeedStringResource> seed = ...;
Path dataFolder = ...;

// The framework instantiates the loader for a specific task
ResolvedVariantsBlockArrayLoader loader = new ResolvedVariantsBlockArrayLoader(seed, dataFolder, blockPaletteJson);

// The result is a ready-to-use array for worldgen algorithms
ResolvedBlockArray blockArray = loader.load();
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ResolvedVariantsBlockArrayLoader()` in high-level game logic. This is a low-level framework component. Define your data in JSON files and let the engine's asset pipeline manage loading.
-   **Concurrent Cache Access:** Do not call the static `loadSingleBlock` methods from multiple threads without external locks. The shared cache is not internally synchronized and can be corrupted.
-   **Ignoring Exceptions:** The loader will throw an IllegalArgumentException if a block name defined in JSON does not exist in the BlockTypeAssetMap. This is a fatal error indicating a data misconfiguration and must be handled during server startup.

## Data Pipeline

The loader functions as a specific step in the data transformation pipeline that converts worldgen source files into usable in-memory data for procedural algorithms.

> Flow:
> Worldgen JSON File -> Framework JSON Parser -> JsonElement -> **ResolvedVariantsBlockArrayLoader** -> ResolvedBlockArray (written to/read from global cache) -> World Generation Algorithm

