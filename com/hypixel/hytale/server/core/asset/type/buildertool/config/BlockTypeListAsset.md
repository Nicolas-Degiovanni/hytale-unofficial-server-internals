---
description: Architectural reference for BlockTypeListAsset
---

# BlockTypeListAsset

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config
**Type:** Data Asset

## Definition
```java
// Signature
public class BlockTypeListAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, BlockTypeListAsset>> {
```

## Architecture & Concepts
The BlockTypeListAsset is a data-driven configuration object, not an active service. Its primary function is to represent a named collection of block types defined in an external JSON file. This class serves as a critical bridge between raw, declarative configuration (a simple list of block identifier strings) and a high-performance, runtime-optimized data structure, the BlockPattern.

The core of this asset's behavior is defined by its static **CODEC** field. This `AssetBuilderCodec` declaratively outlines the entire deserialization and processing pipeline:
1.  It maps a JSON object to a new BlockTypeListAsset instance.
2.  It specifically looks for a JSON key named "Blocks" containing an array of strings.
3.  Crucially, it uses an **afterDecode** hook to perform a transformation step. After the list of block keys is loaded, it constructs a BlockPattern. This pattern is a more complex data structure, likely used by server systems like builder tools or procedural generation to efficiently select or validate blocks from the defined list.

Instances of this class are not managed directly. They are loaded, cached, and owned by the global **AssetRegistry** system, accessible via the static `getAssetStore` method.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale AssetStore during the server's asset loading phase. The static CODEC is invoked to decode a corresponding JSON asset file into a fully-formed BlockTypeListAsset object. Manual instantiation is strictly forbidden.
-   **Scope:** Application-scoped. Once an asset is loaded, it is registered in the static ASSET_STORE and persists for the entire server session. It is effectively a global, read-only resource.
-   **Destruction:** The object and its associated data are garbage collected when the server shuts down and the AssetRegistry is cleared. There is no mechanism for manual destruction or unloading of a single asset.

## Internal State & Concurrency
-   **State:** The object is effectively **immutable** after its creation during the asset loading phase. The `afterDecode` hook populates all internal fields, including the computed BlockPattern. There are no public methods to mutate the state of an instance post-initialization.

-   **Thread Safety:** The instance itself is **thread-safe for reads**. Since its state does not change after the initial, single-threaded asset loading process, multiple threads can safely call `getBlockPattern` or `getBlockTypeKeys` concurrently.

    **WARNING:** The static `getAssetStore` method uses lazy initialization. While the asset loading process is typically synchronous and single-threaded at startup, invoking this method from multiple threads *before* the AssetRegistry is fully initialized could lead to a race condition. All systems should assume assets are fully loaded before attempting concurrent access.

## API Surface
The public API is designed for read-only access to the asset's data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, shared asset store for all BlockTypeListAsset instances. |
| getAssetMap() | static DefaultAssetMap | O(1) | A convenience method to get the asset map directly from the asset store. |
| getBlockPattern() | BlockPattern | O(1) | Returns the pre-computed, runtime-optimized pattern representing the list of blocks. |
| getBlockTypeKeys() | HashSet<String> | O(1) | Returns the raw set of block identifier strings loaded from the source JSON. |
| getId() | String | O(1) | Returns the unique asset key for this list (e.g., "hytale:stone_variants"). |

## Integration Patterns

### Standard Usage
Systems should always retrieve assets from the static, globally-available asset map. The primary use case is to fetch a specific list by its key and then use the resulting BlockPattern for game logic.

```java
// How a developer should normally use this
DefaultAssetMap<String, BlockTypeListAsset> allLists = BlockTypeListAsset.getAssetMap();

// Retrieve a specific list by its unique asset key
BlockTypeListAsset stoneList = allLists.get("hytale:stone_variants");

if (stoneList != null) {
    // Use the pre-computed BlockPattern for efficient operations,
    // such as selecting a random block from the list.
    BlockPattern pattern = stoneList.getBlockPattern();
    // ... logic that uses the pattern ...
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new BlockTypeListAsset()`. This bypasses the asset loading pipeline, resulting in an uninitialized, unregistered object that will cause NullPointerExceptions and will not be available to other systems.
-   **State Mutation:** Do not attempt to modify the HashSet returned by `getBlockTypeKeys`. While the collection itself is mutable, the asset is treated as immutable by the engine. Modifying it can lead to a desynchronization between the raw keys and the computed BlockPattern, causing undefined and unpredictable behavior.

## Data Pipeline
The flow for this asset is a one-way transformation from a declarative file on disk to a usable, in-memory data structure.

> Flow:
> JSON Asset File -> Asset Loading System -> **BlockTypeListAsset.CODEC** -> **BlockTypeListAsset Instance** (with computed BlockPattern) -> Game Systems (e.g., Builder Tool)

