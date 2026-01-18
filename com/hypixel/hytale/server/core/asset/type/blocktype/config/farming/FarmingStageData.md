---
description: Architectural reference for FarmingStageData
---

# FarmingStageData

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.farming
**Type:** Configuration Data Model

## Definition
```java
// Signature
public abstract class FarmingStageData {
```

## Architecture & Concepts
FarmingStageData is an abstract base class that serves as a data-driven definition for a single stage within a multi-stage farming or growth process. It is a fundamental component of the server's block behavior system, allowing complex, sequential state changes to be defined entirely within asset configuration files rather than hard-coded logic.

This class and its concrete implementations are not instantiated directly in game logic. Instead, they are deserialized from asset files by the Hytale **Codec** system. The static CODEC field, a CodecMapCodec, acts as a factory that reads a "Type" field from the data source to determine which concrete subclass (e.g., a growth stage, a harvest stage) to instantiate.

Architecturally, FarmingStageData represents a single node in a state machine. A parent configuration, such as a block type asset, will contain an ordered list of these stages. A game system, such as the block ticking system, transitions a block from one stage to the next by invoking the logic defined within the corresponding FarmingStageData object.

**WARNING:** This object is a shared data definition. A single instance is referenced by all blocks of a given type. It must not contain any per-block instance state.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec framework during the server's asset loading phase. The static CODEC and BASE_CODEC fields define the deserialization contract from a data source like JSON or HOCON.
- **Scope:** The object's lifetime is tied to its parent asset (e.g., a BlockType). It persists in memory as long as that asset is loaded, typically for the entire duration of a server session.
- **Destruction:** The object is eligible for garbage collection when its parent asset is unloaded, which may occur during a server shutdown or a full resource reload. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The state of a FarmingStageData object is considered **immutable** after its initial creation and deserialization. Fields like duration and soundEventId are loaded once and never change. The transient field soundEventIndex is a post-deserialization cache to optimize runtime lookups and is also immutable after its initial calculation.
- **Thread Safety:** The object itself is inherently thread-safe for read operations due to its immutable nature. Methods that mutate world state, such as **apply**, are designed to be called from various worker threads. Thread safety for world modifications is not the responsibility of this class; it is enforced by the **ComponentAccessor** and command buffer system passed into these methods. All world interactions are funneled through this accessor, ensuring deterministic and safe state changes.

## API Surface
The public API provides hooks for a managing system to execute the stage's logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(...) | void | O(1) | Executes the primary logic of this stage, such as playing a sound or triggering a block update. Mediates all world changes through the provided ComponentAccessor. |
| shouldStop(...) | boolean | O(1) | A predicate to determine if the stage progression should halt based on world conditions. The base implementation always returns false. |
| remove(...) | void | O(1) | A cleanup hook called when a block transitions *away* from this stage. Intended for removing temporary components or state. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most game logic developers. It is invoked by core engine systems that manage block state transitions. A system would retrieve the relevant stage data from a block's type definition and execute its apply method.

```java
// Conceptual example within a block update system
BlockType type = world.getBlockType(position);
FarmingConfig farmingConfig = type.getFarmingConfig();
FarmingStageData currentStage = farmingConfig.getStage(stageIndex);

// The system passes a command buffer accessor to ensure thread safety
currentStage.apply(commandBuffer, sectionRef, blockRef, x, y, z, previousStage);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate a subclass of FarmingStageData with the new keyword. The entire system relies on data-driven instantiation via the Codec system to function correctly.
- **State Mutation:** Do not attempt to modify the fields of a FarmingStageData object after it has been loaded. As this is shared configuration data, any runtime modification will cause unpredictable behavior across all blocks of that type.
- **World Access without Accessor:** Do not attempt to access or modify world state directly from within an implementation. All world interactions must be routed through the provided ComponentAccessor to guarantee thread safety and determinism.

## Data Pipeline
The data for this object originates in an asset file and is used by game systems to enact changes in the world state.

> Flow:
> Asset File (JSON/HOCON) -> Asset Loading System -> **Codec Deserializer** -> In-Memory **FarmingStageData** Instance -> Block Ticking System -> apply() -> ComponentAccessor -> World State Change

