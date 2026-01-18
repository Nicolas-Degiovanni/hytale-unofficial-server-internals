---
description: Architectural reference for AmbienceFXSoundEffect
---

# AmbienceFXSoundEffect

**Package:** com.hypixel.hytale.server.core.asset.type.ambiencefx.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class AmbienceFXSoundEffect implements NetworkSerializable<com.hypixel.hytale.protocol.AmbienceFXSoundEffect> {
```

## Architecture & Concepts

The AmbienceFXSoundEffect class is a data model that represents a specific audio modification within the server's ambient sound system. It is not a service or manager, but rather a configuration object that defines how to apply reverb and equalizer effects to a soundscape.

Its primary architectural role is to act as a bridge between human-readable asset definitions (typically JSON files) and the optimized, integer-based data format required for efficient network transmission to game clients. It decouples the asset design process, which uses descriptive string identifiers like *cave_reverb*, from the runtime engine, which operates on compact numerical indices for performance.

This class is a fundamental component of the data-driven audio system, allowing designers to define complex audio environments declaratively without modifying engine code.

## Lifecycle & Ownership

-   **Creation:** Instances are almost exclusively created by the server's asset loading pipeline via the static **CODEC** field. This process involves deserializing a configuration file (e.g., a biome or zone definition) into an object graph. The provided default constructor is for codec use only.
-   **Scope:** The lifetime of an AmbienceFXSoundEffect object is bound to its parent configuration asset. It persists in memory as long as the biome, region, or other containing asset is loaded. It is not a global or session-scoped object.
-   **Destruction:** The object is eligible for garbage collection when its parent asset is unloaded from memory, for example, during a server shutdown or a dynamic asset reload.

## Internal State & Concurrency

-   **State:** The object holds a mutable state. It contains both the original string identifiers from the asset file (e.g., reverbEffectId) and the derived integer indices (e.g., reverbEffectIndex). The state is transformed during a specific post-deserialization step.

    **Warning:** The state is inconsistent immediately after construction. The integer indices are only populated after the **processConfig** method is invoked by the codec's **afterDecode** hook.

-   **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access during the asset loading phase. The internal fields are accessed without synchronization. After the initial loading and processing, the object should be treated as immutable. Concurrent modification will lead to unpredictable behavior and state corruption.

## API Surface

The public contract is minimal, focused on data retrieval and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.AmbienceFXSoundEffect | O(1) | Serializes the object into a network packet using the pre-calculated integer indices. This is the primary runtime operation. |
| getReverbEffectId() | String | O(1) | Returns the original string identifier for the reverb effect. May be null. |
| getReverbEffectIndex() | int | O(1) | Returns the resolved numerical index for the reverb effect. |
| getEqualizerEffectId() | String | O(1) | Returns the original string identifier for the equalizer effect. May be null. |
| getEqualizerEffectIndex() | int | O(1) | Returns the resolved numerical index for the equalizer effect. |
| isInstant() | boolean | O(1) | Returns true if the audio effect should be applied instantaneously. |

## Integration Patterns

### Standard Usage

This class is not intended for direct manipulation. The engine's asset loader uses the static CODEC to instantiate and configure it from data files. Server logic then retrieves the fully processed object from a parent asset and serializes it for network transmission.

```java
// Example of how the system might use a loaded AmbienceFX configuration

// 1. The object is deserialized from a file by the asset system (not shown).
// 2. Server logic retrieves the parent configuration.
AmbienceFXConfig biomeAmbience = world.getBiome("forest").getAmbienceConfig();

// 3. When needed, it gets the specific effect and converts it to a packet.
AmbienceFXSoundEffect soundEffect = biomeAmbience.getSoundEffect("bird_chirp");
if (soundEffect != null) {
    var packet = soundEffect.toPacket();
    player.getConnection().send(packet);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new AmbienceFXSoundEffect()`. This bypasses the critical validation and index resolution logic handled by the codec. If an instance is created manually, the `processConfig` method must also be called manually, which is brittle and couples your code to its internal implementation details.
-   **State Modification After Load:** Do not modify fields like reverbEffectId after the object has been processed. This will create a state mismatch, as the corresponding integer index will not be updated, leading to incorrect effects being played on the client.
-   **Premature Index Access:** Do not call `getReverbEffectIndex` or `getEqualizerEffectIndex` on a manually created instance before `processConfig` has been invoked. The value will be the default of 0, which may be an invalid or incorrect index.

## Data Pipeline

The flow of data from configuration to the client is a multi-stage process designed for efficiency and maintainability.

> Flow:
> JSON Asset File -> Asset Loader (using **CODEC**) -> **AmbienceFXSoundEffect** (in memory) -> `processConfig()` (ID to Index Resolution) -> `toPacket()` -> Network Packet -> Client Audio Engine

