---
description: Architectural reference for ReverbEffect
---

# ReverbEffect

**Package:** com.hypixel.hytale.server.core.asset.type.reverbeffect.config
**Type:** Data Asset

## Definition
```java
// Signature
public class ReverbEffect
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, ReverbEffect>>,
   NetworkSerializable<com.hypixel.hytale.protocol.ReverbEffect> {
```

## Architecture & Concepts

The ReverbEffect class is a data-driven asset that defines the properties of a single audio reverb preset. It is a foundational component of the Hytale **Asset System**, serving as the in-memory representation of a reverb effect configured in external JSON files.

Architecturally, this class embodies several key engine principles:

1.  **Declarative Configuration:** The static **CODEC** field, an AssetBuilderCodec, is the cornerstone of this class. It declaratively defines the entire serialization contract, including JSON keys, data types, validation rules (e.g., range checks), and even property inheritance from parent assets. This decouples the object's structure from the loading logic, allowing designers to configure complex audio effects without modifying engine code.

2.  **Centralized Registry:** All ReverbEffect assets are loaded into and managed by a static **AssetStore**. This singleton-like registry, accessed via the getAssetStore method, ensures that each unique reverb asset is loaded only once and provides a global, canonical source for retrieving effect definitions by their string identifier.

3.  **Network Serialization Bridge:** By implementing NetworkSerializable, this class acts as a bridge between the server's configuration and the client's audio engine. The toPacket method transforms the internal state into a lightweight Data Transfer Object (DTO) suitable for network transmission, ensuring clients have the precise reverb data required for audio rendering.

## Lifecycle & Ownership

-   **Creation:** ReverbEffect instances are **never instantiated directly** by game logic. They are created exclusively by the Hytale Asset System during server bootstrap or asset hot-reloading. The system reads a corresponding JSON file, and the static CODEC deserializes the data into a new ReverbEffect object.

-   **Scope:** The collection of all loaded ReverbEffect assets persists for the entire server session, managed statically within the AssetStore. A reference to a specific ReverbEffect obtained from the store is valid as long as the asset system is active.

-   **Destruction:** The entire AssetStore, including all ReverbEffect instances, is cleared and eligible for garbage collection upon server shutdown or during a full asset reload cycle. The cached network packet within each instance is held by a SoftReference, allowing the garbage collector to reclaim that memory independently if system memory becomes constrained.

## Internal State & Concurrency

-   **State:** An instance of ReverbEffect is **effectively immutable**. Its fields are populated once during deserialization by the CODEC. There are no public methods to modify its state post-creation. This design choice guarantees that a reverb definition remains consistent throughout its lifecycle. Many fields, such as gain, are stored in a linear format pre-converted from the more user-friendly decibel format specified in the JSON configuration.

-   **Thread Safety:** The class is **thread-safe for read operations**. Due to its immutable nature, multiple threads can safely access its properties or call the toPacket method without synchronization.

    **Warning:** The lazy initialization within getAssetStore and the caching mechanism in toPacket are not inherently thread-safe. The engine's design assumes that the AssetStore is fully initialized during a single-threaded startup phase, mitigating this risk. Calling these methods from multiple threads during initial asset loading could lead to unpredictable behavior.

## API Surface

The public API is minimal, focusing on retrieval from the central store and network serialization. Direct interaction with individual properties is handled via standard getters, which are omitted here for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the central, lazily-initialized registry for all ReverbEffect assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Provides direct map-based access to the registered ReverbEffect assets. |
| toPacket() | com.hypixel.hytale.protocol.ReverbEffect | O(1) | Serializes the asset into a network packet. Uses an internal soft-referenced cache to avoid repeated object allocation. |
| getId() | String | O(1) | Returns the unique string identifier for this asset (e.g., "cave_large"). |

## Integration Patterns

### Standard Usage

To apply a reverb effect, you must first retrieve its definition from the global asset map using its unique string identifier.

```java
// How a developer should normally use this
IndexedLookupTableAssetMap<String, ReverbEffect> reverbMap = ReverbEffect.getAssetMap();
ReverbEffect caveReverb = reverbMap.get("hytale:cave_large");

if (caveReverb != null) {
    // Convert to a network packet to send to the client
    com.hypixel.hytale.protocol.ReverbEffect packet = caveReverb.toPacket();
    networkManager.sendPacket(player, packet);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ReverbEffect()`. This bypasses the asset loading system and results in an unmanaged, default-initialized object that does not represent a configured effect. All instances must be retrieved from the AssetStore.

-   **State Modification:** Do not use reflection or other means to modify the fields of a ReverbEffect after it has been loaded. The engine relies on the immutability of these assets for stability and thread safety.

## Data Pipeline

The ReverbEffect class is a key stage in the pipeline that transforms a designer's configuration file into an audible effect for the player.

> Flow:
> ReverbEffect.json File -> Asset Loading System -> **ReverbEffect.CODEC** (Deserializes & Validates) -> **ReverbEffect Instance** (In-memory representation in AssetStore) -> toPacket() -> Network Packet DTO -> Client Audio Engine

