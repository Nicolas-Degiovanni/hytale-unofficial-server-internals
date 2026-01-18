---
description: Architectural reference for AssetLoadResult
---

# AssetLoadResult<K, T>

**Package:** com.hypixel.hytale.assetstore
**Type:** Transient Data Container

## Definition
```java
// Signature
public class AssetLoadResult<K, T> {
```

## Architecture & Concepts
The AssetLoadResult class is a passive, immutable data container that encapsulates the complete outcome of an asset loading operation. It serves as the formal data transfer object between the low-level AssetLoader subsystem and any part of the engine that requests assets.

Its primary architectural purpose is to provide a comprehensive and atomic report on a potentially complex loading task. Modern game assets are often composite; a single logical asset, like a character model, may depend on numerous child assets such as textures, materials, and sound files. The recursive structure of AssetLoadResult, via the *childAssetResults* field, allows the system to represent the success or failure of an entire dependency tree within a single, unified object.

This class is not an active participant in the loading process itself. Instead, it is the final artifact produced by that process, containing a clear separation of successfully loaded assets from a detailed list of failures. This design enforces a robust error-handling pattern, compelling the consuming code to explicitly check the status of the operation before attempting to use the returned assets.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the core asset loading service at the conclusion of a load request. Game logic or client code must never create an instance of this class directly.
- **Scope:** The object's lifetime is ephemeral. It is designed to be short-lived, existing only to bridge the gap between the asynchronous loading operation and the synchronous code that consumes its result.
- **Destruction:** As a simple data object with no external resource handles, it is managed entirely by the Java garbage collector. It becomes eligible for collection as soon as the calling method finishes processing its contents, such as populating a cache or logging errors.

## Internal State & Concurrency
- **State:** AssetLoadResult is designed to be an immutable snapshot. After construction, its internal state cannot be altered. While the collections it holds are technically mutable, the class exposes no methods for their modification, and they should be treated as read-only by all consumers.

- **Thread Safety:** The object is inherently thread-safe for read operations. It is constructed and finalized within a single thread (typically an asset worker thread) before being passed to other systems. Any thread can safely inspect its contents without synchronization, as its state is guaranteed not to change post-construction.

## API Surface
The public API is minimal, consisting of accessors for the result data and a key method to query the overall success of the operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLoadedAssets() | Map<K, T> | O(1) | Returns a map of successfully loaded assets. **Warning:** Do not call this without first checking hasFailed(). |
| getLoadedKeyToPathMap() | Map<K, Path> | O(1) | Returns a map of asset keys to their source file paths for successfully loaded assets. |
| getFailedToLoadKeys() | Set<K> | O(1) | Returns the set of asset keys that could not be loaded. |
| getFailedToLoadPaths() | Set<Path> | O(1) | Returns the set of file paths that could not be loaded. |
| hasFailed() | boolean | O(N) | Recursively checks for any failures in this result or any of its child results. N is the total number of child results. This is the primary method for determining the operation's success. |

## Integration Patterns

### Standard Usage
The correct pattern involves initiating a load, receiving the AssetLoadResult, and immediately checking its failure state before proceeding.

```java
// AssetManager is a hypothetical engine service
AssetLoadResult<String, Texture> result = assetManager.loadAllTextures("assets/ui/main_menu/");

if (result.hasFailed()) {
    // Critical: Handle the failure case. Log specific errors,
    // display placeholder assets, or halt engine initialization.
    for (Path failedPath : result.getFailedToLoadPaths()) {
        LOGGER.error("Failed to load required asset: " + failedPath);
    }
    // Potentially throw an exception to stop a critical loading sequence.
    throw new AssetLoadingException("Could not load main menu UI.");
} else {
    // Only on success, proceed to use the assets.
    Map<String, Texture> loadedTextures = result.getLoadedAssets();
    uiManager.registerTextures(loadedTextures);
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Failures:** Accessing getLoadedAssets without first calling hasFailed is a severe anti-pattern. If the operation partially failed, the returned map will be incomplete, and subsequent code will likely encounter NullPointerExceptions or render an inconsistent game state.
- **Direct Instantiation:** Never use `new AssetLoadResult(...)`. This object's integrity depends on it being a truthful report from the asset system. Constructing it manually circumvents the entire asset loading pipeline and is not supported.
- **Modifying Returned Collections:** The collections returned by the getters should be treated as immutable. Attempting to add or remove elements from them will have no effect on the underlying asset caches and violates the object's contract as a read-only report.

## Data Pipeline
AssetLoadResult is the terminal point of the asset loading data flow. It converts the complex, stateful process of I/O and deserialization into a simple, final data structure.

> Flow:
> Asset Load Request -> AssetLoader Service -> Filesystem I/O -> Deserializer -> **AssetLoadResult** -> Requesting System (e.g., Scene Loader, UI Manager)

