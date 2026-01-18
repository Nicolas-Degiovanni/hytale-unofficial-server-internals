---
description: Architectural reference for AssetBuilderCodec
---

# AssetBuilderCodec

**Package:** com.hypixel.hytale.assetstore.codec
**Type:** Utility

## Definition
```java
// Signature
public class AssetBuilderCodec<K, T extends JsonAsset<K>> extends BuilderCodec<T> implements AssetCodec<K, T> {
```

## Architecture & Concepts

The AssetBuilderCodec is a specialized component within the engine's serialization framework, designed exclusively for handling Hytale's `JsonAsset` types. It serves as the primary mechanism for deserializing JSON data from asset files into fully realized Java objects.

Its core architectural purpose is to bridge the generic `BuilderCodec` system with the specific requirements of the asset pipeline. The most critical feature it provides is the robust implementation of **asset inheritance**. Game assets can specify a `Parent` asset, from which they inherit properties. This codec manages the complex logic of loading the parent, copying its values to the child, and then selectively overwriting those values with properties defined in the child's JSON file.

This class is not a service but a configurable utility. Instances are created using a Builder pattern, tailored for a specific asset type (e.g., a `ModelCodec`, a `SoundCodec`). This configuration includes providing lambdas for setting and getting asset metadata, which decouples the codec from the concrete implementation of the asset classes it manages.

Furthermore, the AssetBuilderCodec is responsible for generating a machine-readable `ObjectSchema`. This schema is used by external tooling, such as in-game editors or documentation generators, to understand the structure and inheritance rules of game assets.

## Lifecycle & Ownership

-   **Creation:** Instances are not created directly. They are constructed via the static `builder` or `wrap` factory methods, typically during the engine's bootstrap phase. A single, configured instance is created for each distinct asset type that needs to be loaded. These instances are then registered with a central codec or asset management system.
-   **Scope:** An AssetBuilderCodec instance is a long-lived, stateless object. Once configured and registered, it persists for the entire application session. It is designed to be reused for decoding any number of assets of its configured type.
-   **Destruction:** The object is garbage collected during application shutdown when the registries holding references to it are cleared. It has no explicit `destroy` or `close` method.

## Internal State & Concurrency

-   **State:** The internal state of an AssetBuilderCodec is **immutable**. All configuration, including field codecs and property accessors, is provided during construction via the builder and stored in `final` fields. The codec itself does not store or cache any per-asset data.

-   **Thread Safety:** This class is inherently **thread-safe**. Its immutable nature ensures that a single instance can be safely invoked by multiple worker threads simultaneously without risk of data corruption or race conditions. This is a critical design feature that enables high-performance, parallel asset loading from disk.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| builder(...) | static Builder | O(1) | Creates a new builder to configure and construct a codec instance. |
| wrap(...) | static AssetBuilderCodec | O(1) | Wraps an existing BuilderCodec, adding asset-specific functionality. |
| decodeJsonAsset(...) | T | O(N) | Decodes a JSON stream into an asset object. N is the size of the JSON input. |
| decodeAndInheritJsonAsset(...) | T | O(N) | The core decoding logic. Handles loading, parent inheritance, and final deserialization. |
| toSchema(...) | ObjectSchema | O(F) | Generates a schema definition for the asset type. F is the number of fields. |

## Integration Patterns

### Standard Usage

The primary pattern is to define a codec for a specific asset type at application startup and register it with the appropriate system. The engine's AssetStore will then invoke it automatically. Direct invocation is rare.

```java
// Example: Creating a codec for a hypothetical "Prop" asset
// This code would run once during engine initialization.

public static final Codec<Prop> PROP_CODEC = AssetBuilderCodec.builder(
        Prop.class,
        Prop::new,
        Codec.STRING, // The type of the asset's ID
        Prop::setId,
        Prop::getId,
        Prop::setAssetData,
        Prop::getAssetData
    )
    .append("model", Codec.STRING, Prop::setModel, Prop::getModel)
    .append("scale", Codec.FLOAT, Prop::setScale, Prop::getScale)
    .build();

// The PROP_CODEC is then registered with a central registry.
// The AssetStore will later use it to load all ".prop.json" files.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** The constructor is `protected`. Do not attempt to call `new AssetBuilderCodec()`. Always use the static `builder` or `wrap` methods to ensure correct initialization.
-   **Stateful Lambdas:** The getter and setter lambdas provided to the builder must not rely on or modify external state. They should operate solely on the asset instance passed to them to maintain thread safety.
-   **Manual Inheritance:** Do not manually read a parent JSON file and copy its values before calling the codec. The `decodeAndInheritJsonAsset` method is optimized to handle the entire inheritance chain correctly and efficiently. Attempting to do this yourself will lead to incorrect or incomplete asset data.

## Data Pipeline

The AssetBuilderCodec is a key processing stage in the asset loading pipeline. It transforms raw text data into structured, in-memory game objects.

> Flow:
> JSON File on Disk -> AssetStore Request -> RawJsonReader Stream -> **AssetBuilderCodec**.decodeAndInheritJsonAsset -> Fully Hydrated Java Asset Object -> Asset Cache / Game Systems

