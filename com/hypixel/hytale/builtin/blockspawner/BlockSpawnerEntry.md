---
description: Architectural reference for BlockSpawnerEntry
---

# BlockSpawnerEntry

**Package:** com.hypixel.hytale.builtin.blockspawner
**Type:** Configuration Model

## Definition
```java
// Signature
public class BlockSpawnerEntry implements IWeightedElement {
```

## Architecture & Concepts
The BlockSpawnerEntry class is a passive data structure that represents a single, weighted outcome for a block spawning operation. It is not an active system component but rather a configuration object that defines *what* to spawn, *how* to spawn it, and the *probability* of it being chosen.

Its primary role is to serve as an entry within a larger collection managed by a spawner system. By implementing the IWeightedElement interface, it directly integrates with the engine's weighted random selection utilities. This pattern is fundamental to procedural generation, allowing designers to create diverse and controllable outcomes by defining a list of possible blocks and their relative spawn frequencies in an external asset file.

The static CODEC field is the most critical architectural feature. It acts as the bridge between serialized asset data (e.g., a JSON or BSON file) and the in-memory Java object. The engine's asset loading pipeline uses this codec to deserialize a spawner's configuration into a list of BlockSpawnerEntry instances, which are then consumed by the world generation or block placement systems.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec system during the asset loading phase. When a parent configuration asset containing a list of spawner entries is parsed, the BlockSpawnerEntry.CODEC is invoked to construct and populate each object from the source data.
- **Scope:** The object's lifetime is bound to its parent configuration asset. It is loaded into memory when the asset is required and persists as a read-only object for the duration of the game session or until the asset is explicitly unloaded.
- **Destruction:** The object is marked for garbage collection when its parent asset is unloaded from memory, typically during a world transition, server shutdown, or resource cleanup cycle.

## Internal State & Concurrency
- **State:** A BlockSpawnerEntry is effectively **immutable** after its initial creation by the codec. While its fields are not declared final, there are no public setters, and the object is not designed to be modified at runtime. It represents a static piece of configuration data.
- **Thread Safety:** The object is **thread-safe for read operations**. Due to its immutable nature post-deserialization, multiple threads (such as parallel chunk generation workers) can safely access its properties without requiring locks or other synchronization primitives.

## API Surface
The public API is designed for read-only access to the configured spawner properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlockName() | String | O(1) | Retrieves the asset name of the block to be spawned. |
| getBlockComponents() | Holder<ChunkStore> | O(1) | Returns the pre-configured component data for the block, such as inventory or state. |
| getRotationMode() | RotationMode | O(1) | Returns the rule for determining the block's orientation upon placement. |
| getWeight() | double | O(1) | Implements IWeightedElement. Returns the value used in weighted random selection. |

## Integration Patterns

### Standard Usage
A BlockSpawnerEntry is never used in isolation. It is always accessed as part of a collection from a parent spawner configuration, which then uses a weighted selection utility to choose an entry for instantiation in the world.

```java
// Conceptual example of how a spawner system would use the entries
public class BlockSpawnerSystem {
    public void spawnBlock(SpawnerConfig config) {
        // Retrieve the list of possible outcomes
        List<BlockSpawnerEntry> entries = config.getEntries();

        // Use a utility to perform a weighted random selection
        BlockSpawnerEntry chosenEntry = WeightedRandom.select(entries);

        // Use the chosen entry's data to place the block
        world.placeBlock(
            chosenEntry.getBlockName(),
            chosenEntry.getBlockComponents(),
            chosenEntry.getRotationMode()
        );
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BlockSpawnerEntry()`. The object is uninitialized and useless without being populated by the codec system. All valid instances must originate from a loaded asset file.
- **Runtime Modification:** Do not attempt to modify the state of a BlockSpawnerEntry after it has been loaded. These objects represent canonical configuration data and changing them at runtime can lead to unpredictable behavior and desynchronization.

## Data Pipeline
The BlockSpawnerEntry is the terminal object in a data deserialization pipeline. It transforms static configuration data into a usable, in-memory representation for the game engine.

> Flow:
> Spawner Asset File (JSON/BSON) -> Engine AssetLoader -> **BlockSpawnerEntry.CODEC** -> In-Memory BlockSpawnerEntry Object -> Weighted Selection Algorithm -> World Generation Logic

