---
description: Architectural reference for GatherObjectiveTaskAsset
---

# GatherObjectiveTaskAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Data Asset

## Definition
```java
// Signature
public class GatherObjectiveTaskAsset extends CountObjectiveTaskAsset {
```

## Architecture & Concepts
The GatherObjectiveTaskAsset is a specialized data-only class that represents a specific type of objective within Hytale's Adventure Mode system. It serves as a static data template, defining the requirements for a "gather" task, such as collecting a certain number of a specific block or item.

This class is not an active component; it does not contain logic for tracking progress. Instead, it acts as a configuration blueprint. Runtime systems, such as a player's ObjectiveTracker, consume instances of this asset to initialize and validate the state of an active quest. Its primary role is to decouple the static definition of a quest objective (authored by designers in data files) from the dynamic, stateful logic that manages a player's progress during gameplay.

It inherits from CountObjectiveTaskAsset, extending the concept of a quantity-based task with the specific requirement of a target block or item, identified by a BlockTagOrItemIdField.

### Lifecycle & Ownership
- **Creation:** Instances are exclusively created by the engine's serialization system during the asset loading phase. The static final field CODEC is the designated deserializer that constructs the object from a source data file, typically JSON.
- **Scope:** An instance of GatherObjectiveTaskAsset is immutable and persists for as long as its parent adventure content is loaded. It is typically held within a central asset registry and shared across the game.
- **Destruction:** The object is marked for garbage collection when the asset manager unloads the associated content pack or world data. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The object's state is effectively immutable after deserialization. All fields, including the target item and required count, are set upon creation and are not intended to be modified at runtime. The public API exposes read-only access to this state.
- **Thread Safety:** This class is inherently thread-safe for all read operations due to its immutable nature. Multiple game systems (e.g., UI, AI, Objective Trackers) can safely access the same instance from different threads without synchronization, as its data is guaranteed not to change.

## API Surface
The public contract is minimal, focusing on providing access to the objective's configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTaskScope() | ObjectiveTaskAsset.TaskScope | O(1) | Returns the scope of the task, defining if it is tracked per player, globally, or tied to a map marker. |
| getBlockTagOrItemIdField() | BlockTagOrItemIdField | O(1) | Retrieves the core data field specifying the block or item to be gathered. |
| matchesAsset0(ObjectiveTaskAsset) | boolean | O(1) | Internal comparison method to check for equality with another task asset. Critical for objective state management. |

## Integration Patterns

### Standard Usage
This asset is not directly instantiated or manipulated. Instead, runtime systems retrieve it from an asset or objective registry to configure a player's active quest state.

```java
// Example: A hypothetical ObjectiveManager initializing a player's task
ObjectiveConfig objective = assetRegistry.get("my_adventure.gather_wood_objective");
GatherObjectiveTaskAsset taskAsset = objective.getTask(); // Assume this is the correct type

// The manager uses the asset's data to create a stateful tracker
PlayerObjectiveState state = new PlayerObjectiveState(taskAsset.getBlockTagOrItemIdField(), taskAsset.getCount());
player.getQuestLog().add(state);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new GatherObjectiveTaskAsset()`. The engine's asset pipeline is the sole authority for creating these objects. Direct instantiation bypasses the asset registry and will lead to unmanaged, untracked, and non-functional objective definitions.
- **State Mutation:** Do not attempt to modify the internal fields of a loaded asset via reflection or other means. The immutability of these assets is a core architectural assumption. Violating it will cause unpredictable behavior and race conditions across the entire objective system.

## Data Pipeline
The GatherObjectiveTaskAsset is the result of a data transformation pipeline that begins with a human-readable design file.

> Flow:
> AdventureModeQuest.json -> Engine Asset Loader -> **BuilderCodec** -> **GatherObjectiveTaskAsset Instance** -> Objective Registry Cache -> Runtime ObjectiveManager

