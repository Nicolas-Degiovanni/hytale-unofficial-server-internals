---
description: Architectural reference for MaskOperation
---

# MaskOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.masks
**Type:** Transient / Command Object

## Definition
```java
// Signature
public class MaskOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The MaskOperation is a concrete implementation of the **Command Pattern**, designed to function within a sequential brush execution pipeline. Its sole responsibility is to modify the state of a shared `BrushConfig` context object by setting an operation mask.

It does not perform any world modification itself. Instead, it acts as a state-mutating filter that influences the behavior of subsequent operations in the same execution sequence. For example, if a `PlaceBlockOperation` follows a `MaskOperation`, the `PlaceBlockOperation` will only affect blocks that match the mask defined by this operation.

This class is data-driven, relying on its static `CODEC` for serialization and deserialization. This allows complex brush behaviors, including masking, to be defined in external configuration files and loaded by the engine at runtime.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale codec system during the deserialization of a scripted brush definition. The static `CODEC` field is the entry point for this process, which instantiates the class and populates the `operationMaskArg` field from the source data. Direct instantiation via `new MaskOperation()` is rare and generally discouraged.
- **Scope:** The object's lifetime is extremely short. It exists only for the duration of a single brush application sequence. It is created, its `modifyBrushConfig` method is invoked, and it is then eligible for garbage collection. It holds no persistent state beyond the execution cycle.
- **Destruction:** Cleanup is handled by the standard Java garbage collector. No native resources are held, and no explicit destruction method is required.

## Internal State & Concurrency
- **State:** The primary state is held in the `operationMaskArg` field, which is an instance of `BlockMask`. This field is mutable but is intended to be written to only once by the codec upon creation. It should be treated as effectively immutable after deserialization.
- **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. It is designed to operate within the synchronous, single-threaded context of the game's core logic loop. No synchronization primitives are used, and concurrent access will lead to unpredictable behavior in the brush system.

## API Surface
The public API is minimal, exposing only the functionality required to integrate with the brush execution system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Applies the internal `operationMaskArg` to the provided `BrushConfig` instance. This is the primary entry point for the object's logic. |

## Integration Patterns

### Standard Usage
This operation is intended to be used as one step in a larger sequence of brush operations. The brush executor iterates through the list of operations, applying each one to a `BrushConfig` object that accumulates state.

```java
// Conceptual example of a brush executor
BrushConfig config = new BrushConfig();
List<SequenceBrushOperation> operations = loadBrushOperationsFromConfig();

// The executor iterates through all defined operations
for (SequenceBrushOperation op : operations) {
    // When op is an instance of MaskOperation, it sets the mask on the config
    op.modifyBrushConfig(entityStoreRef, config, this, accessor);
}

// Subsequent operations will now use the mask set in the config
executeFinalBrushAction(config);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not hold a reference to a `MaskOperation` instance and attempt to re-use it across different brush executions. A new instance should be deserialized for each use to ensure configuration integrity.
- **Direct Modification:** Do not modify the public `operationMaskArg` field after the object has been created by the codec system. This can break the "what you see is what you get" contract of configuration-driven tools.
- **Standalone Execution:** Calling `modifyBrushConfig` outside of a managed brush execution sequence has no effect, as the modified `BrushConfig` object will not be used by any subsequent world-editing logic.

## Data Pipeline
The flow of data for this component is linear, originating from a configuration source and terminating as a state change in a context object.

> Flow:
> Configuration Asset (e.g., JSON) -> `MaskOperation.CODEC` -> **MaskOperation Instance** -> `modifyBrushConfig` call -> `BrushConfig` State Update -> Subsequent Brush Operations

