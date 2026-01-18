---
description: Architectural reference for HistoryMaskOperation
---

# HistoryMaskOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.masks
**Type:** Transient / Command Object

## Definition
```java
// Signature
public class HistoryMaskOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The HistoryMaskOperation is a specific command object within the Scripted Brush framework. It does not directly modify the game world. Instead, its sole responsibility is to configure the active BrushConfig during the execution of a brush sequence.

This class acts as a state-modifier for the brush itself. It controls the *history mask*, a critical feature that determines how a brush interacts with blocks that have been previously affected during the same building session. This allows for complex, non-destructive workflows where a brush can be configured to operate only on untouched blocks (the default), only on previously modified blocks, or on all blocks regardless of their history.

Architecturally, it is a leaf node in a composite pattern, where a scripted brush is a sequence of operations. The engine iterates through these operations, and when it encounters a HistoryMaskOperation, it invokes its `modifyBrushConfig` method to alter the context for all subsequent operations in that sequence.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly via a constructor. They are instantiated by the Hytale codec system, specifically the static `CODEC` field of type `BuilderCodec`. This occurs when the server deserializes a scripted brush definition from a configuration file or network data.
-   **Scope:** The object's lifetime is extremely short and is scoped to a single execution of a scripted brush. It is created, its configuration method is called once, and it is then eligible for garbage collection.
-   **Destruction:** Managed by the Java garbage collector. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
-   **State:** The class holds one primary piece of mutable state: the `historyMaskArg` field. This field is populated by the codec during deserialization and determines which mask setting will be applied to the `BrushConfig`. After initialization, this state is effectively read-only for the remainder of its lifecycle.
-   **Thread Safety:** This class is **not thread-safe**. The public `historyMaskArg` field can be modified externally. However, the surrounding Scripted Brush system is designed for sequential, single-threaded execution on the server's main thread. Concurrency issues are avoided at an architectural level by ensuring that brush operations are processed in a strict, non-parallel sequence.

## API Surface
The public contract is minimal, centered on its role as a step in a larger process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | Applies the configured history mask to the provided BrushConfig object. This is the primary entry point called by the brush execution engine. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code by developers. It is designed to be declared within a data format (like JSON or HOCON) that defines a scripted brush. The engine then deserializes this data into a sequence of operation objects.

A conceptual script definition might look like this:

```json
// Conceptual Example: How this operation is defined in data
"operations": [
  {
    "type": "HistoryMask",
    "HistoryMask": "HistoryOnly"
  },
  {
    "type": "Paint",
    "block": "hytale:stone"
  }
]
```
In this sequence, the brush is first configured to only affect blocks it has previously touched, and then a paint operation is performed.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new HistoryMaskOperation()`. The object will be in an invalid state as its `historyMaskArg` will not be properly configured. Always rely on the engine's codec system for instantiation.
-   **External State Mutation:** Do not modify the public `historyMaskArg` field after the object has been created by the codec. This can lead to unpredictable brush behavior.
-   **Manual Invocation:** Calling `modifyBrushConfig` outside the scripted brush execution loop is unsupported. The method requires a valid `BrushConfig` and entity context that are only provided by the engine during brush execution.

## Data Pipeline
HistoryMaskOperation acts as a control-flow component rather than a data-processing one. It alters the configuration that governs subsequent steps in the pipeline.

> Flow:
> Brush Script Definition (Data) -> Engine Codec Deserialization -> **HistoryMaskOperation Instance** -> Brush Executor invokes `modifyBrushConfig` -> BrushConfig State is updated -> Subsequent World-Modifying Operations (e.g., PaintOperation) execute using the new history mask.

