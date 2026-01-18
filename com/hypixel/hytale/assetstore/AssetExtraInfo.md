---
description: Architectural reference for AssetExtraInfo
---

# AssetExtraInfo

**Package:** com.hypixel.hytale.assetstore
**Type:** Transient

## Definition
```java
// Signature
public class AssetExtraInfo<K> extends ExtraInfo {
```

## Architecture & Concepts
The AssetExtraInfo class is a transient metadata container used exclusively during the asset loading and registration pipeline. It serves as a temporary carrier for all metadata extracted from an asset definition file (e.g., a JSON file) before the final asset is instantiated and stored in the AssetRegistry.

Its primary architectural role is to decouple the asset decoding process from the asset registration and validation process. A decoder can parse a file and populate an AssetExtraInfo object without needing immediate access to the global AssetRegistry.

Key architectural concepts embodied by this class include:

*   **Composite Assets:** The nested Data class supports the definition of assets contained within other assets. A single source file can define a primary asset (the container) and multiple sub-assets (the contained). AssetExtraInfo manages this hierarchy and triggers the subsequent loading of these sub-assets.
*   **High-Performance Tagging:** It processes and stores a rich set of tags associated with an asset. These tags are converted from strings into integer indexes for highly efficient storage and lookup within the AssetRegistry. This system is critical for game logic that needs to query for assets with specific properties (e.g., "all items with tag *rare* and *weapon*").
*   **Validation Context:** By extending ExtraInfo, it integrates directly into the engine's data validation framework. Any warnings or errors discovered during parsing or processing are collected within this object, providing a complete validation report for each asset.

This class is an internal component of the asset store system and is not intended for direct use by game feature developers.

## Lifecycle & Ownership
- **Creation:** An AssetExtraInfo instance is created by an asset decoder or loader at the beginning of an asset file processing operation. Its constructor is called with the file path and a populated Data object containing the asset's core identity.
- **Scope:** The object's lifetime is strictly limited to the duration of a single asset loading transaction. It exists only while the asset is being parsed, validated, and registered.
- **Destruction:** It is a short-lived object. Once the AssetRegistry has consumed its metadata to create and index the final asset, the AssetExtraInfo object is no longer referenced and becomes eligible for garbage collection. There is no explicit cleanup method.

## Internal State & Concurrency
- **State:** The internal state, particularly within the nested Data object, is highly **mutable**. It is designed to be built progressively as an asset file is parsed. Methods like putTags and addContainedAsset modify its internal collections.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without external synchronization. It is designed for single-threaded access within the context of a dedicated asset loading task. The asset pipeline is responsible for ensuring that each instance is confined to a single worker thread.

## API Surface
This section details the public contract of AssetExtraInfo and its critical nested Data class.

### AssetExtraInfo API

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateKey() | String | O(1) | Creates a unique string identifier for debugging or special-case lookups. |
| getKey() | K | O(1) | Returns the primary key of the asset being processed. |
| getAssetPath() | Path | O(1) | Returns the filesystem path of the source asset file, if available. |
| getData() | AssetExtraInfo.Data | O(1) | Provides access to the core metadata container. |
| getValidationResults() | AssetValidationResults | O(1) | Retrieves the collection of validation errors and warnings for this asset. |

### AssetExtraInfo.Data API
The Data class contains the most significant logic for handling asset metadata.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRootContainerData() | AssetExtraInfo.Data | O(N) | Traverses the parent hierarchy to find the top-level asset data. N is the nesting depth. |
| putTags(tags) | void | O(M*V) | Ingests a map of tags. M is the number of tags, V is the average number of values per tag. This is a state-mutating operation. |
| addContainedAsset(class, asset) | void | O(1) | Registers a fully-formed sub-asset that was defined within this asset's file. |
| loadContainedAssets(reloading) | void | O(C) | **CRITICAL:** Triggers a recursive call into the AssetRegistry to load all registered contained assets. C is the number of contained asset classes. |
| getRawTags() | Map | O(1) | Returns an unmodifiable view of the raw string-based tags. |
| getExpandedTagIndexes() | IntSet | O(1) | Returns an unmodifiable set of all integer-indexed tags for this asset. |

## Integration Patterns

### Standard Usage
This class is used internally by asset loaders. The following is a conceptual example of its lifecycle within a hypothetical decoder.

```java
// Conceptual code within an AssetDecoder
public void processAssetFile(Path assetFile) {
    // 1. Parse the raw file (e.g., JSON)
    ParsedData parsed = parseJson(assetFile);

    // 2. Create the core Data object
    AssetExtraInfo.Data data = new AssetExtraInfo.Data(
        parsed.getAssetClass(),
        parsed.getKey(),
        parsed.getParentKey()
    );
    data.putTags(parsed.getTags());
    for (ContainedAsset subAsset : parsed.getContainedAssets()) {
        data.addContainedAsset(subAsset.getClass(), subAsset);
    }

    // 3. Create the main info object
    AssetExtraInfo info = new AssetExtraInfo(assetFile, data);

    // 4. Pass the metadata to the central registry for processing
    AssetRegistry.register(info);

    // The 'info' object is now out of scope and will be garbage collected.
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not hold references to AssetExtraInfo objects after the asset loading process is complete. They are transient and their data is not guaranteed to be relevant after registration.
- **External Modification:** Never modify an AssetExtraInfo object after it has been submitted to the AssetRegistry. This can lead to a corrupted registry state.
- **Game Logic Dependency:** Game logic should never interact with this class. Query the AssetRegistry using tags or keys to retrieve final, immutable asset objects.

## Data Pipeline
AssetExtraInfo acts as a message bus object carrying data through the stages of asset ingestion.

> Flow:
> Raw Asset File (JSON) -> Asset Decoder -> **AssetExtraInfo (Instance Created & Populated)** -> AssetRegistry.register() -> Asset Instantiation & Indexing -> **AssetExtraInfo (Instance Discarded)**

