---
description: Architectural reference for AmbienceFXAmbientBed
---

# AmbienceFXAmbientBed

**Package:** com.hypixel.hytale.server.core.asset.type.ambiencefx.config
**Type:** Transient

## Definition
```java
// Signature
public class AmbienceFXAmbientBed implements NetworkSerializable<com.hypixel.hytale.protocol.AmbienceFXAmbientBed> {
```

## Architecture & Concepts
The AmbienceFXAmbientBed class is a configuration model object, not a service. Its primary role is to represent a single, continuous ambient sound layer, often referred to as an "ambient bed". This class acts as a strongly-typed, validated data structure that is deserialized from asset configuration files.

The core of this class is its static **CODEC** field. This `BuilderCodec` defines the schema for an ambient bed within an asset file, including data types, validation rules, and default values. It is the single source of truth for how AmbienceFXAmbientBed data is loaded and validated by the engine's asset pipeline.

This class serves as a critical bridge between three distinct systems:
1.  **Asset System:** The CODEC is used to parse and validate raw asset data (e.g., JSON) into an in-memory AmbienceFXAmbientBed object.
2.  **Audio System:** It holds audio-specific parameters like the sound track, volume in decibels, and transition speed. The `processConfig` method translates the editor-friendly decibel value into a linear gain value required by the audio engine.
3.  **Network System:** Through the NetworkSerializable interface, it defines how its state is packaged into a network message (`toPacket`) for transmission to the client, ensuring the client's audio engine plays the correct sound with the correct parameters.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale **Codec** system during the asset loading phase. The static CODEC field is invoked by a higher-level asset parser which feeds it raw configuration data. Manual instantiation is an anti-pattern.
-   **Scope:** An AmbienceFXAmbientBed object exists for as long as its parent asset is loaded in memory. It is effectively a read-only data container after its initial creation and post-processing.
-   **Destruction:** The object is eligible for garbage collection when the parent asset (e.g., a biome configuration) is unloaded and all references to it are released. There is no manual destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The state is effectively **immutable** after deserialization. The `afterDecode` hook in the CODEC calls `processConfig`, which performs the only state mutation by calculating the `volume` field from the `decibels` field. The `volume` field is marked `transient` and is not part of the serialized asset state; it is derived, runtime-only data.
-   **Thread Safety:** This class is **thread-safe for reads**. As its state does not change after the initial loading phase, multiple systems can safely access its properties from different threads without synchronization. The creation process itself is managed by the asset loading pipeline, which must handle its own concurrency.

## API Surface
The public contract is minimal, focused on data retrieval and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.AmbienceFXAmbientBed | O(1) | Serializes the object into its network packet representation. This is the primary method used at runtime to synchronize state with the client. |
| getTrack() | String | O(1) | Returns the asset path for the sound file. |
| getVolume() | float | O(1) | Returns the calculated linear gain volume (0.0 to 1.0+), suitable for an audio engine. |
| getDecibels() | float | O(1) | Returns the raw decibel value from the configuration file. Primarily for debugging or tooling. |
| getTransitionSpeed() | AmbienceTransitionSpeed | O(1) | Returns the configured speed for fading this sound bed in or out. |

## Integration Patterns

### Standard Usage
This object is not retrieved directly. It is typically part of a larger configuration object, such as an AmbienceFX. A game system would access it to apply its properties.

```java
// A hypothetical AmbienceManager processes an AmbienceFX asset
public void applyAmbience(AmbienceFX ambience) {
    for (AmbienceFXAmbientBed bed : ambience.getAmbientBeds()) {
        // Convert the configuration object to a network packet
        com.hypixel.hytale.protocol.AmbienceFXAmbientBed packet = bed.toPacket();

        // Send the packet to the relevant players
        networkService.sendToAll(packet);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new AmbienceFXAmbientBed()`. This bypasses the entire validation and data processing pipeline defined in the CODEC, such as the critical conversion from decibels to linear volume. Objects created this way will be malformed and cause runtime errors.
-   **State Mutation:** Do not attempt to modify the state of this object after it has been loaded. It is designed as a read-only representation of asset data. Modifying it would lead to a state mismatch between the server's logic and the original asset configuration.

## Data Pipeline
The flow of data from configuration file to the client's audio engine is linear and well-defined.

> Flow:
> Asset File (JSON) -> Asset Loader -> **BuilderCodec** -> **AmbienceFXAmbientBed (Instance)** -> `toPacket()` -> Network Packet -> Client Audio Engine

