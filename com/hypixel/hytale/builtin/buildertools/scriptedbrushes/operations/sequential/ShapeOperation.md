---
description: Architectural reference for ShapeOperation
---

# ShapeOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient

## Definition
```java
// Signature
public class ShapeOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The ShapeOperation class is a concrete implementation of the Command Pattern, designed to function as a single, atomic step within a larger scripted brush sequence. Its sole responsibility is to encapsulate a brush shape modification, specifically by holding a reference to a BrushShape enum value.

This class is not a service or a manager; it is a passive data structure that represents a configuration change. It is designed to be deserialized from asset files via its static CODEC field, allowing game designers to define complex brush behaviors in data. A higher-level system, the brush script executor, iterates through a list of these operations and applies them in order to a target BrushConfig object. ShapeOperation acts as the data payload that instructs the executor on how to modify the brush's area of effect.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale codec system during the deserialization of a brush script asset. The static CODEC field defines the mapping between the configuration data (e.g., a "Shape" key in a JSON file) and the in-memory ShapeOperation object. Manual instantiation via its constructor is rare and generally discouraged.
- **Scope:** The lifetime of a ShapeOperation instance is bound to its parent brush script definition. It is a value object held within a collection of operations and persists only as long as the brush script is loaded.
- **Destruction:** The object is marked for garbage collection when the containing brush script is unloaded or otherwise dereferenced. It manages no native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** The class holds a single, mutable state field: brushShapeArg. This field is initialized to BrushShape.Cube by default and is overwritten during deserialization by the CODEC. While technically public and mutable, it is intended to be written to only once upon creation.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and executed within a single, well-defined thread, typically the server's main update loop. Concurrent modification of the brushShapeArg field from multiple threads will result in unpredictable behavior and is unsupported. All interactions must be externally synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Applies the stored brushShapeArg to the target BrushConfig by calling its setShape method. This is the primary execution point of the operation. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most systems. It is invoked by an internal brush script execution engine. The conceptual usage pattern involves an executor applying a list of operations to a configuration object.

```java
// Conceptual example of an executor applying the operation
BrushConfig targetConfig = new BrushConfig();
ShapeOperation shapeOp = deserializedFromAsset; // Assume this was loaded by the CODEC

// The executor applies the operation to the configuration
shapeOp.modifyBrushConfig(entityStoreRef, targetConfig, executor, accessor);

// targetConfig.getShape() now returns the value from shapeOp.brushShapeArg
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid creating instances with `new ShapeOperation()`. This bypasses the data-driven architecture and the CODEC system, which is the canonical source for creating and configuring these operations.
- **State Mutation After Application:** Do not modify the public `brushShapeArg` field after the operation has been applied to a BrushConfig. The configuration object will not be updated automatically. These operations should be treated as effectively immutable after their initial creation and configuration.

## Data Pipeline
ShapeOperation functions as a component in a configuration pipeline, not a real-time data processing stream.

> Flow:
> Brush Script Asset (File) -> Hytale Codec System -> **ShapeOperation Instance** -> Brush Script Executor -> BrushConfig State Update

