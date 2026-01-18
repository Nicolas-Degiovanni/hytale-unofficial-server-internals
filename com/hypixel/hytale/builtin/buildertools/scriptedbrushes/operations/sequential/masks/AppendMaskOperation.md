---
description: Architectural reference for AppendMaskOperation
---

# AppendMaskOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.masks
**Type:** Transient / Command

## Definition
```java
// Signature
public class AppendMaskOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The AppendMaskOperation class is a concrete implementation of the Command Pattern, designed to operate within the server-side Scripted Brush system. It represents a single, atomic step in a sequence of operations that collectively define a brush's behavior.

This class's sole responsibility is to modify the state of a BrushConfig object by appending a new BlockMask to its existing operation mask. It does not perform any world edits directly. Instead, it prepares the brush's configuration for subsequent operations in the sequence that will read and apply this mask.

AppendMaskOperation is a data-driven component. Its behavior is determined by the BlockMask provided in its public field, operationMaskArg. This configuration is almost exclusively loaded and instantiated from a data source (e.g., a JSON brush script) via the Hytale codec system, using the static CODEC field.

## Lifecycle & Ownership
- **Creation:** Instances are created by the BuilderCodec deserialization pipeline when a brush script is loaded by the server. It is not intended for manual instantiation in game logic. The codec reads the operation type and its corresponding mask data, then constructs an AppendMaskOperation object.
- **Scope:** Extremely short-lived and stateless. An instance exists only for the duration of its execution within a brush modification sequence. It is created, its modifyBrushConfig method is invoked, and it is then immediately eligible for garbage collection.
- **Destruction:** Managed entirely by the Java garbage collector. There is no explicit cleanup or destruction logic, as the object holds no persistent resources.

## Internal State & Concurrency
- **State:** The class is effectively immutable after its initial construction by the codec. The operationMaskArg field is populated during deserialization and is not expected to be mutated at runtime. The object itself is stateless; its purpose is to transfer its configured data to the BrushConfig.
- **Thread Safety:** This class is not thread-safe and is not designed for concurrent access. All brush configuration modifications are expected to occur serially on the main server thread. Attempting to execute brush operations concurrently on the same BrushConfig will result in race conditions and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, ...) | void | O(1) | Appends the configured operationMaskArg to the target BrushConfig. This is the primary entry point for the command. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in procedural Java code. It is designed to be defined declaratively within a brush script that is processed by the engine. The system invokes the operation as part of a larger sequence.

A conceptual representation of its usage within a script:

```yaml
# This is a conceptual representation, not actual Hytale config syntax.
operations:
  - type: "SetMaterial"
    material: "hytale:stone"
  - type: "AppendMask"
    AppendMask:
      type: "Noise"
      threshold: 0.5
  - type: "Paint" # This paint operation will now use the combined mask
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AppendMaskOperation()`. The object's state will be uninitialized (an empty mask), and it will not be part of a managed execution sequence. Always define operations within a data-driven brush script.
- **State Mutation:** Avoid modifying the public `operationMaskArg` field after the object has been created by the codec. This breaks the declarative nature of the system and can lead to unpredictable brush behavior.
- **Incorrect Sequencing:** Placing this operation after the operation it is intended to modify is a common logical error. The sequence is executed in order; a mask appended after a paint operation will not affect that paint operation.

## Data Pipeline
The primary flow involves deserializing declarative data into an executable command that modifies a target object's state.

> Flow:
> Brush Script Data (e.g., JSON) -> BuilderCodec Deserializer -> **AppendMaskOperation Instance** -> `modifyBrushConfig` Invocation -> BrushConfig State Update

---

