---
description: Architectural reference for PastePrefabOperation
---

# PastePrefabOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient

## Definition
```java
// Signature
public class PastePrefabOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The **PastePrefabOperation** is a stateful, single-execution component within the Scripted Brush framework. It serves as a concrete step in a sequence of world modification actions, designed to place a pre-designed structure, or Prefab, into the game world.

This operation acts as a bridge between the high-level asset system and the low-level world data layer. It is configured with a reference to a **PrefabListAsset**, from which it randomly selects a specific prefab file path. Upon execution, it loads the prefab data into a **PrefabBuffer**, a memory-efficient representation of the structure. It then iterates through this buffer, directly manipulating **WorldChunk** and **ChunkStore** components to place blocks, set block states, and manage fluids.

Its core design principle is "execute-once". A critical internal flag, *hasBeenPlacedAlready*, ensures that a single application of a brush (e.g., a single mouse click) only pastes the prefab one time, even if the brush's logic would otherwise invoke this operation multiple times. This makes it a predictable and non-destructive part of a larger, sequential operation chain.

## Lifecycle & Ownership
- **Creation:** Instances of **PastePrefabOperation** are not created directly via a constructor in gameplay code. They are deserialized and instantiated by the engine's **Codec** system, specifically through the static **CODEC** field. This typically occurs when the server loads a **BrushConfig** asset from storage.

- **Scope:** The object's lifetime is bound to the **BrushConfig** that contains it. It persists as a configured operation within the brush definition. However, its internal state is transient and relevant only to a single application of the brush. The **resetInternalState** method is invoked by the brush system between uses to prepare it for the next execution.

- **Destruction:** The object is eligible for garbage collection when its parent **BrushConfig** is unloaded or discarded. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** This class is mutable and stateful. Its primary state is the boolean flag *hasBeenPlacedAlready*, which tracks whether the operation has been executed within the current sequence. This flag is fundamental to its single-execution behavior. The *prefabListAssetId* is configured at creation and is treated as immutable thereafter.

- **Thread Safety:** **This class is not thread-safe.** The **modifyBlocks** method performs direct, unsynchronized writes to low-level world data structures like **WorldChunk** and **ChunkStore**. All interactions with this class, particularly the invocation of **modifyBlocks**, must occur on the primary world-update thread to prevent catastrophic world corruption, race conditions, and chunk data inconsistencies.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resetInternalState() | void | O(1) | Resets the internal execution flag, allowing the operation to run again on the next brush application. |
| modifyBlocks(...) | boolean | O(N) | Executes the core logic. Loads and pastes a prefab into the world. N is the number of blocks in the prefab. Returns false to signify the operation is complete. |

## Integration Patterns

### Standard Usage
This operation is not intended to be instantiated or called directly in code. It is designed to be defined declaratively within a brush configuration asset file (e.g., a JSON file). The engine's brush system then loads this configuration and manages the operation's lifecycle.

A conceptual configuration might look like this:

```json
// Example BrushConfig.json
{
  "name": "ForestryBrush",
  "operations": [
    {
      "type": "PastePrefabOperation",
      "PrefabListAssetName": "hytale:trees_oak"
    },
    {
      "type": "AnotherOperation",
      "...": "..."
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PastePrefabOperation()`. This bypasses the asset loading mechanism, leaving the *prefabListAssetId* null and rendering the operation non-functional. It must be configured and loaded via the engine's asset pipeline.

- **State Mismanagement:** Calling **modifyBlocks** multiple times within a single brush stroke without an intermediate call to **resetInternalState** will have no effect after the first successful execution. This behavior is by design.

- **Concurrent Execution:** Never invoke **modifyBlocks** from a worker thread. All world modifications must be marshaled to the main world thread to prevent data corruption.

## Data Pipeline
The data flow for this operation involves transforming a high-level asset reference into low-level world modifications.

> Flow:
> BrushConfig Asset (JSON) -> **CODEC Deserializer** -> In-memory **PastePrefabOperation** instance -> Brush Execution Trigger -> **PrefabListAsset** lookup -> Prefab File Path -> **PrefabBuffer** load -> Iteration and direct write to **ChunkStore** / **WorldChunk** data.

