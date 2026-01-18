---
description: Architectural reference for BlockStateFarmingStageData
---

# BlockStateFarmingStageData

**Package:** com.hypixel.hytale.builtin.adventure.farming.config.stages
**Type:** Configuration Object

## Definition
```java
// Signature
public class BlockStateFarmingStageData extends FarmingStageData {
```

## Architecture & Concepts
BlockStateFarmingStageData is a specialized data configuration class that defines a single stage in a plant's growth cycle within the server-side farming system. It represents a world-modification command that transitions a block from its current state to a new one, for example, advancing a *Wheat Stage 1* block to *Wheat Stage 2*.

This class is a concrete implementation of the abstract FarmingStageData. Its specific responsibility is to change the block based on a named *state* defined in the block's asset configuration. The core logic resides in the `apply` method, which performs the following sequence:
1.  Identifies the original block at the target coordinates (x, y, z).
2.  Retrieves the BlockType asset for that original block.
3.  Uses the `state` string field (e.g., "fully_grown") to look up the corresponding new BlockType via the `getBlockForState` method on the original asset.
4.  If a valid new BlockType is found, it schedules a command to update the block in the WorldChunk.

This component is entirely data-driven. Instances are not created programmatically but are deserialized from asset files using the provided static `CODEC`. This allows game designers to define complex farming and growth behaviors in simple configuration files without writing any code.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the Hytale Codec system during server startup. The static `CODEC` field is used to deserialize farming asset files (e.g., JSON) into a hierarchy of configuration objects, including this one.
-   **Scope:** Instances of BlockStateFarmingStageData are owned by a parent farming asset. They are loaded into memory when the server boots and persist for the entire server session as part of the static asset data.
-   **Destruction:** De-referenced and garbage collected only when the server shuts down and the asset registries are cleared.

## Internal State & Concurrency
-   **State:** The object's state, primarily the `state` string, is intended to be immutable after deserialization. It acts as a read-only data container.
-   **Thread Safety:** This class is thread-safe. The internal state is not modified after creation. The `apply` method, which performs world modification, is carefully designed for concurrent execution. It accepts a `ComponentAccessor` and marshals the final, state-changing `worldChunk.setBlock` call to the main world thread via `world.execute()`. This pattern ensures that world data is only ever modified from its designated owner thread, preventing race conditions and data corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(commandBuffer, sectionRef, blockRef, x, y, z, previousStage) | void | O(1) | Schedules a world modification command to transition the block at the specified coordinates to the new state defined by this object. All world lookups and the final write command are handled. |
| getState() | String | O(1) | Returns the target block state name for this farming stage. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by most game logic developers. It is used by the core farming system, which manages the lifecycle of growing plants. The system would call `apply` when a plant's growth timer elapses.

```java
// Conceptual example from within a hypothetical FarmingSystem
FarmingStageData currentStage = getStageForPlant(plantEntity);

if (currentStage instanceof BlockStateFarmingStageData) {
    // The system provides the necessary world context and coordinates
    currentStage.apply(commandBuffer, sectionRef, blockRef, x, y, z, null);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new BlockStateFarmingStageData()`. The `state` field would be null, and the object would be disconnected from the asset system, causing runtime failures. All instances must be defined in configuration files and loaded by the server.
-   **External World Modification:** Do not call `apply` and then attempt to modify the same block from another system in the same tick without proper synchronization. Rely on the engine's command buffer and execution model to handle state changes.
-   **State Mutation:** Do not use reflection or other means to change the `state` field after the object has been loaded. This would violate its contract as an immutable configuration object and lead to unpredictable behavior across the server.

## Data Pipeline
The flow of data from configuration to world-effect is a key aspect of this component's design.

> Flow:
> Farming Asset File (.json) -> Server Asset Loader (using `CODEC`) -> In-Memory **BlockStateFarmingStageData** object -> Farming System Tick -> `apply()` method call -> World Command Queue -> WorldChunk block update

