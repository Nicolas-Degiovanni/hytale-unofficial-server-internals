---
description: Architectural reference for BlockType
---

# BlockType

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Managed Asset

## Definition
```java
// Signature
public class BlockType implements JsonAssetWithMap<String, BlockTypeAssetMap<String, BlockType>>, NetworkSerializable<com.hypixel.hytale.protocol.BlockType> {
```

## Architecture & Concepts

The BlockType class is the definitive data representation for every block in the Hytale world. It is not an active entity but rather a passive, data-centric asset that defines the behavior, appearance, and properties of a block. It serves as the single source of truth for all game systems that interact with blocks, including the rendering engine, physics simulation, world generation, and interaction logic.

Architecturally, BlockType objects are the deserialized result of JSON asset files. The static field **CODEC**, an instance of AssetBuilderCodec, is the cornerstone of this class. It declaratively defines the entire schema for a block's JSON configuration, including property names, data types, validation rules, and inheritance logic. This codec-driven approach centralizes the asset loading and validation process, ensuring that all BlockType instances are constructed consistently and correctly from their source files.

BlockType instances are managed exclusively by the global AssetStore. They are not intended for direct instantiation by developers. Once loaded, they are registered in the BlockTypeAssetMap, which provides fast, indexed lookups by both string key (e.g., "hytale:stone") and integer ID. This registry is fundamental to the performance of chunk serialization and world state management.

A critical function of this class is its role in network serialization. The toPacket method translates the comprehensive server-side BlockType definition into a more compact, index-based protocol message for transmission to the client. This process involves converting asset key strings into integer IDs and pre-calculating values to optimize client-side processing.

## Lifecycle & Ownership

-   **Creation:** BlockType instances are created exclusively by the Hytale Asset System during server or client startup. The static CODEC field is used by the AssetStore to decode JSON definitions into Java objects. A BlockType is typically defined within an Item asset file, establishing a direct link between the placeable block and its inventory representation.
-   **Scope:** An instance of BlockType, once loaded, persists for the entire application session. It is stored in a static AssetStore and is considered a global, immutable definition.
-   **Destruction:** Instances are garbage collected only when the AssetRegistry is cleared, which typically occurs during application shutdown.

## Internal State & Concurrency

-   **State:** A BlockType object is mutable during its initial creation by the AssetBuilderCodec. After the `processConfig` method is called post-deserialization, the object should be treated as **effectively immutable**. Many fields are transient and cache calculated data, such as integer indices for related assets (e.g., `blockSoundSetIndex`) or rotated versions of support rules (`rotatedSupport`).
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be fully loaded and processed on a single thread during the application's bootstrap phase. After this initialization, it is safe for multiple threads to read its properties.

    **WARNING:** Concurrent access to methods that perform lazy initialization, such as `getSupport` or `getRailConfig`, on the same instance can lead to race conditions if not properly synchronized externally. However, in standard engine operation, these objects are fully initialized before being accessed by game threads.

## API Surface

The public API is primarily composed of getters. The following methods represent the most significant parts of its public contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | **STATIC.** Retrieves the central AssetStore responsible for managing all BlockType assets. |
| getAssetMap() | BlockTypeAssetMap | O(1) | **STATIC.** Retrieves the map for direct lookup of BlockType assets by key or integer ID. |
| toPacket() | com.hypixel.hytale.protocol.BlockType | O(N) | Serializes the object into a network-optimized packet. Caches the result in a SoftReference. |
| processConfig() | void | O(1) | Internal post-deserialization hook. Finalizes state, resolves defaults, and calculates derived data. |
| getBlockForState(state) | BlockType | O(1) | Retrieves the BlockType associated with a given state string (e.g., "powered"). Returns null if no such state exists. |
| getSupport(rotationIndex) | Map | O(N) | Calculates and caches the block's support requirements for a given rotation. Critical for physics simulation. |
| getRailConfig(rotationIndex) | RailConfig | O(N) | Calculates and caches the block's rail configuration for a given rotation. |

## Integration Patterns

### Standard Usage

A developer should never instantiate a BlockType directly. It must always be retrieved from the central asset map, which is populated at startup.

```java
// Correctly retrieve a registered BlockType instance
BlockTypeAssetMap map = BlockType.getAssetMap();
BlockType stone = map.getAsset("hytale:stone");

if (stone != null) {
    // Read properties for game logic
    BlockMaterial material = stone.getMaterial();
    // ...
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new BlockType()`. This bypasses the asset loading system, resulting in an uninitialized and invalid object that is not registered with any game systems.
-   **Runtime Modification:** Modifying the state of a BlockType instance after it has been loaded is strictly forbidden. This will cause unpredictable behavior and client-server desynchronization, as clients will retain the original, unmodified version of the asset.
-   **Assuming Existence:** Do not assume an asset exists. Always check for null after retrieving a BlockType from the asset map, as a typo or missing asset will result in a null return.

## Data Pipeline

The BlockType class is a critical stage in the asset data pipeline, transforming declarative configuration files into an optimized, in-memory representation used by the game engine.

> Flow:
> JSON Asset File -> AssetStore Loader -> **BlockType.CODEC** (Deserialization) -> **BlockType Instance** -> `processConfig()` (Finalization) -> BlockTypeAssetMap (Registration) -> Game Systems (Physics, Rendering) -> `toPacket()` -> Network Buffer -> Client Engine

