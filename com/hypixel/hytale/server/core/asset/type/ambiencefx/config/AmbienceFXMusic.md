---
description: Architectural reference for AmbienceFXMusic
---

# AmbienceFXMusic

**Package:** com.hypixel.hytale.server.core.asset.type.ambiencefx.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class AmbienceFXMusic implements NetworkSerializable<com.hypixel.hytale.protocol.AmbienceFXMusic> {
```

## Architecture & Concepts

The AmbienceFXMusic class is a server-side data model that defines the musical properties for a specific ambient effect. It does not manage audio playback directly; instead, it serves as a validated, structured configuration object that is loaded from asset files and can be serialized for transmission to the client.

Its primary architectural role is to act as a container for ambient music settings, enforced by a strict schema defined via the static **CODEC** field. This `BuilderCodec` is the cornerstone of the class, ensuring that all instances loaded from disk are valid. It performs critical functions such as:
-   Validating that the list of music tracks is not empty.
-   Ensuring each track path points to a valid music asset.
-   Constraining the volume to a permissible range.

This class embodies Hytale's data-driven design philosophy. Game designers can define complex ambient soundscapes in external configuration files, and the engine uses this class as the safe, in-memory representation of that data. The implementation of the NetworkSerializable interface signifies its secondary role: a Data Transfer Object (DTO) used to synchronize ambient state with the game client.

### Lifecycle & Ownership
-   **Creation:** Instances are exclusively created by the Hytale asset loading pipeline via the static **CODEC** field. A higher-level system, such as an AmbienceFX asset manager, deserializes a configuration file (e.g., a JSON file defining a biome's atmosphere) which in turn instantiates AmbienceFXMusic. Manual instantiation is an anti-pattern.
-   **Scope:** The object's lifetime is bound to the parent asset that contains it. It persists in memory as long as its containing configuration is considered active by the server. The object is effectively immutable after the `afterDecode` hook completes its processing.
-   **Destruction:** The object is eligible for garbage collection when the parent asset is unloaded from memory. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The class holds configuration state, including a list of track asset paths and a volume level in decibels. A transient `volume` field caches the linear gain representation of the decibels, calculated once during the `afterDecode` lifecycle hook. This state is considered **effectively immutable** post-initialization.
-   **Thread Safety:** The object is **thread-safe for read operations**. As its state does not change after being loaded, multiple threads can safely access its properties. The creation and deserialization process, managed by the asset pipeline, is assumed to be single-threaded or otherwise synchronized. Writing to its fields after creation is not supported and would lead to undefined behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.AmbienceFXMusic | O(N) | Serializes the object into a network packet for client synchronization. N is the number of tracks. |
| getTracks() | String[] | O(1) | Returns the array of music asset paths. |
| getDecibels() | float | O(1) | Returns the raw volume value in decibels as defined in the asset file. |
| getVolume() | float | O(1) | Returns the pre-calculated linear gain volume, suitable for use by an audio engine. |

## Integration Patterns

### Standard Usage
This object is not intended to be created or managed directly. It is retrieved from a parent configuration asset and used to either configure server-side systems or to generate a network packet for the client.

```java
// Example: A server system retrieves an ambience config and sends it to a client
AmbienceConfiguration ambienceConfig = assetManager.get(AmbienceConfiguration.class, "zone.plains.day");
AmbienceFXMusic musicSettings = ambienceConfig.getMusic();

// The toPacket method is used to create the network-ready object
if (musicSettings != null) {
    player.getConnection().send(musicSettings.toPacket());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new AmbienceFXMusic()`. The static CODEC is the only supported mechanism for creation, as it guarantees that validation and post-processing logic (like volume calculation) are executed.
-   **State Mutation:** Do not modify the internal state of this object after it has been loaded. It is a read-only snapshot of the configuration. Modifying the array returned by getTracks would have unpredictable side effects.
-   **Using Decibels for Playback:** Do not use the value from getDecibels directly for audio engine volume. The getVolume method provides the correct, pre-calculated linear gain value required for playback.

## Data Pipeline
The AmbienceFXMusic object is a critical link in the chain that transforms a designer's configuration file into audible music for the player.

> Flow:
> Asset File (e.g., JSON) -> Server Asset Loader -> **BuilderCodec** -> **AmbienceFXMusic Instance** -> Game Logic -> `toPacket()` -> Network Layer -> Client -> Client Audio Engine

