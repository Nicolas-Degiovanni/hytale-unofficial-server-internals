---
description: Architectural reference for UseOperationMaskOperation
---

# UseOperationMaskOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.masks
**Type:** Transient

## Definition
```java
// Signature
public class UseOperationMaskOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The UseOperationMaskOperation is a concrete implementation of the **Command Pattern**, designed to function as a single, atomic step within the Scripted Brush system's sequential pipeline. Its sole responsibility is to modify the execution context of a brush by enabling or disabling the *operation mask* feature for all subsequent operations within the same sequence.

This class does not perform any world modification itself. Instead, it acts as a state controller, altering the behavior of other operations that follow it. It is a leaf-node component in the brush system, intended to be defined declaratively in data files rather than instantiated directly in code. The engine's serialization layer, via the static CODEC field, is responsible for creating instances of this class from brush script definitions.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the `BuilderCodec` deserializer when the engine loads a Scripted Brush definition from a data source. This process translates a declarative configuration (e.g., from a JSON file) into a live Java object.
- **Scope:** The object's lifetime is ephemeral, scoped to the execution of a single brush application. It is instantiated as part of a sequence, its `modifyBrushConfig` method is invoked once, and it is then discarded.
- **Destruction:** The object holds no external resources and is managed entirely by the Java Garbage Collector. It becomes eligible for collection immediately after the brush sequence it belongs to has finished executing.

## Internal State & Concurrency
- **State:** The class contains a single, mutable state field: `useOperationMask`. This value is set during deserialization and is intended to be read-only for the remainder of the object's lifecycle. It effectively acts as an immutable configuration parameter post-creation.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. It is designed to be created, executed, and destroyed within the single, synchronous thread responsible for game world logic and builder tool execution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | static BuilderCodec | O(1) | Provides the contract for serializing and deserializing this operation from data files. This is the primary entry point for creating instances. |
| modifyBrushConfig(...) | void | O(1) | The core execution method. It applies the internal `useOperationMask` state to the provided `BrushConfig` context. |

## Integration Patterns

### Standard Usage
This operation is not intended for direct, imperative use by developers. It is designed to be defined declaratively within a brush script's sequence of operations. The engine then deserializes and executes it as part of the brush pipeline.

A conceptual data definition might look like this:

```yaml
# Example Brush Script Definition
name: "Masked Sphere Brush"
sequence:
  - type: "UseOperationMaskOperation"
    UseOperationMask: true # Enable the mask
  - type: "SetMaskShapeOperation"
    shape: "sphere"
    radius: 5
  - type: "FillOperation" # This operation will now be constrained by the mask
    block: "hytale:stone"
  - type: "UseOperationMaskOperation"
    UseOperationMask: false # Disable the mask for subsequent steps
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new UseOperationMaskOperation()`. The object's state will not be configured correctly. Always define it in a data file to be loaded by the engine's CODEC.
- **State Mutation:** Do not modify the public `useOperationMask` field after the object has been created by the deserializer. This can lead to unpredictable behavior and breaks the declarative model.
- **Incorrect Ordering:** Placing this operation after the operations it is meant to affect will have no impact on them. It must be placed *before* any operation that needs to respect the operation mask setting.

## Data Pipeline
The primary flow involves deserialization from a data source, which configures the object's state. This state is then used to mutate a shared configuration object that influences the behavior of later stages in the pipeline.

> Flow:
> Brush Script Data -> `BuilderCodec` Deserializer -> **UseOperationMaskOperation Instance** -> `modifyBrushConfig` -> BrushConfig State Mutation -> Subsequent Brush Operations

