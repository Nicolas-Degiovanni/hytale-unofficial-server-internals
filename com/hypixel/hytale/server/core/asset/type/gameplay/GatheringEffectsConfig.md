---
description: Architectural reference for GatheringEffectsConfig
---

# GatheringEffectsConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Configuration Model

## Definition
```java
// Signature
public class GatheringEffectsConfig {
```

## Architecture & Concepts
The GatheringEffectsConfig class is a data model that defines the audio-visual feedback for in-game "gathering" actions, such as mining a block or harvesting a plant. It is not an active service but a passive data container, deserialized from asset configuration files (e.g., JSON) at load time.

Its primary architectural role is to serve as a strongly-typed representation of configuration data, bridging the gap between human-readable asset files and the engine's runtime systems. The static CODEC field is the cornerstone of this class, acting as a schema, deserializer, and validator.

A key optimization pattern is visible in its design: during the `afterDecode` lifecycle hook, it resolves a string-based `soundEventId` into a more performant integer `soundEventIndex`. This pre-calculation avoids costly string-based lookups in performance-sensitive game loops, ensuring that triggering sound effects is a fast, O(1) operation at runtime.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale asset loading pipeline. The static `CODEC` field is invoked by an asset manager to deserialize a configuration block into a new GatheringEffectsConfig object. This typically occurs during server startup or when a world zone is loaded.
- **Scope:** An instance of this class is effectively a singleton for its corresponding asset definition. It is loaded once and then cached in a central asset registry. Its lifetime is tied to the server or client session.
- **Destruction:** The object is marked for garbage collection when the asset registry is cleared, which happens upon server shutdown or client exit. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The state is **effectively immutable** post-initialization. All fields are populated once during the deserialization process managed by the CODEC. The `soundEventIndex` field is transient, derived state that is calculated and cached upon creation.
- **Thread Safety:** This class is **thread-safe for reads**. Once an instance is fully constructed and processed by the `afterDecode` hook, it can be safely accessed by multiple systems (e.g., Game Logic Thread, Audio Thread) without locks or other synchronization primitives. The creation process itself is managed by the asset loader, which is responsible for its own thread safety.

## API Surface
The public API is minimal, providing read-only access to the configured and processed data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getParticleSystemId() | String | O(1) | Returns the unique asset identifier for the particle system to spawn. |
| getSoundEventId() | String | O(1) | Returns the unique asset identifier for the sound event to play. |
| getSoundEventIndex() | int | O(1) | Returns the pre-calculated, optimized integer index for the sound event. **This is the preferred method for runtime use.** |

## Integration Patterns

### Standard Usage
This object is not intended for direct instantiation. It is retrieved from a higher-level manager or asset registry. Game logic then uses the instance to query which effects to trigger for a specific action.

```java
// Example: In a block breaking system
void onBlockGathered(Block block) {
    GatheringEffectsConfig effects = block.getDefinition().getGatheringEffects();
    if (effects != null) {
        // Use the fast, pre-calculated index to play a sound
        int soundIndex = effects.getSoundEventIndex();
        audioSystem.playSoundByIndex(soundIndex);

        // Use the ID to spawn a particle system
        String particleId = effects.getParticleSystemId();
        particleSystem.spawn(particleId, block.getPosition());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new GatheringEffectsConfig()`. Doing so bypasses the entire deserialization and processing pipeline, resulting in a useless object with null fields and an uninitialized `soundEventIndex`.
- **Runtime Modification:** Do not attempt to modify the state of a cached GatheringEffectsConfig instance via reflection or other means. The system relies on this configuration being immutable after it is loaded.
- **Using String IDs in Loops:** Avoid calling `getSoundEventId()` within performance-critical game loops. Always prefer `getSoundEventIndex()` for sound playback to leverage the pre-calculation optimization.

## Data Pipeline
The data for this object originates from a static asset file and is transformed into a cached, optimized in-memory representation.

> Flow:
> Asset File on Disk (e.g., `stone.json`) -> Asset Loading Service -> **GatheringEffectsConfig.CODEC** -> Deserialization & Validation -> `processConfig()` Hook (ID to Index conversion) -> In-Memory Asset Cache

