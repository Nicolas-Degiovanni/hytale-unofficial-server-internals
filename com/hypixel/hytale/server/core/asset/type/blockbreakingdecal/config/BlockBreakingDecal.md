---
description: Architectural reference for BlockBreakingDecal
---

# BlockBreakingDecal

**Package:** com.hypixel.hytale.server.core.asset.type.blockbreakingdecal.config
**Type:** Asset Data Model

## Definition
```java
// Signature
public class BlockBreakingDecal
   implements JsonAssetWithMap<String, DefaultAssetMap<String, BlockBreakingDecal>>,
   NetworkSerializable<com.hypixel.hytale.protocol.BlockBreakingDecal> {
```

## Architecture & Concepts
The BlockBreakingDecal class is a data model representing a server-defined asset. It is not a service or a manager, but rather a passive, data-centric object that holds the configuration for the visual effect displayed when a player breaks a block. Each instance corresponds to a specific JSON asset file on disk.

Its primary architectural role is to serve as the in-memory representation of this configuration, loaded and managed by the Hytale Asset Pipeline. The class design is centered around two key static members:

1.  **CODEC:** An `AssetBuilderCodec` that defines the contract for deserialization. It maps keys from a source JSON file (e.g., "StageTextures") to the fields of a BlockBreakingDecal instance. This is the factory and hydration mechanism for the class.
2.  **ASSET_STORE:** A static, lazily-initialized reference to the global registry for all BlockBreakingDecal assets. This singleton store ensures that each decal configuration is loaded only once and is accessible globally via its unique asset key.

The implementation of the NetworkSerializable interface signifies that this configuration is not purely server-side. It is designed to be serialized into a network packet and transmitted to clients, ensuring visual consistency between the server's state and the client's renderer.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale asset loading system during server bootstrap. The static `CODEC` is invoked by the `AssetStore` to instantiate and populate objects from corresponding JSON files. Manual instantiation is a critical anti-pattern.
-   **Scope:** Once an asset is loaded, its corresponding BlockBreakingDecal instance persists for the entire lifetime of the server process. It is a global, shared, and effectively immutable configuration object.
-   **Destruction:** Instances are eligible for garbage collection only during server shutdown when the central `AssetRegistry` is cleared.

## Internal State & Concurrency
-   **State:** The state of a BlockBreakingDecal instance is **immutable** after it has been loaded and hydrated by the asset pipeline. Its fields, such as `stageTextures`, are populated once during creation and are not designed to be modified at runtime.

-   **Thread Safety:** The object is **thread-safe for read operations**. Due to its immutable nature, multiple threads can safely call methods like `toPacket` or `getId` without synchronization.

    **Warning:** The lazy initialization within the static `getAssetStore` method is not protected by a lock. While asset loading is typically a single-threaded process during server startup, invoking this method from multiple threads concurrently before the store is initialized could lead to a race condition. This pattern relies on the engine's bootstrap process to be single-threaded.

## API Surface
The public API is minimal, focusing on data access and serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the singleton global registry for all BlockBreakingDecal assets. |
| toPacket() | com.hypixel.hytale.protocol.BlockBreakingDecal | O(N) | Serializes the object into a network-ready packet. N is the number of stage textures. |
| getId() | String | O(1) | Returns the unique asset key for this decal configuration. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve a fully-hydrated instance from the global asset store using its known asset key. Never create an instance directly.

```java
// Obtain the global registry for this asset type.
AssetStore<String, BlockBreakingDecal, ?> store = BlockBreakingDecal.getAssetStore();

// Fetch a specific, pre-loaded decal configuration by its ID.
// This is a fast map lookup.
BlockBreakingDecal decal = store.get("hytale:stone_breaking_decal");

// Convert to a network packet to send to a client for rendering.
if (decal != null) {
    com.hypixel.hytale.protocol.BlockBreakingDecal packet = decal.toPacket();
    // networkManager.sendToClient(player, packet);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BlockBreakingDecal()`. The constructor is protected for a reason. Manually created instances will be uninitialized, lack an ID, and will not be registered in the global asset store, leading to system desynchronization.
-   **Runtime State Modification:** Do not attempt to modify the internal `stageTextures` array after the object has been loaded. This violates the immutability contract and can cause unpredictable visual behavior for clients.
-   **Assuming Existence:** Do not assume an asset exists. Always perform a null check after retrieving an asset from the `AssetStore`, as the requested key may not correspond to a loaded asset.

## Data Pipeline
The BlockBreakingDecal acts as a conduit for configuration data, transforming it from a static file into a transmissible network object.

> Flow:
> JSON Asset File -> Server Asset Pipeline -> **BlockBreakingDecal.CODEC** -> In-Memory Instance in **BlockBreakingDecal.ASSET_STORE** -> Game Logic requests instance -> `toPacket()` serialization -> Network Layer -> Client Renderer

