---
description: Architectural reference for CameraEffectsConfig
---

# CameraEffectsConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Configuration Object

## Definition
```java
// Signature
public class CameraEffectsConfig {
```

## Architecture & Concepts
The CameraEffectsConfig class is a data-centric configuration object responsible for mapping server-side damage events to client-side camera effects. It serves as a critical link between the gameplay logic of a damage source and the visual feedback a player receives.

Its primary architectural feature is the static **CODEC** field, which leverages Hytale's declarative `BuilderCodec` system. This system defines how the object is deserialized from an asset file (e.g., JSON) into a memory-efficient runtime representation.

The core responsibility of this class is to translate a human-readable map of string identifiers (DamageCause name to CameraEffect name) into a high-performance integer-to-integer map. This transformation occurs once during the asset loading pipeline, ensuring that lookups during gameplay are extremely fast. This is a classic example of a "bake" or "cook" step, where data is pre-processed at load-time to optimize for runtime performance.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via the constructor. They are instantiated exclusively by the Hytale asset loading system, which uses the static **CODEC** field to deserialize a configuration file from disk. The `afterDecode` hook within the codec is the final step of its construction, populating the performance-critical internal state.
- **Scope:** An instance of CameraEffectsConfig persists for the lifetime of the asset it represents. It is loaded during server initialization or world loading and remains in memory until the server shuts down or the asset is unloaded.
- **Destruction:** The object is eligible for garbage collection when the server's asset registry is cleared, typically during a server shutdown sequence.

## Internal State & Concurrency
- **State:** The class maintains two primary state fields:
    1.  **damageEffectIds**: A `Map<String, String>` holding the raw, human-readable data as defined in the configuration file. This field is populated during deserialization and is used only during the `afterDecode` phase.
    2.  **damageEffectIndices**: A `transient Int2IntMap` that stores the optimized, integer-indexed mapping. This map is the "source of truth" for all runtime lookups. The `transient` keyword ensures it is not part of the serialized form and must be derived at load time.

    The object's state is effectively immutable after the `afterDecode` lifecycle hook completes.

- **Thread Safety:** The class is thread-safe under its intended usage model. The initialization and population of its internal maps occur in a single-threaded context during the server's asset loading phase. All subsequent public API access via `getCameraEffectIndex` is a read-only operation.

    **Warning:** The underlying `Int2IntOpenHashMap` is not inherently thread-safe for concurrent writes. Do not attempt to modify the internal state of this object from multiple threads after it has been initialized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCameraEffectIndex(int damageCauseIndex) | int | O(1) | Retrieves the integer index for a CameraEffect based on the provided DamageCause index. Returns a sentinel value (Integer.MIN_VALUE) if no mapping exists. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly in most gameplay code. Instead, a higher-level system, such as a DamageManager, would hold a reference to the loaded configuration. When an entity is damaged, that manager performs the lookup.

```java
// Example from a hypothetical DamageProcessingSystem
CameraEffectsConfig effectsConfig = assetManager.get("myworld:camera_effects.json");
int fireDamageIndex = DamageCause.getAssetMap().getIndex("hytale:fire");

// During a damage event...
int effectIndex = effectsConfig.getCameraEffectIndex(fireDamageIndex);
if (effectIndex != Integer.MIN_VALUE) {
    // Send packet to client to play camera effect with this index
    player.getNetworkConnection().send(new PacketPlayCameraEffect(effectIndex));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new CameraEffectsConfig()`. The resulting object will be empty and non-functional, as it bypasses the critical `CODEC` deserialization and processing pipeline.
- **State Mutation:** Do not attempt to modify the internal maps after the object has been loaded. The configuration is designed to be immutable at runtime. Modifying it can lead to race conditions and inconsistent behavior across the server.

## Data Pipeline
The class is a terminal point in a data transformation pipeline that converts human-readable configuration into a machine-optimized format.

> Flow:
> Asset File (JSON) -> Asset Loader -> **BuilderCodec** -> **CameraEffectsConfig Instance** (String map -> Int map) -> Damage System Lookup

