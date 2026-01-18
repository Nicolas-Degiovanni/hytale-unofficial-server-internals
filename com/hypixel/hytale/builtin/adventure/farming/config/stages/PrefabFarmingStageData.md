---
description: Architectural reference for PrefabFarmingStageData
---

# PrefabFarmingStageData

**Package:** com.hypixel.hytale.builtin.adventure.farming.config.stages
**Type:** Data Object

## Definition
```java
// Signature
public class PrefabFarmingStageData extends FarmingStageData {
```

## Architecture & Concepts
The PrefabFarmingStageData class is a specialized implementation of a farming growth stage. Where the base FarmingStageData typically modifies a single block, this class orchestrates the placement of a complex, multi-block structure known as a Prefab into the game world. It serves as the bridge between the server's declarative farming configuration and the procedural world modification system.

This component is central to creating dynamic and visually complex agriculture, such as trees that grow from a sapling into a full, varied structure, or large crops that occupy multiple blocks. It is designed to be configured entirely through asset files, allowing designers to define sophisticated growth behaviors without writing code.

Key architectural concepts include:

*   **Declarative Configuration:** The behavior of this class is defined by data, not code. A static CODEC is responsible for deserializing asset files into immutable PrefabFarmingStageData instances at server startup.
*   **Weighted Prefab Selection:** To introduce natural variation, the class uses a weighted map of potential prefabs. For each growth event, a prefab is chosen randomly based on its configured weight, allowing the same plant type to grow into different shapes.
*   **Transactional World Updates:** The core `apply` method does not modify the world directly. Instead, it performs checks for obstruction and integrity and then schedules a command for execution on the main world thread via `world.execute`. This pattern ensures that all world modifications are thread-safe and atomic from the perspective of other game systems.
*   **State Transition Logic:** The class is aware of previous growth stages. When transitioning from a prior PrefabFarmingStageData, it first verifies that the old prefab is still intact before placing the new one. This prevents growth if the plant has been partially destroyed by a player or explosion.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale `CODEC` system during server boot. The server scans asset packs for farming configuration files (e.g., JSON), and the `PrefabFarmingStageData.CODEC` deserializes these definitions into Java objects.
-   **Scope:** An instance of this class is a stateless configuration object. Once loaded, it persists for the entire lifetime of the server. It is effectively a singleton from the perspective of a specific farming definition.
-   **Destruction:** Instances are garbage collected when the server shuts down and the AssetRegistry is cleared. They are never destroyed during normal gameplay.

## Internal State & Concurrency
-   **State:** The object's state is **immutable** after its initial creation by the codec. Fields such as `prefabStages` and `replaceMaskTags` are populated once during deserialization and are not modified thereafter. The class holds no runtime state related to any specific plant in the world.
-   **Thread Safety:** The class is thread-safe for read operations due to its immutable state. The primary methods, `apply` and `remove`, which perform world modifications, are designed to be called from any thread (such as a chunk-ticking worker thread). They achieve safety by queuing their block-placement logic onto the main world thread's execution queue. This delegation makes the *effects* of the methods thread-safe, even if the initial call is not on the main thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(...) | void | O(V) | Orchestrates the placement of a prefab into the world. Complexity is proportional to the volume (V) of the prefabs being checked and placed. This is a high-cost operation involving chunk access and scheduling tasks. |
| remove(...) | void | O(V) | Schedules the removal of a prefab structure from the world, replacing it with air or other blocks as defined. Complexity is proportional to the prefab volume (V). |
| getPrefabStages() | IWeightedMap | O(1) | Returns the configured weighted map of possible prefabs for this stage. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by game logic developers. It is configured within an asset file and used polymorphically by the core farming system. The system retrieves the appropriate stage data for a growing block and calls `apply`.

A conceptual view of how the *system* uses it:

```java
// This logic resides deep within the server's farming component system.
// A developer would NOT write this code.

FarmingBlock farmingComponent = ...;
FarmingStageData currentStage = farmingComponent.getCurrentStageData();

if (currentStage instanceof PrefabFarmingStageData) {
    // The system invokes apply with the correct world context.
    currentStage.apply(commandBuffer, sectionRef, blockRef, x, y, z, previousStage);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new PrefabFarmingStageData()`. The object is uninitialized and will fail. It must be created from a configuration file via the engine's asset loader.
-   **Manual Invocation:** Do not call `apply` or `remove` from custom game logic. These methods require a highly specific `ComponentAccessor` context that is only available during the engine's internal block update cycle. Calling them manually will lead to exceptions or world corruption.
-   **State Modification:** Do not attempt to modify the `prefabStages` map or other fields after the object has been loaded. These are treated as immutable configuration.

## Data Pipeline
The execution of this class is the final step in a long chain of events, beginning with server configuration and ending with world modification.

> Flow:
> Server Startup -> Asset Loader parses farming JSON -> **CODEC creates PrefabFarmingStageData instance** -> Instance stored in AssetRegistry -> Game Tick on a FarmingBlock -> Growth logic triggers -> System retrieves stage instance -> System calls **PrefabFarmingStageData.apply()** -> Method schedules task on World thread -> Prefab blocks are placed in World Chunks

