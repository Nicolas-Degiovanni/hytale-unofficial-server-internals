---
description: Architectural reference for OffsetOperation
---

# OffsetOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.offsets
**Type:** Transient Command Object

## Definition
```java
// Signature
public class OffsetOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The OffsetOperation is a specific, data-driven command within the Scripted Brushes framework. It does not represent a long-lived service but rather a single, configurable step in a larger sequence of brush modifications. Its primary architectural role is to decouple the logic of offsetting a brush's origin from the core brush execution engine.

This class embodies the **Command Pattern**. An instance of OffsetOperation is a self-contained request to modify the origin point of a BrushConfig. The engine that processes scripted brushes simply executes this command by invoking its modifyBrushConfig method, without needing to know the internal details of how the offset is calculated.

The static CODEC field is central to its design. It signifies that OffsetOperation objects are not intended to be manually instantiated in Java code. Instead, they are deserialized from external data files (e.g., JSON definitions for a custom brush). This data-driven approach allows designers and developers to create complex brush behaviors by composing sequences of operations in configuration files rather than writing new Java classes.

## Lifecycle & Ownership
- **Creation:** An OffsetOperation instance is created by the Hytale Codec system during the loading and parsing of a scripted brush definition file. The static CODEC field is used by a higher-level service, such as a BrushManager, to deserialize the configuration data into a concrete Java object.

- **Scope:** The object's lifetime is bound to the in-memory representation of the scripted brush it belongs to. It is a stateless command object that persists as part of a brush's definition until that definition is unloaded or reloaded.

- **Destruction:** The object is eligible for garbage collection when the parent brush definition is discarded. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** The state of an OffsetOperation is defined by its configuration fields: offsetArg, targetFieldArg, and negateArg. This state is populated once during deserialization and should be considered **immutable** for the remainder of its lifecycle. The operation itself does not cache data or modify its own state during execution; it only modifies the passed-in BrushConfig object.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and executed within a single, synchronous thread, typically the main server thread responsible for world modification and builder tool logic. Concurrent modification of its fields or execution from multiple threads will lead to unpredictable behavior.

## API Surface
The public contract is minimal, exposing only the execution logic. Configuration is handled exclusively through the Codec system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Executes the offset logic. Calculates the final offset vector based on its internal state and applies it to the provided BrushConfig. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is incorrect. Its usage is defined declaratively within a brush script file. The system then deserializes this definition to create and execute the operation.

A conceptual brush definition might look like this:

```yaml
# In a theoretical brush_definitions.yml file
name: "MyOffsettingBrush"
sequence:
  - type: "hytale:offset" # This maps to OffsetOperation
    Offset: "~5 ~0 ~0"    # Use a relative offset of 5 on the X axis
  - type: "hytale:place_block"
    block: "hytale:stone"
```

The game's brush loading system would parse this YAML and internally execute the OffsetOperation before moving to the next operation in the sequence.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using **new OffsetOperation()**. The object will be unconfigured and useless. It is designed to be created and configured exclusively by the Codec deserializer.

- **State Mutation:** Do not modify the public fields (offsetArg, targetFieldArg) after the object has been loaded by the system. Treating the object as a mutable bean will break the declarative nature of the scripted brush system and can lead to non-deterministic behavior.

## Data Pipeline
The OffsetOperation acts as a transformation step in the data flow of a brush's configuration.

> Flow:
> Brush Definition File (JSON/YAML) -> Hytale Codec Deserializer -> **OffsetOperation Instance** -> Scripted Brush Executor -> `modifyBrushConfig` call -> Modified BrushConfig State -> Subsequent Brush Operations

