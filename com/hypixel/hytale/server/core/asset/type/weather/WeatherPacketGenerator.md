---
description: Architectural reference for WeatherPacketGenerator
---

# WeatherPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.weather
**Type:** Utility

## Definition
```java
// Signature
public class WeatherPacketGenerator extends SimpleAssetPacketGenerator<String, Weather, IndexedLookupTableAssetMap<String, Weather>> {
```

## Architecture & Concepts
The WeatherPacketGenerator is a specialized, stateless component within the server-side asset synchronization framework. Its sole responsibility is to act as a translator, converting the server's in-memory representation of Weather assets into a compact, network-efficient binary format for transmission to clients.

This class is a critical bridge between the server's Asset Management System and the Hytale Network Protocol. It implements a generic pattern, extending SimpleAssetPacketGenerator, which indicates that similar generators exist for other asset types (e.g., Models, Textures, Sounds).

The core architectural concept is the use of an IndexedLookupTableAssetMap. Instead of sending verbose string identifiers like "hytale:sunny_day" over the network, the generator uses the map to convert these keys into small, consistent integer indices. This significantly reduces packet size and client-side lookup time. The generator is responsible for packaging these integer-indexed Weather definitions into a dedicated UpdateWeathers packet.

## Lifecycle & Ownership
- **Creation:** An instance of WeatherPacketGenerator is created on-demand by a higher-level asset management service, likely when that service detects changes to Weather assets that must be synchronized with connected clients. It is not a long-lived, persistent service.
- **Scope:** The object's lifetime is extremely short, typically scoped to a single asset synchronization operation. It is instantiated, one of its methods is called, and it is then eligible for garbage collection.
- **Destruction:** The Java Garbage Collector reclaims the object once the calling method completes and the reference falls out of scope. There is no manual destruction or cleanup logic required.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no member fields and all necessary data is provided as arguments to its methods. Its behavior is entirely deterministic based on its inputs.
- **Thread Safety:** The stateless nature of this class makes it inherently **thread-safe**. An instance can be safely shared and used by multiple threads without any risk of race conditions or data corruption. No locking mechanisms are employed or required.

## API Surface
The public API is designed around the three fundamental types of asset synchronization: initial load, incremental update, and removal.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet containing all specified weather assets. Designed for a client's initial connection. Throws IllegalArgumentException if any asset key is not found in the assetMap. |
| generateUpdatePacket(assetMap, loadedAssets) | Packet | O(N) | Creates an incremental update packet for new or modified weather assets. Used for live-reloading or adding assets post-connection. Throws IllegalArgumentException for unknown keys. |
| generateRemovePacket(assetMap, removed) | Packet | O(N) | Creates a packet instructing the client to remove one or more weather assets, identified by their keys. Throws IllegalArgumentException for unknown keys. |

## Integration Patterns

### Standard Usage
The generator should be invoked by a service responsible for managing the lifecycle of Weather assets. The service provides the canonical asset map and the delta (added, updated, or removed assets) to be synchronized.

```java
// Within a hypothetical WeatherAssetService during a sync operation
IndexedLookupTableAssetMap<String, Weather> weatherMap = this.getWeatherAssetMap();
Map<String, Weather> changedAssets = this.getRecentlyUpdatedAssets();

// Generator is created and used immediately
WeatherPacketGenerator generator = new WeatherPacketGenerator();
Packet updatePacket = generator.generateUpdatePacket(weatherMap, changedAssets);

// The packet is then dispatched to the network layer
networkManager.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Long-Term Storage:** Do not create an instance of WeatherPacketGenerator and hold a reference to it. It is a stateless utility and should be created and discarded within the scope of a single operation.
- **Using Stale Asset Maps:** The IndexedLookupTableAssetMap passed to any generator method **must** be the current, authoritative version. Passing a stale map that does not contain the keys from the second argument will cause a runtime crash via IllegalArgumentException. This can occur if assets are loaded without first registering them in the primary lookup map.
- **Inefficient Packet Generation:** Do not call generateInitPacket for small, incremental changes. This forces the client to re-process its entire weather asset collection and wastes significant bandwidth. Use generateUpdatePacket for deltas.

## Data Pipeline
The WeatherPacketGenerator sits at a specific point in the server-to-client data flow for asset synchronization. It performs the crucial serialization step.

> Flow:
> Server Asset Change (e.g., Mod Load) -> Asset Service detects delta -> **WeatherPacketGenerator** serializes Weather objects into an UpdateWeathers packet -> Network Layer sends packet to Client -> Client Network Layer deserializes packet -> Client Asset Registry updates its local Weather definitions

