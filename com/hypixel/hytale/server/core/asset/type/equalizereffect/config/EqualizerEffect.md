---
description: Architectural reference for EqualizerEffect
---

# EqualizerEffect

**Package:** com.hypixel.hytale.server.core.asset.type.equalizereffect.config
**Type:** Data Asset

## Definition
```java
// Signature
public class EqualizerEffect
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, EqualizerEffect>>,
   NetworkSerializable<com.hypixel.hytale.protocol.EqualizerEffect> {
```

## Architecture & Concepts

The EqualizerEffect class is a data-driven configuration asset that defines the parameters for a 4-band audio equalizer. It is not a service or manager; rather, it serves as an in-memory representation of a JSON asset file loaded from disk. Its primary role is to bridge the gap between human-readable configuration and the binary data required by the server's audio and network systems.

The core of this class's architecture is the static **CODEC** field, an instance of AssetBuilderCodec. This codec is the single source of truth for serialization and deserialization. It declaratively maps JSON keys (e.g., "LowGain", "HighCutOff") to the class's internal fields. Critically, it also encapsulates business logic during this process, such as:
*   **Data Transformation:** Converting gain values from decibels (a user-friendly unit in JSON) to linear gain (a format required by the audio engine) using AudioUtil.
*   **Validation:** Enforcing value constraints, such as frequency and gain ranges, via Validators.
*   **Inheritance:** Allowing child assets to inherit and override values from parent assets within the JSON structure.

All loaded EqualizerEffect instances are registered in a static, lazily-initialized **AssetStore**. This provides a global, centralized registry for retrieving any defined equalizer effect by its unique string identifier.

Finally, by implementing NetworkSerializable, this class defines a contract for efficient network transmission. The `toPacket` method converts the object into a protocol buffer message, which is the standard format for server-client communication.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale **Asset Loading System** during server bootstrap. The system reads `.json` files, and the static CODEC is invoked to deserialize the data into new EqualizerEffect objects. Direct instantiation is strongly discouraged. The static `EMPTY_EQUALIZER_EFFECT` is a special case, created at class-load time to serve as a null object.
- **Scope:** An EqualizerEffect instance, once loaded into the AssetStore, is a session-scoped singleton. It persists for the entire lifetime of the server process and is shared across all systems that require it.
- **Destruction:** Instances are destroyed when the AssetStore is cleared, which typically occurs only during a full server shutdown or a manual asset reload. The Java garbage collector reclaims the memory.

## Internal State & Concurrency
- **State:** An EqualizerEffect object is **effectively immutable** after its creation and deserialization. While its fields are not marked as `final`, they are populated only once by the CODEC and are not intended to be modified thereafter. The class exposes no public setters. The `cachedPacket` field uses a SoftReference, allowing the garbage collector to reclaim the cached network packet under memory pressure.
- **Thread Safety:** The class is **conditionally thread-safe**. Because its state is effectively immutable post-initialization, it is safe for concurrent reads from multiple threads.

    **WARNING:** The static `getAssetStore` method contains a lazy-initialization pattern that is not protected by a lock. This is safe under the assumption that the first call occurs during the single-threaded server startup sequence. Calling this method for the first time from a concurrent context could lead to a race condition.

    The `toPacket` method's caching logic is not fully atomic, but the consequence of a race condition is minimal: the creation of a redundant packet object, not data corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global registry for all EqualizerEffect assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Provides direct access to the underlying map of asset keys to instances. |
| toPacket() | com.hypixel.hytale.protocol.EqualizerEffect | O(1) | Converts the asset into a network-serializable packet. Uses an internal cache. |
| getId() | String | O(1) | Returns the unique asset key for this effect (e.g., "hytale:default_music_eq"). |

## Integration Patterns

### Standard Usage
An EqualizerEffect should always be retrieved from the global AssetStore or AssetMap via its unique key. It is then typically converted to a packet for network transmission to a client.

```java
// Retrieve the global map of all equalizer effects
IndexedLookupTableAssetMap<String, EqualizerEffect> effects = EqualizerEffect.getAssetMap();

// Look up a specific effect by its asset key
EqualizerEffect zoneEffect = effects.get("hytale:music.overworld_zone_1");

// Convert it to a network packet for transmission to the client's audio engine
if (zoneEffect != null) {
    com.hypixel.hytale.protocol.EqualizerEffect packet = zoneEffect.toPacket();
    // networkSubsystem.sendToClient(player, packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new EqualizerEffect()`. This bypasses the asset pipeline, resulting in an unmanaged object with default values that is invisible to the rest of the engine.
- **State Mutation:** Do not modify the fields of an EqualizerEffect after it has been loaded. This violates its design as an immutable data asset and can lead to unpredictable behavior, especially with the `toPacket` caching mechanism.

## Data Pipeline
The EqualizerEffect class is a key component in the audio configuration pipeline, transforming declarative files into runtime objects used by the network and audio engines.

> Flow:
> JSON Asset File -> Asset Loading System -> **EqualizerEffect.CODEC** (Deserialization & Validation) -> In-Memory **EqualizerEffect** Instance -> `toPacket()` -> Network Subsystem -> Client Audio Engine

