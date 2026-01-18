---
description: Architectural reference for BlockSetModule
---

# BlockSetModule

**Package:** com.hypixel.hytale.server.core.modules.blockset
**Type:** Singleton

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public class BlockSetModule extends JavaPlugin {
```

## Architecture & Concepts

The BlockSetModule is a server-side plugin responsible for processing and caching **BlockSet** definitions. A BlockSet is a logical collection of blocks, defined declaratively in asset files through a series of inclusion and exclusion rules based on properties like block name, group, or category.

This module's primary architectural purpose is to serve as a performance-oriented cache. It "flattens" the complex, rule-based BlockSet definitions into a simple, fast-lookup data structure: a map of BlockSet IDs to a final, calculated set of BlockType IDs. This pre-computation avoids the expensive process of evaluating BlockSet rules repeatedly at runtime.

The module operates reactively, subscribing to asset loading events. When BlockType or BlockSet assets are loaded or reloaded by the server, this module intercepts the corresponding `LoadedAssetsEvent` and completely rebuilds its internal cache. This ensures that game logic querying the module always has access to the most up-to-date block groupings.

**WARNING:** This class is annotated as `@Deprecated(forRemoval = true)`. It represents a legacy system for block grouping and should not be used for new feature development. Its functionality is expected to be replaced by a more robust system in future engine versions.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's `PluginManager` during the server bootstrap sequence. The static `INSTANCE` field is set within the constructor, enforcing the Singleton pattern.
-   **Scope:** The instance persists for the entire lifetime of the server process.
-   **Destruction:** The object is dereferenced and eligible for garbage collection during server shutdown when the `PluginManager` unloads all plugins.

## Internal State & Concurrency
-   **State:** The module's state is mutable and centered around its pre-computed cache.
    -   **flattenedBlockSets:** An `Int2ObjectMap<IntSet>` that stores the result of the flattening process. This is the core state of the module. This field is completely replaced during an asset reload.
    -   **unmodifiableFlattenedBlockSets:** An unmodifiable wrapper around `flattenedBlockSets`. This is exposed via the public API to prevent external mutation of the cache.
    -   **blockSetLookupTable:** A transient helper object used during the cache rebuild process to efficiently map string identifiers (like block names or groups) to integer IDs.

-   **Thread Safety:** This class is **not thread-safe**. State modification occurs within event handlers (`onBlockTypesChanged`, `onBlockSetsChanged`). The Hytale engine's event bus is expected to deliver these events serially on a main or asset-processing thread. Concurrent access from multiple threads during an asset reload could lead to inconsistent reads. Public query methods like `blockInSet` read from the state that is being modified.

    **WARNING:** Do not invoke methods on this class from arbitrary threads. All interactions should originate from the main server thread to avoid race conditions with the asset reloading mechanism.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInstance() | BlockSetModule | O(1) | Retrieves the global singleton instance. |
| getBlockSets() | Int2ObjectMap | O(1) | Returns an unmodifiable view of the flattened block set cache. |
| blockInSet(int set, int blockId) | boolean | O(1) | Performs a fast lookup to check if a block ID is a member of a set ID. |
| blockInSet(int set, BlockType blockType) | boolean | O(1) | Convenience overload for checking a BlockType object. |
| blockInSet(int set, String blockTypeKey) | boolean | O(1) | Convenience overload for checking a block by its string key. Throws IllegalArgumentException if the key is unknown. |

## Integration Patterns

### Standard Usage
The primary use case is to perform efficient checks against logical block groupings in game code, such as for tool effectiveness, mob spawning rules, or world generation logic.

```java
// How a developer should normally use this
BlockSetModule module = BlockSetModule.getInstance();
int stoneBlockSetId = BlockSet.getAssetMap().getIndex("namespace:stone_variants");
int dioriteBlockId = BlockType.getAssetMap().getIndex("namespace:diorite");

// Check if diorite is considered a "stone variant"
if (module.blockInSet(stoneBlockSetId, dioriteBlockId)) {
    // Game logic here...
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BlockSetModule()`. The module is managed by the plugin system. Always use `BlockSetModule.getInstance()`.
-   **State Modification:** Do not attempt to cast or use reflection to modify the map returned by `getBlockSets()`. It is intended to be read-only.
-   **Premature Access:** Querying the module before the initial `LoadedAssetsEvent` has been processed will yield empty or incorrect results. Logic should ensure assets are loaded first.
-   **New Development:** Do not build new systems that rely on this module. Its deprecated status means it will be removed, and any dependency will break in a future update.

## Data Pipeline
The module functions as a transformation and caching step in the server's asset processing pipeline. It converts declarative, rule-based data into an optimized, queryable in-memory format.

> Flow:
> BlockSet/BlockType Asset Files -> AssetStore Loader -> `LoadedAssetsEvent` -> **BlockSetModule** (Receives Event & Flattens Rules) -> Internal `Int2ObjectMap` Cache -> Game Logic (via `blockInSet` queries)

