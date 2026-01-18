---
description: Architectural reference for TagSet
---

# TagSet

**Package:** com.hypixel.hytale.builtin.tagset
**Type:** Data Contract

## Definition
```java
// Signature
public interface TagSet extends JsonAsset<String> {
```

## Architecture & Concepts
The TagSet interface is a fundamental data contract within the asset and content management system. It provides a standardized model for defining complex collections of tags used for filtering, categorization, and rule evaluation. By extending JsonAsset, it signals that all TagSet definitions are loaded directly from JSON files as part of the engine's asset pipeline.

This interface is not a system in itself, but rather the data schema used by higher-level systems such as biome generation, loot table resolution, and quest objective validation. Its primary architectural function is to decouple the *definition* of a rule set (the TagSet) from the *evaluation* of that rule set (the game logic).

The design supports compositional complexity by allowing TagSets to include or exclude not only individual tags but also entire other TagSets. This enables content creators to build hierarchical and highly reusable filtering logic without code changes. For example, a master *HostileCreatures* TagSet could include the *Undead* and *Monsters* TagSets while excluding the *Bosses* TagSet.

## Lifecycle & Ownership
As an interface, TagSet itself has no lifecycle. The following pertains to the concrete objects that implement this contract, which are managed by the asset system.

- **Creation:** TagSet objects are instantiated and deserialized from JSON files by the AssetManager during the engine's asset loading phase. They are never created programmatically during active gameplay.
- **Scope:** Once loaded, a TagSet object is considered an immutable, shared asset. It is cached by the AssetManager and persists for the entire duration of the client or server session.
- **Destruction:** All TagSet instances are de-referenced and eligible for garbage collection when the AssetManager is shut down, typically upon exiting a world or closing the application.

## Internal State & Concurrency
- **State:** The contract implies an **immutable** state. Once a TagSet is loaded from its source JSON, its internal lists of included and excluded tags are fixed and cannot be altered at runtime. This guarantees deterministic behavior for any system that references it.
- **Thread Safety:** Implementations are expected to be **fully thread-safe for reads**. The immutable nature of the underlying data means that any thread can safely call the accessor methods at any time without synchronization or locking. This is critical for performance in multi-threaded systems like parallel world generation or asset pre-loading.

## API Surface
The public contract consists entirely of accessors for the predefined inclusion and exclusion rules.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIncludedTagSets() | String[] | O(1) | Returns the names of other TagSets whose tags should be included. |
| getExcludedTagSets() | String[] | O(1) | Returns the names of other TagSets whose tags should be excluded. |
| getIncludedTags() | String[] | O(1) | Returns the list of individual tags to be included. |
| getExcludedTags() | String[] | O(1) | Returns the list of individual tags to be excluded. |

## Integration Patterns

### Standard Usage
The primary pattern involves retrieving a pre-defined TagSet from the AssetManager and passing it to a system that performs evaluation logic. The TagSet itself does not contain the logic to check if an object matches its criteria.

```java
// A game system retrieves a TagSet to determine valid spawns
AssetManager assets = context.getService(AssetManager.class);
TagSet validSpawns = assets.get("hytale:biome.forest.day_spawns");

// The evaluation logic resides elsewhere, consuming the TagSet data
boolean canSpawn = SpawnSystem.evaluate(entity, validSpawns);
```

### Anti-Patterns (Do NOT do this)
- **Runtime Modification:** Do not attempt to cast a TagSet to a concrete class to modify its internal lists. This violates the immutable asset contract and will lead to unpredictable behavior across the engine.
- **Manual Instantiation:** Never create an ad-hoc implementation of TagSet at runtime. The entire system is designed around definitions loaded from source JSON files, ensuring content can be managed and validated by the toolchain.

## Data Pipeline
The TagSet is a pure data object whose lifecycle is managed by the asset pipeline.

> Flow:
> Content Creator defines `my_tags.json` -> Asset Pipeline processes and validates JSON -> **AssetManager** deserializes JSON into a **TagSet** object -> Game System (e.g., LootManager) requests the TagSet by name -> The system uses the TagSet's data for filtering logic.

