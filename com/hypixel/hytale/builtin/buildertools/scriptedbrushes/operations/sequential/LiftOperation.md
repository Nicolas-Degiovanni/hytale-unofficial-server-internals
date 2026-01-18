---
description: Architectural reference for LiftOperation
---

# LiftOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient

## Definition
```java
// Signature
public class LiftOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The LiftOperation is a concrete implementation of the Strategy Pattern, designed to function as a single, atomic step within a larger sequence of brush operations. It is a server-side component responsible for a specific world modification behavior: extruding terrain upwards.

This class is not a standalone service but a data-driven command object. It is intended to be defined within a BrushConfig asset and instantiated by the engine's serialization system via its static CODEC. Its primary role is to modify a transactional block buffer, the BrushConfigEditStore, based on a simple rule: if a block is air but is adjacent to a solid block, it becomes a copy of a nearby block. This creates a "lifting" or "extruding" effect on surfaces.

LiftOperation is entirely dependent on the context provided by the parent brush system, including the player's current BuilderState for sampling materials and the BrushConfig for potential material overrides.

### Lifecycle & Ownership
- **Creation:** LiftOperation is instantiated by the BuilderCodec deserializer when a BrushConfig containing this operation is loaded. It is never created directly by developers using the `new` keyword.
- **Scope:** The object's lifetime is tied to its parent BrushConfig. It is effectively a stateless command object, and an instance exists for each "lift" operation defined in a brush's sequence. It is used and discarded during a single brush application cycle.
- **Destruction:** The instance is eligible for garbage collection once the BrushConfig is unloaded or the brush application completes.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no mutable instance fields. All state required for its execution, such as world data and brush settings, is passed as arguments to the `modifyBlocks` method.
- **Thread Safety:** The class itself is immutable and inherently thread-safe. However, the methods it exposes are designed to be called exclusively from the main server thread that governs world updates. The objects it operates on, particularly BrushConfigEditStore, are not thread-safe and must not be accessed from multiple threads. The parent brush system is responsible for ensuring this execution constraint.

## API Surface
The public API is minimal and consists of overrides from the SequenceBrushOperation base class. Direct invocation is strongly discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBlocks(...) | boolean | O(1) | Core execution logic. Modifies the provided BrushConfigEditStore for a single block coordinate. This method is the heart of the operation. |
| modifyBrushConfig(...) | void | O(1) | A lifecycle hook for operations that need to alter the brush's configuration dynamically. This is a no-op in LiftOperation. |

## Integration Patterns

### Standard Usage
This class is not used directly in code but is configured within a brush's asset definition. The engine's BuilderToolsPlugin is responsible for invoking it. The conceptual usage by the engine is as follows.

```java
// Conceptual engine code for applying a brush
BrushConfig activeBrush = player.getActiveBrush();
BrushConfigEditStore editBuffer = new BrushConfigEditStore(world);

for (BlockPos pos : brush.getAffectedVolume()) {
    for (SequenceBrushOperation operation : activeBrush.getOperations()) {
        // The engine invokes modifyBlocks on each operation in the sequence,
        // including an instance of LiftOperation if configured.
        operation.modifyBlocks(playerRef, activeBrush, executor, editBuffer, pos.x, pos.y, pos.z, accessor);
    }
}

// After all operations, the transactional buffer is applied to the world.
editBuffer.applyToWorld();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new LiftOperation()`. The class is designed to be managed by the serialization system. Direct creation will result in an object that is not integrated into any brush sequence.
- **External Invocation:** Do not call `modifyBlocks` from outside the context of the BuilderToolsPlugin's brush application loop. The method requires a fully populated and valid state (BrushConfigEditStore, ComponentAccessor) that is only guaranteed by the engine.

## Data Pipeline
The LiftOperation acts as a processor in a larger data flow for world editing. It reads from a transactional buffer and the live world state, and writes its calculated changes back into the buffer.

> Flow:
> Player Input (Use Brush) -> BuilderToolsPlugin -> BrushConfig -> **LiftOperation.modifyBlocks** -> BrushConfigEditStore (Buffer) -> World Storage (Commit)

