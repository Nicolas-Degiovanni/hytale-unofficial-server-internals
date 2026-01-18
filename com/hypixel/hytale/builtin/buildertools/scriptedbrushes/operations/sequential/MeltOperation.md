---
description: Architectural reference for MeltOperation
---

# MeltOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Command

## Definition
```java
// Signature
public class MeltOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The MeltOperation class is a concrete implementation of the **Command Pattern**, representing a single, stateless, and atomic step within a scripted brush sequence. Its sole responsibility is to evaluate a single block coordinate and, if it meets specific criteria, replace it with air. This emulates a "melting" effect by removing the outermost layer of solid blocks.

This operation is designed to be entirely data-driven. The static CODEC field is the primary integration point with the Hytale engine, allowing instances of MeltOperation to be dynamically created from brush configuration files. This decouples the brush logic from the engine code, enabling content creators to define complex brush behaviors without writing Java.

MeltOperation is a sequential operation, meaning the brush engine invokes it for each block within the brush's area of effect, one by one. It does not modify the overall brush configuration; its effects are limited to the block data being edited.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale `BuilderCodec` system during the deserialization of a brush configuration. The `MeltOperation::new` constructor reference is provided to the codec for this purpose. Direct instantiation by developers is an anti-pattern.
- **Scope:** Extremely short-lived and function-scoped. An instance is typically created, used for a single `modifyBlocks` invocation as part of a larger brush application, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed by the Java Garbage Collector. As the object is stateless and holds no references after its execution method completes, it is cleaned up rapidly.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. This class contains no mutable instance fields. All contextual data required for its execution, such as the world state and player information, is passed as arguments to the `modifyBlocks` method. The primary state it interacts with is the external `BrushConfigEditStore`.

- **Thread Safety:** **Conditionally Thread-Safe**. The object itself is inherently thread-safe due to its stateless design. However, its methods operate on mutable, external state objects like `BrushConfigEditStore`. Concurrency control is therefore the responsibility of the calling system, which is the Scripted Brush engine. The engine must guarantee that calls to `modifyBlocks` and modifications to the provided `BrushConfigEditStore` are properly synchronized. It is assumed that the engine processes blocks in a single-threaded, sequential manner for any given brush application.

## API Surface
The public contract is minimal, focusing on its role as a step in a larger process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBlocks(...) | boolean | O(1) | Core execution logic. Evaluates and potentially modifies a single block at (x, y, z) within the provided `BrushConfigEditStore`. Always returns true. |
| modifyBrushConfig(...) | void | O(1) | No-op. This operation does not alter the persistent `BrushConfig`. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code by developers. Instead, it is declared within a data file that defines a scripted brush's behavior. The engine's `BuilderCodec` then parses this data and integrates the operation into the brush's execution sequence.

*Example of a conceptual brush definition:*
```json
{
  "name": "SurfaceMelterBrush",
  "operations": [
    {
      "type": "Melt"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MeltOperation()`. The brush system relies on the `CODEC` for proper registration and lifecycle management. Bypassing the codec will result in an operation that is not recognized by the engine.
- **Stateful Modification:** Do not alter this class to store state in instance variables. The brush engine assumes operations are stateless and may reuse or recreate instances unpredictably.
- **External Invocation:** Calling the `modifyBlocks` method from outside the Scripted Brush engine's processing loop is unsupported and will likely fail. The method requires a fully-formed and valid context, including a `BrushConfigEditStore`, which is managed exclusively by the engine during a brush stroke.

## Data Pipeline
The flow of data during this operation's execution is linear and localized to a single block.

> Flow:
> Brush Engine Iteration -> Provides (x, y, z) coordinate -> **MeltOperation.modifyBlocks** -> Reads neighbor blocks from `BrushConfigEditStore` -> Applies melt logic -> Writes `Material.EMPTY` to `BrushConfigEditStore` -> Brush Engine commits `BrushConfigEditStore` to World State

