---
description: Architectural reference for DecodedAsset
---

# DecodedAsset

**Package:** com.hypixel.hytale.assetstore
**Type:** Transient

## Definition
```java
// Signature
public class DecodedAsset<K, T extends JsonAsset<K>> implements AssetHolder<K> {
```

## Architecture & Concepts
The DecodedAsset class is an immutable data carrier that serves as a fundamental component within the asset loading pipeline. Its primary role is to encapsulate an asset that has been successfully deserialized from a persistent format (such as JSON) into a memory-resident Java object.

It acts as a standardized container, pairing the deserialized asset object (T) with its unique identifier or key (K). This structure ensures that once an asset is loaded, its data and its identity are tightly coupled, preventing state corruption as it is passed between different engine subsystems. By conforming to the AssetHolder interface, it provides a consistent contract for any system that needs to access an asset's key.

This class is not a service or manager; it is a simple value object. Its existence signifies the successful completion of the decoding stage in the broader asset-sourcing data flow.

## Lifecycle & Ownership
- **Creation:** A DecodedAsset is instantiated exclusively by asset decoders or loaders. For example, upon reading a JSON file from disk, a deserialization service will construct the corresponding JsonAsset object and wrap it, along with its key, inside a new DecodedAsset instance.
- **Scope:** The object is short-lived and its scope is typically confined to a single asset loading transaction. It is created, passed to a consumer such as an AssetCache or AssetManager, and is then intended to be discarded.
- **Destruction:** Managed entirely by the Java Garbage Collector. As a pure data holder with no external resource handles, it requires no explicit destruction or cleanup logic.

## Internal State & Concurrency
- **State:** Immutable. Both the key and asset fields are declared as final and are assigned only once during construction. This design guarantees that a DecodedAsset instance is a stable, non-modifiable snapshot of a loaded resource.
- **Thread Safety:** This class is inherently thread-safe. Its immutability allows instances to be safely passed between and read by multiple threads without locks or other synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getKey() | K | O(1) | Returns the unique key identifying the asset. |
| getAsset() | T | O(1) | Returns the deserialized, memory-resident asset object. |

## Integration Patterns

### Standard Usage
The DecodedAsset is typically the return type of a loading or decoding operation. The consumer then unwraps the object to populate a cache or pass the asset to a game system.

```java
// A hypothetical AssetLoader returns a DecodedAsset
DecodedAsset<ModelKey, ModelAsset> loadedModel = assetLoader.decode("models/player.json");

// The consumer extracts the key and asset for further processing
ModelKey key = loadedModel.getKey();
ModelAsset asset = loadedModel.getAsset();
modelCache.put(key, asset);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not maintain large collections of DecodedAsset objects for long-term caching. This class is a transient wrapper. For caching, it is more memory-efficient to extract the key and asset into a dedicated data structure like a Map.
- **Null Instantiation:** While the constructor may permit it, creating a DecodedAsset with a null key or a null asset violates the implicit contract of the asset system and will almost certainly lead to NullPointerExceptions in downstream consumers.

## Data Pipeline
The DecodedAsset represents a specific stage in the transformation of data from disk to a usable in-engine resource.

> Flow:
> Raw Asset File (e.g., JSON) -> Deserialization Service -> **DecodedAsset** -> Asset Cache -> Game System (e.g., Renderer)

