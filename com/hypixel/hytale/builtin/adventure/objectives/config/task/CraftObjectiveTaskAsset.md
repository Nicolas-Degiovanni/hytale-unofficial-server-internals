---
description: Architectural reference for CraftObjectiveTaskAsset
---

# CraftObjectiveTaskAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Asset / Data Model

## Definition
```java
// Signature
public class CraftObjectiveTaskAsset extends CountObjectiveTaskAsset {
```

## Architecture & Concepts
The CraftObjectiveTaskAsset is a specialized, data-driven configuration model within Hytale's Adventure Mode objective system. It does not represent a live, running task but rather the static *definition* of a task that requires a player to craft a specific item a certain number of times.

This class acts as a concrete implementation within a larger inheritance hierarchy of objective tasks. It extends CountObjectiveTaskAsset, inheriting the core concept of a target count, and specializes it by introducing an **itemId**.

Its most critical architectural feature is the static **CODEC** field. This integrates the class directly into Hytale's asset serialization and loading pipeline. Game designers define crafting objectives in external data files (e.g., JSON), and the engine uses this codec to deserialize that data into an immutable in-memory instance of CraftObjectiveTaskAsset. The codec also enforces data integrity through validators, ensuring that the specified item ID is valid before the asset is even loaded.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale asset loading system during server or client bootstrap. The static CODEC field is invoked to deserialize a corresponding entry from an adventure mode configuration file. Manual instantiation is a design violation.
- **Scope:** An instance of CraftObjectiveTaskAsset is effectively a singleton definition for a specific objective. Once loaded, it persists in the central asset registry for the entire server or client session. It is an immutable, shared resource referenced by any quest or objective that uses this task.
- **Destruction:** The object is marked for garbage collection only when the asset registry is cleared, typically during a full server shutdown or a comprehensive asset hot-reload.

## Internal State & Concurrency
- **State:** **Immutable**. All fields, including the inherited count and the local itemId, are populated once during deserialization. The object's state is not intended to be mutated at runtime. It serves as a read-only template.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, an instance of CraftObjectiveTaskAsset can be safely read and accessed by multiple threads simultaneously without any need for locks or other synchronization primitives. This is crucial for systems where game logic, networking, and UI threads may all need to reference objective definitions.

## API Surface
The public API is minimal, focusing on data retrieval. It is designed to be consumed by higher-level objective management systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTaskScope() | ObjectiveTaskAsset.TaskScope | O(1) | Returns the scope of the task, indicating it applies to a specific player and may have associated map markers. |
| getItemId() | String | O(1) | Retrieves the unique identifier for the item that must be crafted to progress the task. |

## Integration Patterns

### Standard Usage
This asset is not used directly. Instead, it is held as a configuration property by a live objective tracker or state machine. The system retrieves the asset from a registry and uses its data to initialize and validate player actions.

```java
// Pseudo-code demonstrating consumption by an objective system
// Note: You do not create this asset; you retrieve its definition.

ObjectiveDefinition objectiveDef = assetRegistry.get("my_adventure_quest.craft_iron_sword_task");
CraftObjectiveTaskAsset taskAsset = (CraftObjectiveTaskAsset) objectiveDef.getTask();

// The system now uses the asset's data to check player events
void onPlayerCraftEvent(Player player, Item craftedItem) {
    if (craftedItem.getId().equals(taskAsset.getItemId())) {
        player.getQuestState("my_adventure_quest").incrementTaskProgress();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new CraftObjectiveTaskAsset()`. Doing so bypasses the asset pipeline, validation, and caching. All objective definitions must originate from data files to ensure consistency and allow for content modification without code changes.
- **State Modification:** Do not attempt to modify the asset's fields at runtime via reflection or other means. This asset represents canonical game design data, not a player's current progress. Modifying it would affect every player and system referencing this objective definition, leading to catastrophic state corruption.

## Data Pipeline
The CraftObjectiveTaskAsset is the terminal product of a data deserialization pipeline. It transforms static configuration into a usable in-memory object for the game engine.

> Flow:
> Adventure Quest JSON File -> Hytale Asset Loader -> **CraftObjectiveTaskAsset.CODEC** -> **CraftObjectiveTaskAsset Instance** -> Objective Management System -> Live Player Quest Tracker

