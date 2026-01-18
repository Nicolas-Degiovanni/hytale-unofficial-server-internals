---
description: Architectural reference for ItemReticleConfig
---

# ItemReticleConfig

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Asset / Data Transfer Object

## Definition
```java
// Signature
public class ItemReticleConfig
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, ItemReticleConfig>>,
   NetworkSerializable<com.hypixel.hytale.protocol.ItemReticleConfig> {
```

## Architecture & Concepts

The ItemReticleConfig class is a data model that defines the appearance and behavior of the player's targeting reticle for a specific item. It is a fundamental component of the Hytale **Asset System**, designed to be loaded from JSON configuration files rather than being instantiated directly in code.

This class serves as a critical bridge between static game data and the runtime networking layer. Its primary responsibilities are:

1.  **Data Deserialization:** Through its static CODEC field, it defines a strict schema for parsing reticle JSON files. This process includes validation, inheritance from base configurations, and post-processing logic.
2.  **Configuration Holding:** An instance of this class holds the complete, processed configuration for a single reticle type, including its base appearance and special appearances triggered by client-side or server-side game events.
3.  **Network Serialization:** It implements the NetworkSerializable interface, allowing its state to be efficiently converted into a network packet (`com.hypixel.hytale.protocol.ItemReticleConfig`) for synchronization with the game client.

Internally, it performs a key optimization during its lifecycle. String-based event names defined in JSON are converted into a more performant integer-indexed map (`indexedServerEvents`) for rapid runtime lookups.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale AssetStore during the asset loading phase at server startup. The static `AssetBuilderCodec`, referenced by the CODEC field, governs the entire instantiation and data-binding process from a source JSON file. Direct instantiation by a developer is an anti-pattern and will result in a non-functional object.
-   **Scope:** The object's lifetime is bound to the AssetStore. Once loaded, an instance persists for the entire server session, or until assets are reloaded.
-   **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for destruction when the AssetStore is cleared, typically during a server shutdown or a full asset reload. There are no manual cleanup methods.

## Internal State & Concurrency

-   **State:** The class is **effectively immutable** after its initial creation and post-processing. All configuration fields (`id`, `base`, `serverEvents`, `clientEvents`) are populated once by the asset loader. No public methods exist to mutate this state.
-   **State (continued):** The class maintains two forms of mutable, internal cache for performance:
    1.  **indexedServerEvents:** An integer-keyed map that is populated once in the `processConfig` method after decoding.
    2.  **cachedPacket:** A `SoftReference` to the generated network packet. This avoids the cost of rebuilding the packet object on every network transmission.
-   **Thread Safety:** This class is **conditionally thread-safe**.
    -   **Reading** configuration data from multiple threads is safe due to its effectively immutable nature.
    -   **Writing**, specifically the lazy initialization within the `toPacket` method, is **not thread-safe**. Concurrent calls to `toPacket` on the same instance can result in a race condition, potentially creating multiple packet objects. If this method must be called from different threads, access must be controlled via external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the singleton AssetStore for all ItemReticleConfig assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Retrieves the map of all loaded ItemReticleConfig assets from the store. |
| getId() | String | O(1) | Returns the unique identifier for this asset, as defined in its source file. |
| toPacket() | com.hypixel.hytale.protocol.ItemReticleConfig | O(N) first call, O(1) subsequent | Converts the asset into a network-ready packet. The result is cached in a SoftReference. N is the number of event configurations. **Warning:** Not thread-safe. |

## Integration Patterns

### Standard Usage

The intended use is to retrieve a pre-loaded configuration from the central AssetStore and convert it to a packet for network transmission.

```java
// How a developer should normally use this
// Retrieve the map of all loaded reticle configurations
IndexedLookupTableAssetMap<String, ItemReticleConfig> reticleMap = ItemReticleConfig.getAssetMap();

// Get a specific configuration by its ID, for example, "Default"
ItemReticleConfig config = reticleMap.get(ItemReticleConfig.DEFAULT_ID);

if (config != null) {
    // Convert the configuration into a network packet
    com.hypixel.hytale.protocol.ItemReticleConfig packet = config.toPacket();

    // Send the packet to the client
    // player.sendPacket(packet);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ItemReticleConfig()`. This bypasses the asset loading pipeline, including the crucial `processConfig` step. The resulting object will be incomplete, lack indexed events, and will not function correctly.
-   **State Modification:** Do not attempt to modify the internal fields of a loaded ItemReticleConfig via reflection or other means. The system relies on these objects being immutable after they are loaded.
-   **Concurrent Packet Creation:** Do not call `toPacket` on the same instance from multiple threads without external locking. This can lead to race conditions and unnecessary object allocation.

## Data Pipeline

The flow of data for this class begins on disk and ends as a packet on the network wire.

> Flow:
> JSON File (`assets/.../item/reticle/Default.json`) -> AssetStore Loader -> **ItemReticleConfig.CODEC** (Deserialization & Validation) -> **ItemReticleConfig Instance** (Post-processing via `processConfig`) -> Game Logic calls `toPacket()` -> Network Packet -> Client Render Engine

