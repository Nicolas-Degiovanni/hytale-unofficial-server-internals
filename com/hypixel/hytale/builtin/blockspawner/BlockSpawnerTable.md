---
description: Architectural reference for BlockSpawnerTable
---

# BlockSpawnerTable

**Package:** com.hypixel.hytale.builtin.blockspawner
**Type:** Data Asset

## Definition
```java
// Signature
public class BlockSpawnerTable implements JsonAssetWithMap<String, DefaultAssetMap<String, BlockSpawnerTable>> {
```

## Architecture & Concepts
The BlockSpawnerTable is a data-driven asset that defines a weighted collection of blocks for probabilistic spawning. It does not contain any active logic; instead, it serves as a configuration file parsed by the engine, which is then consumed by systems like world generation or spawner entities.

This class is a foundational component of Hytale's **Asset System**. Its integration is primarily managed through its static **CODEC** field. This codec instructs the AssetRegistry on how to deserialize a raw data file, likely JSON, into a fully-formed BlockSpawnerTable Java object. The process includes validation to ensure that all referenced block names exist, preventing runtime errors from malformed configuration.

Once loaded, all BlockSpawnerTable instances are stored in a static, lazily-initialized **ASSET_MAP**. This provides a global, centralized registry for any game system to query for a specific table by its unique string identifier. The core data payload is the *entries* field, an IWeightedMap that allows systems to perform a weighted random selection to determine which block to spawn.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **AssetRegistry** during the engine's asset loading phase. The registry reads a corresponding data file from disk and uses the static BlockSpawnerTable.CODEC to construct the object. Manual instantiation by developers is strongly discouraged.
- **Scope:** An instance of BlockSpawnerTable, once loaded, persists for the entire application session. All loaded tables are held as strong references within the static ASSET_MAP, making them globally accessible.
- **Destruction:** Objects are eligible for garbage collection only when the AssetRegistry is cleared or reloaded, which typically occurs on server shutdown or major game state transitions. This process involves clearing the static ASSET_MAP, releasing all references.

## Internal State & Concurrency
- **State:** A BlockSpawnerTable object is **effectively immutable** after its creation by the asset pipeline. Its fields, such as *id* and *entries*, are populated during deserialization and are not intended to be modified at runtime. The public API exposes read-only access to this state.

- **Thread Safety:**
    - **Instance Safety:** Individual instances are thread-safe for read operations due to their immutable nature. Multiple threads can safely call getEntries and getId without synchronization.
    - **Static Safety:** The static getAssetMap method contains a lazy initialization block. While asset loading is typically a single-threaded, blocking operation, calling this method from multiple threads *before* assets are fully loaded could theoretically lead to a race condition.
    - **WARNING:** All systems should ensure that asset loading is complete before attempting to access any asset map from concurrent game logic threads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetMap() | static DefaultAssetMap | O(1) | Retrieves the global, read-only map of all loaded BlockSpawnerTable assets. The first call triggers a lookup in the AssetRegistry. |
| getId() | String | O(1) | Returns the unique string identifier for this asset. |
| getEntries() | IWeightedMap | O(1) | Returns the weighted map of BlockSpawnerEntry objects, which constitutes the core data of this table. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve the global asset map and then look up the desired table by its string identifier. This ensures you are using the instance managed by the engine.

```java
// How a developer should normally use this
DefaultAssetMap<String, BlockSpawnerTable> allTables = BlockSpawnerTable.getAssetMap();
BlockSpawnerTable oreTable = allTables.get("hytale:cave_ores");

if (oreTable != null) {
    // Use the IWeightedMap to perform a weighted random selection
    IWeightedMap<BlockSpawnerEntry> entries = oreTable.getEntries();
    // ... selection logic ...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BlockSpawnerTable()`. This creates an orphan object that is not registered with the engine's AssetRegistry. Game systems will be unable to find it, and it will not benefit from the asset lifecycle.
- **Static Map Modification:** Do not attempt to modify the map returned by getAssetMap. It is a shared, global resource, and modification will lead to unpredictable behavior across the entire application.
- **Pre-Initialization Access:** Do not call getAssetMap from a system that may initialize before the AssetRegistry has completed its loading phase. This can lead to race conditions or null pointer exceptions.

## Data Pipeline
The BlockSpawnerTable represents the final in-memory state of a configuration file after it has been processed by the engine's asset pipeline.

> Flow:
> JSON Data File -> AssetRegistry Loader -> **BlockSpawnerTable.CODEC** -> In-Memory BlockSpawnerTable Instance -> Stored in Static ASSET_MAP -> Queried by Game Logic

