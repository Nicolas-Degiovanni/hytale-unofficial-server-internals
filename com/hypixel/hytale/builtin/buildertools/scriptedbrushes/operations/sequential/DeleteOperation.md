---
description: Architectural reference for DeleteOperation
---

# DeleteOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Command

## Definition
```java
// Signature
public class DeleteOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The DeleteOperation is a stateless, atomic command object that represents the action of removing a single block from the world. It functions as a concrete implementation within the Scripted Brush framework, which uses a sequence of such operations to define complex building and editing behaviors.

Architecturally, this class is a leaf node in a Composite Pattern, where a brush's behavior is composed of a list of `SequenceBrushOperation` instances. Its sole responsibility is to instruct a transactional cache, the BrushConfigEditStore, to mark a block at specific coordinates as `Material.EMPTY`. It does not directly modify world state; instead, it contributes to a batch of changes that are committed by the parent brush execution system.

The static `CODEC` field is critical for the system's data-driven design. It allows the engine to serialize and deserialize brush configurations, enabling users to save, load, and share custom brushes without writing code.

## Lifecycle & Ownership
- **Creation:** Instances are created by the `BuilderCodec` framework during the deserialization of a brush configuration. A user or system defines a brush (e.g., in a UI or a configuration file), and the codec instantiates `DeleteOperation` as one of the steps in that brush's sequence. Direct instantiation by developers is an anti-pattern.
- **Scope:** The lifetime of a DeleteOperation instance is tied to its parent `BrushConfig` object. It is held within the configuration's operation list and persists as long as the brush is loaded in memory.
- **Destruction:** The object is eligible for garbage collection when its parent `BrushConfig` is unloaded or dereferenced. It requires no manual resource management.

## Internal State & Concurrency
- **State:** **Stateless and Immutable**. The DeleteOperation class contains no instance fields and its behavior is entirely determined by the arguments passed to its methods. The same instance can be reused for millions of block operations without side effects.
- **Thread Safety:** **Conditionally Safe**. The object itself is inherently thread-safe due to its stateless nature. However, its primary method, `modifyBlocks`, operates on a mutable `BrushConfigEditStore` object. The calling context, typically the brush execution engine, **must** ensure that access to the `BrushConfigEditStore` is synchronized. It is unsafe to invoke `modifyBlocks` from multiple threads on the same `BrushConfigEditStore` instance concurrently.

## API Surface
The public API is minimal, adhering to the interface defined by `SequenceBrushOperation`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBlocks(...) | boolean | O(1) | Marks a single block for deletion within the provided `BrushConfigEditStore`. Always returns true. |
| modifyBrushConfig(...) | void | O(1) | No-op. This operation does not alter the parent brush configuration during execution. |

## Integration Patterns

### Standard Usage
A developer or content creator does not interact with this class directly in Java code. Instead, it is declared as part of a data-driven brush definition. The brush system then loads this configuration and executes the operation.

A conceptual configuration might look like this:

```json
// Conceptual brush definition
{
  "name": "EraserBrush",
  "operations": [
    {
      "type": "DeleteOperation"
    }
  ]
}
```

The engine would then use the `CODEC` to instantiate the object and invoke it for each block in the brush's area of effect.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new DeleteOperation()`. The brush system relies on the `BuilderCodec` to manage instantiation and ensure proper integration. Bypassing the codec can lead to brushes that cannot be saved or networked.
- **Manual Invocation:** Do not call the `modifyBlocks` method directly. This bypasses the transactional integrity, undo/redo tracking, and performance optimizations provided by the parent `BrushConfigCommandExecutor`. All operations must be executed through the proper brush system entry points.

## Data Pipeline
The DeleteOperation acts as a simple processor in the builder tools data pipeline. It transforms a world coordinate into a "set to empty" command within a pending changeset.

> Flow:
> Brush Executor -> Iterates over target coordinates (x, y, z) -> **DeleteOperation.modifyBlocks(x, y, z)** -> `BrushConfigEditStore` (records pending change) -> World Transaction Commit -> Final World State

