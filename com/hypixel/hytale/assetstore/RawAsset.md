---
description: Architectural reference for RawAsset
---

# RawAsset

**Package:** com.hypixel.hytale.assetstore
**Type:** Transient

## Definition
```java
// Signature
public class RawAsset<K> implements AssetHolder<K> {
```

## Architecture & Concepts
The RawAsset class is a foundational component of the asset loading pipeline. It does **not** represent a fully deserialized, usable game asset like a model or texture. Instead, it serves as an intermediate, immutable data structure that represents a *discovered* asset whose raw data has been located but not yet parsed.

It acts as a pointer to the asset's source data, which can exist in one of two forms:
1.  **Standalone Asset:** A single asset contained within its own file on the filesystem. In this state, the RawAsset holds a Path to the file.
2.  **Contained Asset:** An asset defined as part of a larger container file (e.g., a single JSON file defining multiple blocks). In this state, the RawAsset holds a reference to the parent file's path and an in-memory character buffer corresponding to the specific JSON object for that asset.

This dual-mode design allows the asset system to abstract away the physical storage layout of assets, providing a uniform interface to the subsequent parsing and deserialization stages. The generic type parameter K represents the asset's key or identifier.

## Lifecycle & Ownership
- **Creation:** RawAsset instances are created exclusively by the asset discovery and scanning systems. When the engine scans the filesystem for assets, it instantiates RawAsset objects for each asset it finds, whether standalone or contained within a larger file.
- **Scope:** This object is short-lived and transient. Its scope is confined to the duration of the asset loading and processing pipeline. It is created, passed to a codec for parsing, and then is no longer needed.
- **Destruction:** A RawAsset becomes eligible for garbage collection as soon as the consuming AssetCodec has finished using it to produce a fully deserialized game object. There is no explicit destruction method. Holding long-term references to RawAsset objects is an anti-pattern, as they may retain large character buffers in memory.

## Internal State & Concurrency
- **State:** The RawAsset is **immutable**. All of its internal fields are final, and methods that appear to modify state, such as withResolveKeys, return a new instance of RawAsset with the updated values. This design simplifies its use in a multi-threaded asset loading environment.

- **Thread Safety:** This class is inherently **thread-safe** for read operations due to its immutability. It can be safely passed between threads without synchronization.

    **WARNING:** The getBuffer method returns a direct reference to the internal character array. While the RawAsset class itself does not modify this array, consumers of the API must treat the returned array as read-only. Modifying the contents of the buffer will lead to unpredictable behavior and data corruption during parsing.

## API Surface
The public API is designed to provide access to the asset's source data and metadata required for parsing.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toRawJsonReader(Supplier) | RawJsonReader | O(N) / O(1) | **Primary consumption method.** Creates a JSON reader for the asset's data. Complexity is O(N) for file-based assets (N = file size) and O(1) for buffer-based assets. Throws IOException on file read errors. |
| makeData(Class, K, K) | AssetExtraInfo.Data | O(1) | Constructs a metadata object used by the codec during final deserialization. This object carries information about inheritance and keys. |
| withResolveKeys(K, K) | RawAsset | O(1) | Creates a new RawAsset instance with updated key and parentKey information. This is used in a later stage of the pipeline after parent-child relationships have been resolved. |

## Integration Patterns

### Standard Usage
The primary consumer of a RawAsset is an AssetCodec. The codec uses the RawAsset to get a reader for the raw data, which it then uses to deserialize the final game object.

```java
// Hypothetical usage within an AssetCodec
public Asset process(RawAsset<AssetKey> rawAsset) throws IOException {
    // Obtain a reader for the raw JSON data
    Supplier<char[]> bufferSupplier = this.bufferPool::getBuffer;
    RawJsonReader reader = rawAsset.toRawJsonReader(bufferSupplier);

    // Deserialize the JSON into a final game object
    MyGameObject finalAsset = gson.fromJson(reader, MyGameObject.class);

    // The rawAsset is no longer needed and can be garbage collected
    return finalAsset;
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new RawAsset()`. These objects are intended to be created only by the core asset scanning system. Manual creation bypasses the discovery pipeline and can lead to inconsistent or incomplete asset data.
- **Long-Term Storage:** Do not cache or store RawAsset instances in long-lived services or managers. They are transient objects that may hold large memory buffers. They should be processed and then discarded to allow the garbage collector to reclaim memory.
- **Buffer Modification:** Never modify the character array returned by `getBuffer()`. This violates the immutability contract and will corrupt the asset data for all consumers.

## Data Pipeline
The RawAsset serves as a critical link between the file discovery stage and the data parsing stage of the asset loading system.

> Flow:
> Filesystem Scan -> **RawAsset** (Pointer to raw file or buffer) -> AssetCodec -> Deserialized Game Asset -> AssetManager Cache

