---
description: Architectural reference for Weather
---

# Weather

**Package:** com.hypixel.hytale.server.core.asset.type.weather.config
**Type:** Asset Model

## Definition
```java
// Signature
public class Weather implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, Weather>>, NetworkSerializable<com.hypixel.hytale.protocol.Weather> {
```

## Architecture & Concepts
The Weather class is a server-side data model that defines the complete set of environmental and rendering parameters for a specific in-game weather type, such as *sunny*, *rainy*, or *snowy*. It functions as a static configuration asset, loaded from JSON files at server startup.

Its primary role is to act as a bridge between the server's asset configuration system and the client's rendering engine. It achieves this by implementing the NetworkSerializable interface, allowing an instance of a Weather asset to be converted into a network-ready protocol message. When the server decides to change the weather, it finds the appropriate Weather object in the AssetStore, serializes it into a packet using the toPacket method, and broadcasts it to all relevant clients.

The structure of the corresponding JSON asset file is declaratively defined by the static **CODEC** field, an instance of AssetBuilderCodec. This powerful codec handles deserialization, validation, and property inheritance from parent weather assets, forming the backbone of the asset loading pipeline for this type.

## Lifecycle & Ownership
- **Creation:** Weather objects are instantiated exclusively by the Hytale AssetStore during the server's initial asset loading phase. The static CODEC is used to parse a corresponding JSON file (e.g., `sunny.json`) into a fully-populated Weather object in memory. Direct instantiation by developers is strictly forbidden.

- **Scope:** Once loaded, a Weather object persists for the entire lifetime of the server process. It is considered part of the server's immutable, static game data.

- **Destruction:** The object is eligible for garbage collection only when the server shuts down and the AssetRegistry is cleared.

## Internal State & Concurrency
- **State:** An instance of Weather is highly stateful, containing numerous arrays and fields that define colors, textures, fog parameters, and particle effects over the in-game day-night cycle. After being deserialized from JSON, the object is **effectively immutable**. There are no public methods to mutate its state. It contains a `cachedPacket` field, a SoftReference to the last generated network packet, as a performance optimization to avoid repeated serialization.

- **Thread Safety:** This class is **not thread-safe**.
    - The static `getAssetStore` method uses a non-thread-safe lazy initialization pattern for the static ASSET_STORE field. Concurrent calls to this method during server initialization from different threads will result in a race condition. Access to this method must be externally synchronized during the server bootstrap phase.
    - The `toPacket` method contains a benign race condition when writing to the `cachedPacket` field. If multiple threads call it concurrently on the same instance, they may redundantly generate the same packet, but this will not cause data corruption. However, reads and writes of the object's primary state are safe, as the state is not mutated after initialization.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | **STATIC**. Retrieves the global, shared AssetStore for all Weather assets. **WARNING:** Not thread-safe on first call. |
| getAssetMap() | IndexedLookupTableAssetMap | O(1) | **STATIC**. A convenience method to get the underlying asset map from the AssetStore. |
| toPacket() | com.hypixel.hytale.protocol.Weather | O(N) | Converts the asset's configuration into a network protocol message for client consumption. The complexity is proportional to the number of time-based keyframes in the asset. |
| getId() | String | O(1) | Returns the unique identifier for this weather asset, such as "hytale:sunny". |

## Integration Patterns

### Standard Usage
The intended use is to retrieve a pre-loaded Weather instance from the central AssetStore and use it to generate a network packet. This is typically done by a world or zone management system.

```java
// Retrieve the asset map for all weather types
IndexedLookupTableAssetMap<String, Weather> weatherAssets = Weather.getAssetMap();

// Get a specific weather configuration by its ID
Weather rainyWeather = weatherAssets.get("hytale:rainy");

if (rainyWeather != null) {
    // Convert the asset to a network packet
    com.hypixel.hytale.protocol.Weather weatherPacket = rainyWeather.toPacket();

    // Broadcast the packet to players in the world
    world.broadcastPacket(weatherPacket);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new Weather()`. Weather objects must be defined in JSON assets and loaded through the AssetStore. Manual creation results in an unmanaged object that lacks critical metadata and will not be recognized by other game systems.

- **Concurrent Initialization:** Do not call `Weather.getAssetStore()` from multiple threads simultaneously during server startup without an external locking mechanism. This will lead to a race condition.

- **State Mutation:** Do not attempt to modify the fields of a Weather object after it has been loaded using reflection or other means. The game engine expects these assets to be immutable.

## Data Pipeline
The Weather asset is a key component in the pipeline that transforms a static configuration file into a dynamic visual experience for the player.

> Flow:
> JSON Asset File (`rainy.json`) -> Server AssetStore Loader -> **Weather Object (In Memory)** -> `toPacket()` -> Network Protocol Message -> Client Rendering Engine -> Visual Weather Effects (Fog, Rain, Sky Color)

