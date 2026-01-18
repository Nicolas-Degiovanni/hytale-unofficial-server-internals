---
description: Architectural reference for DimensionsOperation
---

# DimensionsOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.dimensions
**Type:** Transient / Command Object

## Definition
```java
// Signature
public class DimensionsOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The DimensionsOperation is a data-driven command object within the Scripted Brush framework. It represents a single, atomic step in a sequence of operations designed to configure a BrushConfig instance before it is used. Its sole responsibility is to modify the width and height of a brush's shape.

This class is not a service or a manager; it is a pure data container with a single execution method. Its behavior is defined declaratively through a static CODEC, which deserializes configuration data (e.g., from a brush script file) into a fully-formed DimensionsOperation instance. This design decouples the brush configuration logic from the execution engine, allowing for complex brush behaviors to be defined entirely through data.

It operates on the principle of a Command Pattern, where the operation and its parameters (width and height) are encapsulated into an object that can be stored, sequenced, and executed by a generic processor.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the engine's serialization system using the static **CODEC** field. A brush script or definition containing a "Modify Dimensions" operation is parsed, and the codec instantiates this object, populating its fields from the source data. Direct instantiation by developers is a critical anti-pattern.
- **Scope:** Ephemeral. An instance of DimensionsOperation exists only for the duration of a single brush configuration event. It is created, its execution method is called once, and it is then immediately discarded.
- **Destruction:** The object holds no external references and is not registered with any system. It becomes eligible for garbage collection as soon as the `modifyBrushConfig` method returns and the parent operation sequence completes.

## Internal State & Concurrency
- **State:** The object is stateful, containing the `widthArg` and `heightArg` parameters. However, this state is intended to be configured only once upon creation by the CODEC. It should be treated as effectively immutable after deserialization.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be executed within a single-threaded context, such as the server's main update loop. The `modifyBrushConfig` method directly mutates the passed-in BrushConfig object. Concurrent access would result in severe data corruption and unpredictable brush behavior.

**WARNING:** Never share a BrushConfig instance across multiple threads, and ensure all operations modifying it are executed sequentially.

## API Surface
The public contract is minimal, centered on the execution method inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, ...) | void | O(1) | Resolves the relative dimensions and applies them to the BrushConfig. Clamps values between 1 and 75. This is the primary entry point for the command's logic. |

## Integration Patterns

### Standard Usage
This class is not used imperatively. It is defined declaratively within a brush script. The engine then deserializes and executes it as part of a larger operation sequence.

A developer would define this operation in a data file, not in Java code.

```yaml
# Hypothetical Brush Script Definition
# The engine parses this and creates a DimensionsOperation instance.
name: "Large Square Brush"
sequence:
  - type: "Modify Dimensions"
    Width: 15
    Height: 15
  - type: "Set Shape"
    shape: "square"
  # ... other operations
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DimensionsOperation()`. The object is not designed for manual setup; its state must be populated by the framework's CODEC to ensure correctness.
- **State Mutation:** Do not modify the `widthArg` or `heightArg` fields after the object has been created by the deserializer. This can lead to inconsistent behavior if the operation is ever re-used, which violates its ephemeral lifecycle contract.
- **Asynchronous Execution:** Do not invoke `modifyBrushConfig` from a separate thread or callback. All brush modifications must occur synchronously within the server tick.

## Data Pipeline
The flow of data for this component is linear and unidirectional, originating from a static configuration and resulting in a mutated in-memory object.

> Flow:
> Brush Script (Data Asset) -> Engine CODEC Deserializer -> **DimensionsOperation Instance** -> Brush Execution Sequencer -> `modifyBrushConfig` call -> Mutated BrushConfig State

