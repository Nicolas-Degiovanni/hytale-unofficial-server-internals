---
description: Architectural reference for SoundEventLayer
---

# SoundEventLayer

**Package:** com.hypixel.hytale.server.core.asset.type.soundevent.config
**Type:** Configuration / Data Model

## Definition
```java
// Signature
public class SoundEventLayer implements NetworkSerializable<com.hypixel.hytale.protocol.SoundEventLayer> {
```

## Architecture & Concepts
The SoundEventLayer is a data model that represents a single, configurable layer of sound within a larger, composite sound event. In Hytale's audio system, a single event, such as a block breaking, can be composed of multiple layers to create a rich and dynamic audio experience. This class defines the properties for one such layer, including which audio files to play, their volume, pitch, and playback probability.

Its primary role is to act as a server-side representation of configuration defined in an asset file, likely JSON. The class is fundamentally tied to the engine's `Codec` system, which is responsible for deserializing the asset file into a hydrated SoundEventLayer object at runtime. The static `CODEC` field is the contract that defines this mapping, including data validation, transformations (e.g., decibels to linear gain), and post-processing logic.

Crucially, this class implements `NetworkSerializable`. This signifies that after the server loads and processes the sound configuration from disk, it serializes this object into a more compact network packet (`com.hypixel.hytale.protocol.SoundEventLayer`) to be sent to the client. The client's audio engine then uses this network object to play the sound, ensuring server-authoritative audio behavior.

### Nested RandomSettings
The SoundEventLayer contains a nested static class, `RandomSettings`, which is also a `Codec`-driven data model. This component encapsulates properties for randomizing audio parameters like volume and pitch within a specified range, further enhancing the dynamic nature of the sound. It follows the same architectural principles as its parent class.

## Lifecycle & Ownership
- **Creation:** Instances are not intended to be created manually via the `new` keyword. The Hytale asset pipeline instantiates this class exclusively through its static `CODEC` field during the deserialization of sound event asset files. The `BuilderCodec` invokes the private constructor as part of this automated process.
- **Scope:** A SoundEventLayer object exists for the lifetime of its parent sound asset. These assets are typically loaded into an asset manager upon server startup and persist for the entire server session.
- **Destruction:** The object is eligible for garbage collection when the server shuts down or the asset manager is explicitly cleared.

## Internal State & Concurrency
- **State:** The object's state is mutable during the asset loading and decoding process. However, once the `afterDecode` hook is executed, it should be treated as an immutable data record. It contains derived, runtime-only state in the `highestNumberOfChannels` field, which is calculated by inspecting the referenced OGG sound files. This field is marked `transient` to indicate it is not part of the original asset definition.

- **Thread Safety:** **This class is not thread-safe.** It is a plain data object with no internal synchronization mechanisms. It is designed to be created and populated on a single asset-loading thread and subsequently read by multiple game logic threads. Any external mutation of a loaded SoundEventLayer instance post-initialization is a severe anti-pattern and will result in unpredictable audio behavior across the server.

## API Surface
The public API is minimal, primarily exposing configuration data and the network serialization mechanism.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.SoundEventLayer | O(1) | Serializes the object's state into a network packet for client consumption. This is the primary integration point with the networking layer. |
| get...() | various | O(1) | A collection of standard getters providing read-only access to the layer's configured properties, such as volume, files, and looping behavior. |

## Integration Patterns

### Standard Usage
Developers do not typically interact with this class directly in code. Instead, they define its properties within a sound event asset file. The engine's sound system consumes the loaded object.

The following conceptual example shows how the engine might use a loaded layer to prepare a sound for playback.

```java
// Engine-level code, not for typical gameplay developers.
// Assume 'soundEvent' is loaded from an AssetManager and contains 'layers'.

for (SoundEventLayer layer : soundEvent.getLayers()) {
    if (shouldPlayLayer(layer.getProbability())) {
        // The sound system would use the layer's data to configure an audio source.
        AudioSource source = audioEngine.createSource();
        String fileToPlay = selectFile(layer.getFiles(), layer.getRoundRobinHistorySize());
        
        source.setFile(fileToPlay);
        source.setVolume(layer.getVolume());
        source.setLooping(layer.isLooping());
        
        // ... and so on, using other properties from the layer.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SoundEventLayer()`. This bypasses the `Codec` system, skipping critical validation, data transformation (e.g., decibel conversion), and post-decode processing like calculating channel counts. All instances must be created by the asset loader.
- **Post-Load Mutation:** Do not modify the state of a SoundEventLayer object after it has been loaded. These objects are shared resources. Mutating one will affect every future playback of that sound event across the entire server, leading to inconsistent and unpredictable behavior.

## Data Pipeline
The SoundEventLayer serves as an in-memory, server-side representation of data that flows from a configuration file to the game client.

> Flow:
> Sound Asset File -> Server Asset Loader (using Codec) -> **SoundEventLayer (Server Memory)** -> toPacket() -> Network Packet -> Client Audio Engine

