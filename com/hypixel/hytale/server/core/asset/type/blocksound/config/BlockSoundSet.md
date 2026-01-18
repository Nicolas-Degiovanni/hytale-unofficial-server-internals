---
description: Architectural reference for BlockSoundSet
---

# BlockSoundSet

**Package:** com.hypixel.hytale.server.core.asset.type.blocksound.config
**Type:** Asset Model

## Definition
```java
// Signature
public class BlockSoundSet
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, BlockSoundSet>>,
   NetworkSerializable<com.hypixel.hytale.protocol.BlockSoundSet> {
```

## Architecture & Concepts

The BlockSoundSet class is a server-side data model that represents a complete collection of sounds associated with a specific block type, such as *stone*, *wood*, or *glass*. It acts as a configuration object, loaded from JSON asset files at server startup.

Its primary architectural role is to serve as a translation layer between human-readable asset identifiers and engine-optimized integer indices. During the asset loading pipeline, it consumes a map of sound event types (e.g., BREAK, PLACE, STEP) to their corresponding SoundEvent asset string keys (e.g., "hytale:sound.block.stone.break"). It then resolves these string keys into compact integer indices by looking them up in the global SoundEvent asset registry.

This pre-computation is critical for performance. The game engine and network protocol can then refer to sounds using these efficient integer indices rather than performing expensive string comparisons or lookups during gameplay.

Furthermore, by implementing NetworkSerializable, this class defines the contract for synchronizing block sound configurations with connected game clients. It contains logic to transform itself into a lightweight network packet, ensuring clients have the necessary data to play the correct sounds.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale Asset System during the server's bootstrap phase. The static CODEC field defines the deserialization logic, which is invoked by the AssetStore when it parses the corresponding JSON configuration files. The lifecycle hook *afterDecode* ensures the critical `processConfig` method is called immediately after an instance is created from data.
-   **Scope:** Application-scoped. Once loaded, all BlockSoundSet instances are stored in a static AssetStore and persist for the entire lifetime of the server session. They are treated as immutable configuration data.
-   **Destruction:** Instances are dereferenced and eligible for garbage collection only when the server shuts down and the central AssetRegistry is cleared.

## Internal State & Concurrency

-   **State:** The object is effectively immutable after initialization. The core data fields, *id*, *soundEventIds*, and the derived *soundEventIndices*, are set once during the asset loading process and are not intended to be modified thereafter. The only mutable state is the *cachedPacket* field, which is a `SoftReference` used for performance optimization.
-   **Thread Safety:** This class is **conditionally thread-safe**. All primary data fields are populated during the single-threaded asset loading phase and can be safely read by multiple threads. The `toPacket` method, however, performs a non-atomic check-then-act operation on the *cachedPacket* field. If multiple threads call `toPacket` on the same instance concurrently, it may result in the creation of redundant packet objects. This is a benign race condition, but it is a consideration in highly concurrent environments. In standard engine operation, this is not a practical concern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.BlockSoundSet | O(1) amortized | Converts the object into its network packet representation. Uses a soft-referenced cache for high performance. |
| getId() | String | O(1) | Returns the unique string identifier for this asset (e.g., "hytale:wood"). |
| getSoundEventIndices() | Object2IntMap | O(1) | Returns the optimized map of BlockSoundEvent enums to their integer asset indices. This is the primary API used by game logic. |
| getAssetStore() | static AssetStore | O(1) | Provides access to the global registry containing all loaded BlockSoundSet assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | A convenience method to get the underlying map structure from the AssetStore for direct lookups. |

## Integration Patterns

### Standard Usage

A developer should never instantiate this class directly. It should always be retrieved from the central asset map using its known string identifier. The primary use case is to get the integer index for a specific sound event to be played by the engine.

```java
// Retrieve the central map of all block sound sets
IndexedLookupTableAssetMap<String, BlockSoundSet> map = BlockSoundSet.getAssetMap();

// Get the specific sound set for "stone"
BlockSoundSet stoneSounds = map.get("hytale:stone");

if (stoneSounds != null) {
    // Get the integer index for the 'BREAK' sound event
    // This index is what the audio engine or network layer will use
    int breakSoundIndex = stoneSounds.getSoundEventIndices().getInt(BlockSoundEvent.BREAK);
    
    // Trigger sound playback using the resolved index
    AudioEngine.playSoundByIndex(breakSoundIndex);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new BlockSoundSet()`. Instances created this way will be incomplete and malformed, as they bypass the asset loading pipeline and the crucial `processConfig` initialization step. This will result in an empty `soundEventIndices` map and will cause runtime errors.
-   **State Modification:** Do not attempt to modify the maps returned by `getSoundEventIds` or `getSoundEventIndices`. The engine relies on this data being immutable after the initial loading phase. Modifying it can lead to desynchronization and unpredictable behavior.

## Data Pipeline

The flow of data from configuration to engine usage is a key aspect of this class's design. It transforms data from a human-readable format to a machine-optimized one.

> Flow:
> 1.  **JSON Asset File** (e.g., `stone.json` on disk)
>     - Defines `SoundEvents` as a map of event names to string asset keys.
> 2.  **AssetStore Loading**
>     - Reads the JSON file from disk.
> 3.  **BlockSoundSet.CODEC**
>     - Deserializes the JSON data into a new `BlockSoundSet` object in memory.
> 4.  **processConfig() method**
>     - Iterates over the `soundEventIds` map.
>     - For each string key, it queries the `SoundEvent.getAssetMap()` to resolve the string into its corresponding integer index.
>     - Populates the internal `soundEventIndices` map with this optimized data.
> 5.  **BlockSoundSet Instance**
>     - The fully initialized and processed object is now stored in the `BlockSoundSet.ASSET_STORE`.
> 6.  **Game Logic** (e.g., a block is broken)
>     - Retrieves the `BlockSoundSet` and calls `getSoundEventIndices()` to get the integer index for the `BREAK` event.
> 7.  **Audio Engine / Network Layer**
>     - Uses the final integer index to play the sound or to notify clients.

