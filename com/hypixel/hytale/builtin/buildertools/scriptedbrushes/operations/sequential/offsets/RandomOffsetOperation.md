---
description: Architectural reference for RandomOffsetOperation
---

# RandomOffsetOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.offsets
**Type:** Transient Operation

## Definition
```java
// Signature
public class RandomOffsetOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The RandomOffsetOperation is a stateful, single-purpose component within the Scripted Brushes subsystem. It does not directly modify the game world. Instead, its sole function is to act as a *configuration mutator* within a pipeline of brush operations.

This class embodies the Command pattern, where an operation and its parameters are encapsulated into a standalone object. Its primary role is to introduce controlled proceduralism into building tools by altering the brush's origin point before it is applied. By randomly offsetting the origin, it allows creators to design tools that paint with less uniformity, creating more organic or varied structures.

The static CODEC field is the most critical architectural feature. It signifies that this class is not intended for direct instantiation in code but is designed to be deserialized from a data format (like JSON or HOCON). This allows game designers and content creators to declaratively construct complex brush behaviors without writing Java code.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale codec system during the deserialization of a scripted brush definition. The public no-argument constructor exists to satisfy the requirements of the BuilderCodec factory.
- **Scope:** The object's lifetime is extremely short, scoped to the execution of a single brush application. It is created, its `modifyBrushConfig` method is invoked once as part of a sequence, and it is then immediately eligible for garbage collection.
- **Destruction:** Managed by the Java garbage collector. It holds no native resources or references that would require manual cleanup.

## Internal State & Concurrency
- **State:** The class is stateful. Its state consists of the three `RelativeIntegerRange` fields: xOffsetArg, yOffsetArg, and zOffsetArg. This state is configured once at creation time by the codec system and is treated as immutable thereafter. The operation itself is a container for this configuration.
- **Thread Safety:** This class is **not thread-safe** and must not be used concurrently. It is designed to be executed sequentially on the main server thread. The `modifyBrushConfig` method introduces a side effect by mutating the passed-in BrushConfig object. Concurrent calls would lead to race conditions and unpredictable brush behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, ...) | void | O(1) | Mutates the provided BrushConfig object. Calculates a new random origin offset based on its internal range fields and applies it to the configuration. This is the primary entry point for the operation's logic. |

## Integration Patterns

### Standard Usage
This class is never instantiated or called directly by developers. It is designed to be defined declaratively within a scripted brush's configuration file. The brush system's execution engine will then manage its lifecycle and invocation.

A conceptual configuration defining this operation might look like this:

```json
// Fictional brush definition format
{
  "name": "Scattered Boulders Brush",
  "sequence": [
    {
      "type": "Randomize Offset",
      "XOffsetRange": { "min": -5, "max": 5 },
      "YOffsetRange": { "min": 0, "max": 0 },
      "ZOffsetRange": { "min": -5, "max": 5 }
    },
    {
      "type": "Place Schematic",
      "schematic": "boulder_small.hyschem"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new RandomOffsetOperation()`. This creates an unconfigured object with default ranges (1, 1), which provides no randomization. Instances must be created via the data-driven codec system.
- **State Modification:** Do not attempt to modify the public `xOffsetArg`, `yOffsetArg`, or `zOffsetArg` fields after the object has been created. This violates the assumption that the operation's configuration is immutable during its short lifecycle.
- **Concurrent Execution:** Never share a BrushConfig instance across multiple threads and have this operation (or any other) modify it. All brush operations on a given config must be serialized.

## Data Pipeline
The primary flow involves the transformation of a BrushConfig object. This operation is one step in a potential chain of modifications.

> Flow:
> Brush Definition (Data File) -> Codec Deserializer -> **RandomOffsetOperation Instance** -> Brush Execution Engine -> `modifyBrushConfig` -> Mutated BrushConfig -> Subsequent Brush Operations -> Final World Edit

