---
description: Architectural reference for SoundEvent
---

# SoundEvent

**Package:** com.hypixel.hytale.server.core.asset.type.soundevent.config
**Type:** Data Asset

## Definition
```java
// Signature
public class SoundEvent
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, SoundEvent>>,
   NetworkSerializable<com.hypixel.hytale.protocol.SoundEvent> {
```

## Architecture & Concepts

The SoundEvent class is a foundational data asset within the Hytale audio system. It does not represent an audio file itself, but rather the complete configuration for how a sound is triggered and behaves in the game world. Each instance defines properties such as volume, pitch, spatial attenuation, layering of multiple sounds, and instance limiting.

This class is designed to be loaded from JSON configuration files by the engine's core Asset System. Its integration with this system is signaled by the implementation of the JsonAssetWithMap interface. The static field CODEC, an instance of AssetBuilderCodec, declaratively defines the entire deserialization process. This codec is responsible for mapping JSON keys to object fields, applying validators, and crucially, handling property inheritance from parent assets. This allows designers to create variations of sound events without duplicating configuration.

The implementation of NetworkSerializable indicates that a SoundEvent's configuration is critical for client-side prediction and playback. The server can serialize a SoundEvent into a compact network packet and send it to the client, ensuring that both server and client have an identical understanding of how a specific sound should be played.

## Lifecycle & Ownership

-   **Creation:** SoundEvent instances are exclusively instantiated by the AssetStore during the engine's asset loading phase at startup. The static CODEC field acts as the factory, deserializing JSON definitions from disk into memory. Manual instantiation is a critical anti-pattern. A special static instance, EMPTY_SOUND_EVENT, is created at class load time to represent a null or non-existent sound.

-   **Scope:** An instance of SoundEvent, once loaded, persists for the entire application session. It is stored and owned by the central AssetStore, which acts as a global registry.

-   **Destruction:** Objects are eligible for garbage collection only when the AssetStore is cleared, which typically occurs during application shutdown.

## Internal State & Concurrency

-   **State:** The state of a SoundEvent is considered **effectively immutable** after its initial loading and processing. During deserialization, the `afterDecode` hook triggers the `processConfig` method, which calculates and caches derived data like `audioCategoryIndex` and `highestNumberOfChannels`.

    The class contains a mutable cache field, `cachedPacket`, which holds a SoftReference to the generated network packet. This is a performance optimization to avoid repeated serialization.

-   **Thread Safety:** The class is **not thread-safe** for mutation but is **safe for concurrent reads**. As all instances are loaded in a single-threaded context at startup and not modified thereafter, reading a SoundEvent's properties from multiple game threads is safe.

    **Warning:** The lazy initialization of the static ASSET_STORE field in `getAssetStore` is not synchronized. If multiple threads attempt to access the asset store for the first time concurrently, it could lead to race conditions. Similarly, the `toPacket` method's cache update is not atomic, but the consequence of a race (re-creating a packet) is minor.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, singleton registry for all SoundEvent assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Provides direct access to the underlying map structure for fast, indexed lookups. |
| getId() | String | O(1) | Returns the unique string identifier for the asset (e.g., "hytale.sound.player.footstep"). |
| getLayers() | SoundEventLayer[] | O(1) | Returns the array of sound layers that compose this event. |
| toPacket() | com.hypixel.hytale.protocol.SoundEvent | O(N) / O(1) | Serializes the object into a network packet. Complexity is O(N) on first call (N=layers), and O(1) on subsequent calls due to caching. |

## Integration Patterns

### Standard Usage

A SoundEvent should always be retrieved from the central AssetStore via its unique string key. It is then typically passed to an audio playback service to be instantiated in the game world.

```java
// How a developer should normally use this
IndexedLookupTableAssetMap<String, SoundEvent> soundEvents = SoundEvent.getAssetMap();
SoundEvent footstepSound = soundEvents.get("hytale.sound.player.footstep.grass");

if (footstepSound != null) {
    // The audio engine would then use this configuration to play the sound
    // at a specific location for a specific entity.
    AudioEngine.playSoundAt(entity.getPosition(), footstepSound);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new SoundEvent()`. The asset system is the sole owner and creator of these objects. Bypassing it will lead to unmanaged, disconnected objects that the rest of the engine cannot find or use.

-   **State Mutation:** Do not modify the public fields of a SoundEvent instance after it has been loaded. These objects are shared globally, and any mutation will cause unpredictable and difficult-to-debug audio behavior throughout the entire application.

-   **Null Checks:** Instead of checking for null when a sound event might be missing, check against the static `SoundEvent.EMPTY_SOUND_EVENT` constant. This provides a safe, non-functional default.

## Data Pipeline

The SoundEvent class is a key component in the pipeline that transforms declarative JSON files into audible in-game effects.

> **Configuration to In-Memory Asset:**
> `*.soundevent.json` -> Asset Loader -> **AssetBuilderCodec** -> `SoundEvent` Instance -> Populated in `AssetStore`

> **Server Gameplay to Client Audio:**
> Server Game Logic -> `SoundEvent.getAssetStore().get(id)` -> `soundEvent.toPacket()` -> Network Layer -> Client Network Handler -> **`protocol.SoundEvent`** -> Client Audio Engine -> Sound Played
---

