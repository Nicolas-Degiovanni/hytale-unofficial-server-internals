---
description: Architectural reference for SoundSet
---

# SoundSet

**Package:** com.hypixel.hytale.server.core.asset.type.soundset.config
**Type:** Data Asset

## Definition
```java
// Signature
public class SoundSet
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, SoundSet>>,
   NetworkSerializable<com.hypixel.hytale.protocol.SoundSet> {
```

## Architecture & Concepts

The SoundSet class is a data-driven configuration object that defines a logical grouping of related sound effects. It serves as a fundamental component of the audio engine's data model, mapping abstract sound identifiers (e.g., "step", "swing") to concrete SoundEvent assets.

Its primary architectural role is to act as a bridge between human-readable JSON asset definitions and the engine's high-performance, integer-indexed runtime representation. This decoupling allows designers to configure audio behavior in simple text files, while the engine can reference and trigger sounds with minimal overhead.

The class's behavior is declaratively defined by its static **CODEC** field, an AssetBuilderCodec. This codec dictates the entire loading process:
1.  **Schema Definition:** It specifies the expected JSON structure, including fields like SoundEvents and Category.
2.  **Validation:** It enforces rules, such as ensuring that all referenced SoundEvent assets actually exist.
3.  **Hydration:** It orchestrates the post-load processing via the `afterDecode` hook, which calls the `processConfig` method.

By implementing `JsonAssetWithMap`, SoundSet integrates directly with the global AssetStore system. By implementing `NetworkSerializable`, it defines its own contract for efficient transmission to game clients.

## Lifecycle & Ownership

-   **Creation:** SoundSet instances are **never** created directly with the `new` keyword. They are exclusively instantiated by the Hytale Asset Pipeline during engine startup. The static `CODEC` field acts as a factory, parsing a corresponding JSON file and constructing the object.
-   **Scope:** An instance of SoundSet has a **global, application-wide scope**. Once loaded into the static `ASSET_STORE`, it persists for the entire lifetime of the server or client process. These objects are treated as immutable, read-only configuration data.
-   **Destruction:** Instances are not garbage collected under normal operation. They are cleared only when the entire `AssetRegistry` is shut down, typically upon application exit.

## Internal State & Concurrency

-   **State:** The state is considered **immutable after hydration**. The initial fields (`id`, `soundEventIds`, `category`) are populated from JSON. The `processConfig` method then computes the derived, performance-oriented `soundEventIndices` map. The only mutable field is `cachedPacket`, a `SoftReference` used for performance caching, which does not affect the logical state of the object.

-   **Thread Safety:** The class is **conditionally thread-safe**. All primary data fields are populated during the single-threaded asset loading phase and are not modified thereafter, making them safe for concurrent reads.

    **WARNING:** The `toPacket` method performs a non-atomic check-then-act operation on the `cachedPacket` field. If multiple threads call `toPacket` on the same instance simultaneously, it could result in the creation of redundant packet objects. This is a benign race condition but should be noted. In practice, packet creation is typically handled by a single network or game thread, mitigating this risk.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, static repository for all SoundSet assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Provides direct access to the indexed map of all loaded SoundSet assets. |
| getId() | String | O(1) | Returns the unique asset key for this SoundSet. |
| getSoundEventIds() | Map | O(1) | Returns the raw map of logical names to SoundEvent asset keys, as defined in JSON. |
| getSoundEventIndices() | Object2IntMap | O(1) | Returns the optimized map of logical names to integer indices for high-performance lookups. |
| toPacket() | com.hypixel.hytale.protocol.SoundSet | O(N) / O(1) | Converts the object to its network protocol representation. O(N) on first call, O(1) on subsequent cached calls. |

## Integration Patterns

### Standard Usage

A SoundSet should always be retrieved from the global asset map. The primary use case is to obtain the integer index for a specific sound event, which is then used by the audio engine.

```java
// How a developer should normally use this
IndexedLookupTableAssetMap<String, SoundSet> allSoundSets = SoundSet.getAssetMap();
SoundSet grassFootsteps = allSoundSets.get("hytale:player_footstep_grass");

if (grassFootsteps != null) {
    // Use the optimized integer map for performance-critical code
    int stepSoundIndex = grassFootsteps.getSoundEventIndices().getInt("step");

    // Pass this index to the audio engine
    AudioEngine.playSound(stepSoundIndex);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new SoundSet()`. This bypasses the asset pipeline, the `CODEC` definition, and the critical `processConfig` hydration step. The resulting object will be incomplete, lack its optimized index map, and will not be registered in the global store, leading to runtime failures.

-   **State Modification:** Do not attempt to modify the maps returned by `getSoundEventIds` or `getSoundEventIndices`. These are intended to be read-only configuration data. Modifying them can lead to undefined behavior across the application.

## Data Pipeline

The SoundSet class is a key stage in the audio asset data pipeline, transforming declarative configuration into a runtime-ready format.

> Flow:
> JSON Asset File -> AssetStore (via CODEC) -> **SoundSet Instance** -> `processConfig()` (Hydration & Indexing) -> `toPacket()` -> Network Protocol Object

