---
description: Architectural reference for AmbienceFXSound
---

# AmbienceFXSound

**Package:** com.hypixel.hytale.server.core.asset.type.ambiencefx.config
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class AmbienceFXSound implements NetworkSerializable<com.hypixel.hytale.protocol.AmbienceFXSound> {
```

## Architecture & Concepts
AmbienceFXSound is a server-side data configuration class that defines the properties of a single ambient sound effect. It is not a service or manager, but rather a data container that represents a deserialized asset from a configuration file.

Its primary architectural role is to act as an intermediate representation between a human-readable configuration (likely JSON) and a network-optimized binary packet. This is achieved through its static **CODEC** field, which leverages the engine's **BuilderCodec** system. This codec defines the exact structure of the source data, including validation rules that ensure asset integrity at load time. For example, it validates that the referenced **SoundEventId** and **BlockSoundSetId** exist in their respective asset caches.

The class encapsulates all parameters needed to play a sound in the world, such as its frequency, radius, and altitude constraints. The most critical transformation it performs is resolving string-based asset identifiers (e.g., *my_cool_sound_event*) into integer indices, which is a significant performance optimization for network transmission.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the engine's asset loading system via the static **CODEC** field. A higher-level system, such as an **AssetManager**, reads a configuration file and invokes the codec to deserialize the data into a new **AmbienceFXSound** object. The **afterDecode** hook in the codec automatically calls the **processConfig** method to finalize the object's state.

- **Scope:** An **AmbienceFXSound** object's lifetime is bound to its parent asset, typically an **AmbienceFX** configuration. It persists in memory as long as that parent asset is loaded.

- **Destruction:** The object is marked for garbage collection when the parent asset is unloaded from memory. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** The state is mutable during the deserialization process managed by the **BuilderCodec**. However, after the **processConfig** method is invoked, the object should be treated as immutable. The fields **soundEventIndex** and **blockSoundSetIndex** are transient, derived state. They act as a cache, storing the integer-based index of the string-based asset IDs to avoid repeated lookups.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and processed on a single thread during the asset loading phase. Concurrent modification or access, especially before **processConfig** has completed, will lead to unpredictable behavior and state corruption. All interactions with instances of this class must be synchronized by the calling system.

## API Surface
The public API is minimal, primarily consisting of getters for its configured properties and the critical **toPacket** conversion method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.AmbienceFXSound | O(1) | Converts this configuration object into its network-optimized packet representation. |
| getSoundEventId() | String | O(1) | Returns the raw string identifier for the sound event asset. |
| getSoundEventIndex() | int | O(1) | Returns the cached integer index for the sound event. **Warning:** This is only valid after asset loading is complete. |
| getFrequency() | Rangef | O(1) | Returns the configured frequency range for this sound effect. |
| getRadius() | Range | O(1) | Returns the configured radius range for this sound effect. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most game logic developers. It is consumed by the server's ambient sound system, which uses it as a template to spawn sound effects in the world. The system would retrieve a fully-formed instance from a parent asset and use its properties.

```java
// Example of how the sound system might use a pre-loaded instance
AmbienceFX parentAsset = assetManager.get("hytale:forest_ambience");
AmbienceFXSound soundConfig = parentAsset.getRandomSound();

// Convert to a packet for network transmission
com.hypixel.hytale.protocol.AmbienceFXSound packet = soundConfig.toPacket();
networkManager.sendToClient(player, packet);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using **new AmbienceFXSound()**. The object will be in an invalid state, as it bypasses the crucial **CODEC** deserialization and the **processConfig** lifecycle hook. The index fields will remain uninitialized, leading to runtime errors.

- **State Mutation After Load:** Do not modify the fields of an **AmbienceFXSound** object after it has been loaded by the asset system. Changing a value like **soundEventId** will not automatically update the cached **soundEventIndex**, causing a state desynchronization that will be difficult to debug.

## Data Pipeline
This class is a key component in the asset-to-network data pipeline for ambient sounds. It transforms declarative configuration data into an efficient, network-ready format.

> Flow:
> JSON Asset File -> Asset Loader -> **BuilderCodec** -> **AmbienceFXSound Instance** -> **processConfig()** (ID to Index Resolution) -> **toPacket()** -> Network Layer

