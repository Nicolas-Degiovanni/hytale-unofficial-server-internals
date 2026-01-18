---
description: Architectural reference for BlockTypeFarmingStageData
---

# BlockTypeFarmingStageData

**Package:** com.hypixel.hytale.builtin.adventure.farming.config.stages
**Type:** Configuration Object

## Definition
```java
// Signature
public class BlockTypeFarmingStageData extends FarmingStageData {
```

## Architecture & Concepts
BlockTypeFarmingStageData is a data-driven configuration object that represents a discrete stage in a plant's growth cycle. It is a concrete implementation of the abstract FarmingStageData, specializing in the action of transforming the plant's location into a specific, predefined block.

This class is a cornerstone of Hytale's data-driven farming system. Instead of hard-coding plant growth logic, developers define a series of stages in external configuration files (e.g., JSON). The engine's asset loading system uses the static CODEC field to deserialize these definitions into a collection of FarmingStageData objects.

The core responsibility of this class is encapsulated in the **apply** method. When the farming system determines a plant should advance to this stage, it invokes **apply**, which in turn schedules a command to change the block in the world to the one specified in the configuration. This decouples the game logic (when to grow) from the game data (what it grows into).

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the engine's serialization system via the static **CODEC** field during server startup or asset loading. The codec reads a configuration file and uses the provided builder (`BlockTypeFarmingStageData::new`) to instantiate the object, populating its fields from the file's contents.

- **Scope:** An instance of BlockTypeFarmingStageData has a static lifetime. It is loaded once when the server initializes its game assets and persists in memory for the entire server session, held by its parent farming asset. It is effectively an immutable singleton within the context of a specific farming definition.

- **Destruction:** The object is eligible for garbage collection only when the server shuts down or initiates a full asset reload. There is no explicit destruction or cleanup mechanism.

## Internal State & Concurrency
- **State:** The internal state consists of a single field, **block**, which holds the string identifier for the target BlockType. This state is **immutable** after its initial creation by the codec. It serves as read-only configuration data.

- **Thread Safety:** The object is inherently thread-safe for read operations due to its immutable state. However, the **apply** method performs a world-modifying write operation.

    **WARNING:** The **apply** method is not re-entrant and is unsafe to call from arbitrary threads. It manipulates world components like WorldChunk and ChunkSection. The implementation correctly handles this by using `world.execute(...)` to schedule the block placement onto the main world simulation thread, thus ensuring thread safety and preventing race conditions within the world state. Callers must provide a valid and thread-appropriate ComponentAccessor.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(...) | void | O(1) (Scheduled) | Schedules a command to set the block at the specified coordinates. The operation itself is deferred to the main world thread. |
| getBlock() | String | O(1) | Returns the configured block identifier string. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or manipulation by game logic developers. It is configured in asset files and consumed by the core farming system. The system retrieves the current stage for a crop and executes it.

```java
// Pseudo-code demonstrating how the engine's FarmingSystem would use this object.
// A developer would NOT write this code.

FarmingStageData currentStage = getStageForCrop(cropEntity);

// The system provides the necessary world context and coordinates
currentStage.apply(commandBuffer, sectionRef, blockRef, x, y, z, previousStage);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new BlockTypeFarmingStageData()`. This bypasses the codec, leaving the internal **block** field null. A subsequent call to **apply** will fail when looking up the block index, likely causing a NullPointerException or an engine error.

- **Asynchronous Execution:** Do not call the **apply** method from a thread that does not have ownership of the provided ComponentAccessor or world context. Doing so will bypass the engine's threading model and lead to world corruption, deadlocks, or crashes.

## Data Pipeline
The class functions as a data container and an execution command, sitting between asset configuration and world simulation.

> **Asset Loading Flow:**
> Farming Asset File (JSON) -> AssetManager -> **BlockTypeFarmingStageData.CODEC** -> In-Memory **BlockTypeFarmingStageData** Instance

> **Gameplay Logic Flow:**
> Farming System Tick -> Growth Condition Met -> **BlockTypeFarmingStageData.apply()** -> World Command Queue -> WorldChunk.setBlock() -> Block Update Propagates to Clients

