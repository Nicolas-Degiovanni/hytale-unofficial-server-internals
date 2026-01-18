---
description: Architectural reference for TreasureMapObjectiveTaskAsset
---

# TreasureMapObjectiveTaskAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Data Asset

## Definition
```java
// Signature
public class TreasureMapObjectiveTaskAsset extends ObjectiveTaskAsset {
```

## Architecture & Concepts

The TreasureMapObjectiveTaskAsset is a specialized configuration blueprint used by the Adventure & Objective System. It is not an active game component but rather a static data definition that describes a "treasure map" style quest task. Its primary responsibility is to define the rules for spawning and populating one or more treasure chests in the world for a player to find.

This class is designed to be deserialized from a game asset file (e.g., a JSON file). The static field **CODEC** is the cornerstone of this design, implementing the Hytale Codec system to map raw data from an asset file into a strongly-typed Java object. This pattern ensures that quest logic is data-driven, allowing content creators to design complex treasure hunts without modifying engine code.

The core of this asset is the `chestConfigs` array, which holds one or more `ChestConfig` definitions. Each `ChestConfig` specifies:
*   The spawn area for a chest, defined by a minimum and maximum radius.
*   The loot table to use for the chest's contents, referenced by a `droplistId`.
*   The specific block type for the chest, such as a `hytale:gold_chest`.
*   A `WorldLocationProvider`, which enforces complex placement rules (e.g., "must spawn underground" or "must be near water").

By extending `ObjectiveTaskAsset`, it inherits common properties like a description, completion conditions, and map markers, integrating it seamlessly into the broader objective framework.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale AssetManager during the server's asset loading phase. The static **CODEC** is invoked to deserialize a corresponding asset file from disk. **WARNING:** Manual instantiation of this class will result in an unmanaged object that is invisible to the objective system.
-   **Scope:** An instance of this asset is loaded once and cached by the AssetManager. It persists for the entire server session and is shared as a read-only template for all players undertaking this specific objective task.
-   **Destruction:** The object is eligible for garbage collection only when the AssetManager clears its cache, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency
-   **State:** This object is **effectively immutable** after creation. Its state is fully determined by the contents of its source asset file at load time. All fields are populated by the codec, and no public methods exist to modify them.
-   **Thread Safety:** The immutable nature of this class makes it inherently thread-safe. It can be safely accessed from any thread without synchronization primitives. Game logic threads, such as those handling player quest progression, can read from this asset concurrently without risk of data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTaskScope() | ObjectiveTaskAsset.TaskScope | O(1) | Returns **TaskScope.PLAYER**, indicating this task is tracked individually for each player. |
| getChestConfigs() | ChestConfig[] | O(1) | Returns the array of chest configurations that define this treasure hunt. |

## Integration Patterns

### Standard Usage

This class is not used directly in procedural code. Instead, its corresponding asset is defined in a data file. The game's objective system retrieves the pre-loaded asset from a central registry to initialize the task for a player.

```java
// Pseudo-code for how the engine might use this asset
ObjectiveTaskAsset asset = assetManager.get("my_mod:quests/find_pirate_gold_task");

if (asset instanceof TreasureMapObjectiveTaskAsset treasureTask) {
    // The engine now has the configuration to spawn chests for the player
    TreasureMapObjectiveTaskAsset.ChestConfig[] configs = treasureTask.getChestConfigs();
    worldGenerator.spawnTreasureChestsForPlayer(player, configs);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new TreasureMapObjectiveTaskAsset()`. The object must be created by the asset loading system to be correctly registered and managed.
-   **State Mutation:** Do not attempt to modify the returned `ChestConfig` array or its contents. This asset is shared globally, and any modification would create inconsistent and unpredictable behavior for all players.

## Data Pipeline

The primary flow for this class is during the asset loading process, not during real-time gameplay.

> Flow:
> JSON Asset File -> Hytale Asset Pipeline -> **TreasureMapObjectiveTaskAsset.CODEC** -> **TreasureMapObjectiveTaskAsset Instance** -> AssetManager Cache -> Objective System

---

