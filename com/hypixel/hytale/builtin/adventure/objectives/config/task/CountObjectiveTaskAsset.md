---
description: Architectural reference for CountObjectiveTaskAsset
---

# CountObjectiveTaskAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Transient

## Definition
```java
// Signature
public abstract class CountObjectiveTaskAsset extends ObjectiveTaskAsset {
```

## Architecture & Concepts
CountObjectiveTaskAsset is an abstract base class that serves as a configuration blueprint for any in-game objective task that involves counting. It is a foundational component of the Adventure Mode objective system, designed to represent the static, definition-side of a task, not the live, in-progress state of a player's objective.

This class is not intended for direct implementation. Instead, concrete subclasses like *KillEntityObjectiveTaskAsset* or *CollectItemObjectiveTaskAsset* extend it to inherit the core counting mechanism. Its primary architectural role is to be deserialized from game data files (e.g., JSON) via the Hytale Codec system. The static **CODEC** field defines the contract for this deserialization, mapping a data key, "Count", to the internal count field and enforcing validation rules, such as ensuring the count is a positive integer.

In essence, this asset acts as a data-centric template that separates objective *design* from objective *runtime logic*.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **Codec** engine during the asset loading phase. Game designers define objectives in data files, and the engine's asset loader uses the static CODEC definition to deserialize this data into a valid CountObjectiveTaskAsset object in memory. Manual instantiation is a critical anti-pattern.
- **Scope:** An instance of this asset is loaded once and persists for the entire game session or until the associated content pack is unloaded. It is a shared, immutable template. Live game systems may hold references to it, but the asset itself is globally scoped within the asset registry.
- **Destruction:** The object is marked for garbage collection when the asset registry is cleared, typically upon server shutdown, returning to the main menu, or unloading a world.

## Internal State & Concurrency
- **State:** The internal state is effectively **immutable** post-initialization. All fields, including the inherited ones and the local *count*, are populated by the Codec system upon creation. While not explicitly declared final, modifying this state at runtime would violate the asset's contract as a static data template and lead to unpredictable behavior.
- **Thread Safety:** The class is **thread-safe for reads**. As a stateless and immutable configuration object, its data can be safely accessed by any system on any thread without requiring locks or other synchronization primitives.

**WARNING:** Any attempt to mutate the state of a loaded asset after its creation is an unsupported operation and will break system invariants.

## API Surface
The public API is minimal, focusing exclusively on retrieving the configured count value.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCount() | int | O(1) | Returns the target count required to complete this objective task. |

## Integration Patterns

### Standard Usage
This asset is not used directly in code but is defined in data files. A game system then interacts with the loaded asset to configure a live objective.

First, a game designer defines a concrete objective task in a data file:
```json
// In a file like "adventure/objectives/kill_skeletons.json"
{
  // The 'type' determines the concrete subclass that will be instantiated.
  // That subclass must extend CountObjectiveTaskAsset.
  "type": "hytale:kill_entity",

  "descriptionId": "objective.skeleton_hunter.desc",
  "mapMarkers": [ [120, 64, 350] ],

  // This "Count" key is deserialized by the CountObjectiveTaskAsset.CODEC
  "Count": 15,

  // This key would be handled by the concrete subclass's codec
  "entityType": "hytale:skeleton"
}
```

Then, an engine system retrieves the loaded asset and uses its data:
```java
// Example from an objective initialization service
ObjectiveTaskAsset asset = assetManager.get("hytale:kill_skeletons");

if (asset instanceof CountObjectiveTaskAsset) {
    // Safely access the count property via the base class API
    int requiredKills = ((CountObjectiveTaskAsset) asset).getCount(); // Returns 15

    // Use requiredKills to initialize the player's live objective state
    player.getObjectiveState().initializeTask(asset.getId(), 0, requiredKills);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new` on this class or its subclasses. The Codec system is the sole authority for its creation. Doing so bypasses validation and asset registration.
- **State Mutation:** Do not modify the `count` field after the asset is loaded. This configuration data is considered a source of truth and must remain constant.
- **Incorrect Type Assumption:** Do not assume every ObjectiveTaskAsset is a CountObjectiveTaskAsset. Always perform an `instanceof` check before casting and accessing `getCount`.

## Data Pipeline
The flow for this asset is unidirectional, from a static data definition on disk to an immutable object representation in memory.

> Flow:
> Game Data File (.json) → Asset Loading Service → Hytale Codec Engine → **CountObjectiveTaskAsset Instance** → Objective Management System → Live Player Objective State

