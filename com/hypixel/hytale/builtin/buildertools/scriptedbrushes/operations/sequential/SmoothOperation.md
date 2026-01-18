---
description: Architectural reference for the SmoothOperation class, a sequential brush operation for builder tools.
---

# SmoothOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Operation

## Definition
```java
// Signature
public class SmoothOperation extends SequenceBrushOperation {
```

## Architecture & Concepts

The SmoothOperation class is a concrete implementation of a sequential brush operation within the Hytale Builder Tools framework. It is designed to be a single, modular step in a multi-stage "scripted brush" pipeline. Its primary function is to homogenize a targeted area by changing blocks to match the most common block type in their immediate vicinity.

Architecturally, this class embodies a data-driven design pattern, facilitated by its static **CODEC** field. This allows instances of SmoothOperation, including its configurable parameters like *smoothStrength*, to be defined in external data files (e.g., HOCON or JSON) and deserialized by the engine at runtime. It does not operate directly on the world state but rather on a transactional buffer, the BrushConfigEditStore, ensuring that changes are atomic and can be rolled back.

This class is not a standalone service but a stateful, configurable component whose lifecycle is managed entirely by the scripted brush system.

### Lifecycle & Ownership

-   **Creation:** SmoothOperation is instantiated by the Hytale **Codec** system during the deserialization of a BrushConfig. The static CODEC field provides the `SmoothOperation::new` constructor reference for this purpose. Direct instantiation by developers is a design violation.
-   **Scope:** An instance of SmoothOperation exists for the lifetime of its parent BrushConfig object. It is created when a brush is selected or loaded and persists until that brush configuration is no longer in use.
-   **Destruction:** The object is eligible for garbage collection once its parent BrushConfig is dereferenced. It manages no native resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency

-   **State:** This class is **highly mutable**. It maintains configurable state (*smoothStrength*) and derived internal state (*smoothVolume*, *smoothRadius*). The `modifyBrushConfig` method is responsible for synchronizing the derived state from the configured state before any block modifications occur.

-   **Thread Safety:** This class is **not thread-safe** and must be confined to a single thread. The Builder Tools system guarantees that all operations on a given brush, including calls to `modifyBrushConfig` and `modifyBlocks`, are executed sequentially within the server's main tick or a dedicated world modification thread. Unsynchronized access from multiple threads will lead to race conditions and unpredictable smoothing behavior.

## API Surface

The public contract is designed for invocation by the parent brush system, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBlocks(...) | boolean | O(1) | Executes the core smoothing logic for a single block at coordinates (x, y, z). Reads neighborhood data and writes the result to the BrushConfigEditStore. Complexity is constant per block as the neighborhood scan radius is fixed. |
| modifyBrushConfig(...) | void | O(1) | A pre-computation callback invoked by the brush system before any `modifyBlocks` calls. It updates internal derived state based on the configured *smoothStrength*. |

## Integration Patterns

### Standard Usage

Developers should not invoke SmoothOperation methods directly. Instead, they should define it as part of a brush's operation sequence in a data file. The system will then manage its lifecycle and execution. The following example is a conceptual illustration of how the *system* uses the class.

```java
// Conceptual example of system-level invocation
// This code would exist within the core brush execution logic.

// 1. Pre-computation step for all operations in the sequence
smoothOp.modifyBrushConfig(ref, brushConfig, executor, accessor);

// 2. Per-block execution loop
for (BlockPos pos : brush.getAffectedBlocks()) {
    smoothOp.modifyBlocks(ref, brushConfig, executor, editStore, pos.x, pos.y, pos.z, accessor);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new SmoothOperation()`. The class is designed to be configured and created via the `CODEC` system. Manual creation bypasses this critical data-driven pipeline.
-   **Execution Order Violation:** Calling `modifyBlocks` before `modifyBrushConfig` has been invoked for the current configuration will result in the operation using stale internal parameters (*smoothVolume*, *smoothRadius*), leading to incorrect behavior.
-   **State Mutation After Pre-computation:** Modifying the public `smoothStrength` field after `modifyBrushConfig` has been called but before the `modifyBlocks` loop is complete is an unsupported pattern that will cause inconsistent results across a single brush application.

## Data Pipeline

The SmoothOperation functions as a transformation stage within the larger Builder Tools data pipeline. It reads from a buffered view of the world and writes its intended changes back to that same buffer.

> Flow:
> Brush Configuration File -> **CODEC Deserializer** -> SmoothOperation Instance -> Brush Executor calls `modifyBrushConfig` -> Brush Executor calls `modifyBlocks` -> Writes to **BrushConfigEditStore** -> World Patcher applies changes from store

