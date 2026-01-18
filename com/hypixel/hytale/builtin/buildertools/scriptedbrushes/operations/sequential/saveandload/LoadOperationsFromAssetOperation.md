---
description: Architectural reference for LoadOperationsFromAssetOperation
---

# LoadOperationsFromAssetOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.saveandload
**Type:** Transient Data Object

## Definition
```java
// Signature
public class LoadOperationsFromAssetOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The LoadOperationsFromAssetOperation class is a declarative, data-driven command object within the Scripted Brush system. It does not contain executable logic itself; rather, it serves as a placeholder instruction that directs the brush execution engine to dynamically load and inline a sequence of operations from a different asset.

Its primary architectural function is to enable **composition and reusability** of brush logic. Complex sequences of operations can be defined once in a central ScriptedBrushAsset and then referenced by many other brushes, avoiding duplication and simplifying maintenance.

The static CODEC field is the most critical component. It signifies that this class is designed to be instantiated and configured via Hytale's data serialization system, not through direct Java construction. The engine deserializes a brush definition from an asset file, creating a list of operation objects. When the brush executor encounters an instance of LoadOperationsFromAssetOperation, it intercepts this specific type, reads its assetId, and modifies its own execution plan to include the operations from the referenced asset.

The empty implementation of the modifyBrushConfig method is intentional. It indicates that this is a **meta-operation**; its purpose is to alter the flow of execution, not to directly modify the world or the BrushConfig state.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the Hytale Codec system during the deserialization of a ScriptedBrushAsset from a data file (e.g., JSON). It is never intended to be created manually in game logic.
- **Scope:** The object's lifetime is bound to its parent list of operations within an in-memory BrushConfig. It is a short-lived object that exists only to be interpreted by the brush execution engine.
- **Destruction:** Eligible for garbage collection immediately after the brush execution sequence completes and the parent BrushConfig is no longer referenced.

## Internal State & Concurrency
- **State:** The internal state consists of a single String, assetId. This state is mutable during deserialization via the Codec but should be considered **effectively immutable** once the brush asset is fully loaded. It holds no runtime state or cached data.
- **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access within the context of the server's world modification loop. It must not be shared or mutated across different threads.

## API Surface
The public API is primarily for interaction with the Codec system and the brush execution engine, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | No-op. This operation is a directive for the execution engine and performs no direct action itself. |
| getAssetId() | String | O(1) | Returns the identifier of the ScriptedBrushAsset to load. |
| setAssetId(String assetId) | void | O(1) | Sets the asset identifier. Called by the Codec system during object construction. |

## Integration Patterns

### Standard Usage
This class is not used directly in Java code. Instead, it is declared within a Scripted Brush asset file. The engine interprets this declaration to load and run another asset's operations.

**Conceptual Asset Definition (e.g., JSON)**
```json
{
  "name": "My Composite Brush",
  "operations": [
    {
      "type": "hytale:set_block",
      "block": "hytale:stone"
    },
    {
      "type": "hytale:load_operations_from_asset",
      "AssetId": "hytale:common_erosion_pattern"
    },
    {
      "type": "hytale:smooth_voxels",
      "radius": 3
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new LoadOperationsFromAssetOperation()`. The resulting object will be unconfigured and will not function correctly within the engine. All instances must be created via the Codec system.
- **Manual Invocation:** Calling the `modifyBrushConfig` method on an instance will have no effect. The object must be placed in a sequence and processed by the appropriate `BrushConfigCommandExecutor`.

## Data Pipeline
This component acts as a control-flow directive within a larger data processing pipeline.

> Flow:
> ScriptedBrushAsset File -> AssetManager -> **Codec Deserialization** -> In-memory `LoadOperationsFromAssetOperation` instance -> Brush Execution Engine -> **AssetId Resolution** -> Inlining of new operations -> Continued Execution

