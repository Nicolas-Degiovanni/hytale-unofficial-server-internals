---
description: Architectural reference for AmbienceFXBlockSoundSet
---

# AmbienceFXBlockSoundSet

**Package:** com.hypixel.hytale.server.core.asset.type.ambiencefx.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class AmbienceFXBlockSoundSet implements NetworkSerializable<com.hypixel.hytale.protocol.AmbienceFXBlockSoundSet> {
```

## Architecture & Concepts
The AmbienceFXBlockSoundSet class is a server-side configuration model that defines a relationship between a specific BlockSoundSet asset and a probability range. It acts as a critical intermediary component, translating human-readable asset configurations into a network-optimized format.

Its primary architectural role is to decouple the asset definition system from the network protocol. Asset designers can specify sounds using string-based identifiers (e.g., *stone_footsteps*) in configuration files. During server startup or asset loading, this class is instantiated, and its internal logic resolves the string identifier into a more performant integer index. This resolved index, not the original string, is what gets serialized into a network packet for transmission to the client.

This ID-to-Index resolution pattern is a core engine optimization, reducing packet size and client-side lookup complexity. This class encapsulates that translation logic for ambient sound effects.

## Lifecycle & Ownership
- **Creation:** Instances are not created manually. They are exclusively instantiated by the Hytale asset pipeline via the static **CODEC** field. The codec deserializes data from a configuration file (e.g., a JSON asset) into a new AmbienceFXBlockSoundSet object.

- **Scope:** The object's lifetime is tied to its parent configuration asset, typically an AmbienceFX definition. It is a transient object that exists only after asset loading and before the final game data is compiled for distribution to clients.

- **Destruction:** Once the parent asset has been fully processed and its data converted into network packets or other runtime formats, the AmbienceFXBlockSoundSet instance is no longer referenced and becomes eligible for garbage collection. It does not persist for the entire server session.

## Internal State & Concurrency
- **State:** The class holds mutable state upon creation. The **CODEC** populates the blockSoundSetId and percent fields. A crucial state transition occurs when the codec invokes the internal processConfig method, which populates the transient blockSoundSetIndex field. After this point, the object's state should be considered effectively immutable.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and processed on a single thread, typically the main server thread or a dedicated asset-loading thread. Unsynchronized access from multiple threads will lead to race conditions, particularly if one thread attempts to read the blockSoundSetIndex while another is triggering its calculation.

## API Surface
The public API is minimal, focusing on data retrieval and conversion to a network packet.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.AmbienceFXBlockSoundSet | O(1) | Converts this configuration object into its corresponding network packet representation. This is the primary consumption pathway. |
| getBlockSoundSetId() | String | O(1) | Returns the original, human-readable string identifier for the BlockSoundSet. |
| getPercent() | Rangef | O(1) | Returns the configured probability or intensity range for this sound effect. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is rare. It is managed entirely by the server's asset loading and processing systems. A higher-level system, such as an AmbienceFX manager, would hold a collection of these objects after deserialization and use them to build the final runtime data.

The conceptual flow within the engine is as follows:

```java
// Engine-level code, not for typical developer use.
// The CODEC is used by the asset loader to create instances from a file.
AmbienceFX parentAsset = assetLoader.load("data/ambience/forest.json");

// The parent asset now contains a list of AmbienceFXBlockSoundSet objects.
// The engine can then convert them to packets.
for (AmbienceFXBlockSoundSet soundSet : parentAsset.getSounds()) {
    protocol.AmbienceFXBlockSoundSet packet = soundSet.toPacket();
    // ... send packet to client
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new AmbienceFXBlockSoundSet()`. Doing so bypasses the **CODEC** and the critical `afterDecode` hook that calls `processConfig`. This will result in an uninitialized `blockSoundSetIndex` (value 0), causing the client to play the wrong sound or no sound at all.

- **Post-Creation Modification:** Do not modify the state of this object after it has been loaded by the asset system. Changing the blockSoundSetId after `processConfig` has run will create an inconsistent state where the ID and the resolved index no longer match.

## Data Pipeline
This class is a key step in the data transformation pipeline from server configuration to client-side effect rendering.

> Flow:
> JSON Asset File -> Asset Loader with **CODEC** -> **AmbienceFXBlockSoundSet Instance** (ID resolved to Index) -> toPacket() -> Network Packet -> Client Game Engine<ctrl63>

