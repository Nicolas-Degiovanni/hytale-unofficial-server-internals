---
description: Architectural reference for AssetCodec
---

# AssetCodec

**Package:** com.hypixel.hytale.assetstore.codec
**Type:** Core Interface / Contract

## Definition
```java
// Signature
public interface AssetCodec<K, T extends JsonAsset<K>> extends InheritCodec<T>, ValidatableCodec<T> {
```

## Architecture & Concepts
The AssetCodec interface is a foundational contract within Hytale's asset pipeline. It defines the standardized mechanism for deserializing raw JSON data from disk into a fully-realized, in-memory JsonAsset object. This is not a simple data-to-object mapper; it is a sophisticated component responsible for three critical engine functions:

1.  **Decoding:** Translating raw JSON streams into structured Java objects.
2.  **Inheritance Resolution:** Applying Hytale's powerful asset inheritance system. An asset can be defined as a delta or a modification of a parent asset, and the codec is responsible for correctly merging the parent's data with the child's overrides. This is signified by its extension of the InheritCodec interface.
3.  **Validation:** Ensuring the decoded asset conforms to required schemas and engine constraints, as mandated by the ValidatableCodec interface.

An AssetCodec implementation acts as the authoritative "translator" for a specific type of asset (e.g., a Model, a Texture, a Sound configuration). The asset loading system maintains a registry of these codecs, dispatching decoding tasks to the appropriate implementation based on the asset type being loaded.

## Lifecycle & Ownership
As an interface, AssetCodec itself does not have a lifecycle. However, its concrete implementations follow a strict, managed lifecycle.

-   **Creation:** Implementations are instantiated once by the core asset system during engine bootstrap. They are discovered and registered in a central codec registry.
-   **Scope:** A registered codec implementation is a de-facto singleton that persists for the entire application lifetime. Its purpose is to provide stateless, reusable decoding logic.
-   **Destruction:** Codecs are not destroyed until the application shuts down and the entire asset system is torn down.

**WARNING:** Implementations of this interface are service-level objects. They must never be instantiated directly by game logic or other engine systems. Always rely on the central asset management system to provide and invoke the correct codec.

## Internal State & Concurrency
-   **State:** Implementations of AssetCodec **must be stateless**. The methods are designed to be pure functions where the output (the decoded asset) depends only on the inputs (the raw data and metadata). They must not cache results or maintain any state between invocations.
-   **Thread Safety:** Implementations **must be thread-safe**. The asset loading system is highly parallelized and will invoke codec methods from multiple worker threads simultaneously. The stateless design is the primary enabler of this concurrency. All necessary context is passed via method arguments, eliminating the need for internal locks or synchronization.

## API Surface
The public contract of AssetCodec is focused entirely on the decoding and metadata extraction process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getKeyCodec() | KeyedCodec<K> | O(1) | Returns the codec responsible for handling the asset's unique key or identifier. |
| getParentCodec() | KeyedCodec<K> | O(1) | Returns the codec for resolving the key of a parent asset, crucial for the inheritance system. |
| getData(T asset) | AssetExtraInfo.Data | O(1) | Extracts supplementary metadata from an already-decoded asset instance. |
| decodeJsonAsset(RawJsonReader, AssetExtraInfo) | T | O(N) | Decodes a base asset from a raw JSON stream. This is used for assets that have no parent. N is the size of the input stream. |
| decodeAndInheritJsonAsset(RawJsonReader, T, AssetExtraInfo) | T | O(N) | Decodes a child asset and merges its data with the provided parent asset instance. This is the core of the inheritance mechanism. |

## Integration Patterns

### Standard Usage
Direct interaction with an AssetCodec is rare and typically handled by the engine's AssetManager. The conceptual flow involves the manager selecting the correct codec and invoking it.

```java
// Conceptual example within the AssetManager
// Note: This is a simplified representation.

// 1. Locate the appropriate codec for the asset type
AssetCodec<ModelKey, Model> modelCodec = codecRegistry.get(Model.class);
RawJsonReader reader = getReaderForAsset(assetPath);
AssetExtraInfo<ModelKey> extraInfo = getMetadataForAsset(assetPath);

// 2. If the asset has a parent, load it first
Model parentModel = null;
if (extraInfo.hasParent()) {
    parentModel = assetManager.load(extraInfo.getParentKey());
}

// 3. Decode the asset, applying inheritance if necessary
Model finalModel;
if (parentModel != null) {
    finalModel = modelCodec.decodeAndInheritJsonAsset(reader, parentModel, extraInfo);
} else {
    finalModel = modelCodec.decodeJsonAsset(reader, extraInfo);
}
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Creating a codec that stores a reference to the last asset it decoded or any other mutable state. This will break in a multi-threaded environment and lead to catastrophic data corruption.
-   **Bypassing Inheritance:** Calling decodeJsonAsset on an asset that is known to have a parent. This will result in an incomplete object that is missing all inherited properties.
-   **Manual Invocation:** Attempting to read a file and call a codec directly from game code. This bypasses the entire asset management system, including caching, dependency tracking, and metadata handling.

## Data Pipeline
The AssetCodec is a critical transformation step in the data pipeline that converts raw asset definitions into usable engine objects.

> Flow:
> Asset File on Disk (.json) -> RawJsonReader (Byte Stream) -> **AssetCodec** (Deserialization & Inheritance) -> JsonAsset (In-Memory Object) -> Game Systems (Renderer, Physics, etc.)

