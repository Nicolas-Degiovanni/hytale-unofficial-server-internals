---
description: Architectural reference for SpreadFarmingStageData
---

# SpreadFarmingStageData

**Package:** com.hypixel.hytale.builtin.adventure.farming.config.stages.spread
**Type:** Configuration Model

## Definition
```java
// Signature
public class SpreadFarmingStageData extends FarmingStageData {
```

## Architecture & Concepts
SpreadFarmingStageData is a data-driven configuration class that defines the logic for a "spread" stage within the server-side farming system. It is a concrete implementation of the abstract FarmingStageData, specializing in behaviors where a plant or fungus propagates to adjacent blocks.

This class acts as a stateless operator on the game world's state. It does not hold any mutable gameplay data itself; instead, its methods are invoked by the core farming system during a world tick. It receives references to game world components, such as FarmingBlock and ChunkSection, via a ComponentAccessor and uses its configured parameters to read and modify those components.

The primary architectural pattern is **data-driven design**. All behavior—such as the number of spread attempts, the chance of success, and the specific placement logic—is defined in external asset files (e.g., JSON) and deserialized into an instance of this class at server startup via its static CODEC. This allows designers to create complex farming behaviors without modifying engine source code.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly using the constructor. They are instantiated and configured exclusively by the Hytale codec system during the server's asset loading phase. The static field CODEC defines the deserialization and validation logic from configuration files.
- **Scope:** An instance of SpreadFarmingStageData is effectively a singleton for a given farming definition. It is loaded once at server startup and persists for the entire server session, shared by all instances of the corresponding plant or crop.
- **Destruction:** The object is garbage collected when the server shuts down and the asset registries are cleared.

## Internal State & Concurrency
- **State:** This object is **effectively immutable** after its creation by the codec system. Its fields, such as executions and spreadGrowthBehaviours, are populated once and are not modified during gameplay. It is a pure data-holder.

- **Thread Safety:** The object's internal state is read-only, making the instance itself safe to share across threads. However, its primary methods, *apply* and *shouldStop*, are **not thread-safe**. These methods operate on the game world through a ComponentAccessor, which must only be used from the main server thread responsible for the corresponding world chunk.

    **Warning:** Calling *apply* or *shouldStop* from any thread other than the designated world processing thread will lead to race conditions, data corruption, and server instability.

## API Surface
The public API is designed to be called by the central farming system, not by general gameplay logic programmers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldStop(accessor, section, block, x, y, z) | boolean | O(1) | Determines if the spread process for a specific block entity should be terminated. Considers the number of executions already performed against the configured maximum. |
| apply(accessor, section, block, x, y, z, prevStage) | void | O(N) | Executes one tick of the spread logic. Iterates through the configured SpreadGrowthBehaviour array (N) and attempts to propagate the block. Modifies the FarmingBlock component to track execution count. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most developers. It is configured in asset files and invoked automatically by the server's farming tick processor. The conceptual flow within the system is as follows.

```java
// Conceptual example of how the Farming System uses this class.
// This code does not exist in one place but represents the overall pattern.

FarmingBlock farmingComponent = command.getComponent(blockRef, FarmingBlock.getComponentType());
FarmingStageData currentStage = farmingComponent.getCurrentStage(); // This could be a SpreadFarmingStageData instance

if (currentStage instanceof SpreadFarmingStageData) {
    SpreadFarmingStageData spreadData = (SpreadFarmingStageData) currentStage;

    // Check if the stage is complete
    if (spreadData.shouldStop(command, sectionRef, blockRef, x, y, z)) {
        // Advance to the next stage...
    } else {
        // Execute the spread logic for this tick
        spreadData.apply(command, sectionRef, blockRef, x, y, z, previousStage);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SpreadFarmingStageData()`. The object will be unconfigured and bypass critical validation logic defined in its CODEC. All instances must be created via asset file deserialization.
- **Asynchronous Execution:** Do not call the *apply* method from a separate thread or asynchronous task. All interactions with the game world via the ComponentAccessor must be synchronized with the main world tick.
- **State Modification:** Do not attempt to modify the fields of a cached SpreadFarmingStageData instance at runtime. This would affect every block governed by that configuration and break the data-driven paradigm.

## Data Pipeline
The data for this object originates in a server-side configuration file and is used to manipulate component state during the game loop.

> Flow:
> Farming Asset File (JSON) -> Server Asset Loader -> **SpreadFarmingStageData.CODEC** -> In-Memory **SpreadFarmingStageData** Instance -> Farming System Tick -> ComponentAccessor -> FarmingBlock Component State Update

