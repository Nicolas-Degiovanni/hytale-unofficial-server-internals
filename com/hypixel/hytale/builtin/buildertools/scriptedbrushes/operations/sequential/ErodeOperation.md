---
description: Architectural reference for ErodeOperation
---

# ErodeOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Component

## Definition
```java
// Signature
public class ErodeOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The ErodeOperation is a specialized algorithm component designed to be plugged into the Scripted Brush system. It functions as a single, stateful step within a multi-pass `SequenceBrushOperation`, simulating effects like erosion, melting, smoothing, or filling on a targeted volume of blocks.

Its primary architectural role is to provide a declarative, rule-based block transformation. Instead of being invoked directly, it is defined within a brush's configuration and deserialized at runtime via its static `CODEC`. The operation does not modify the game world directly; instead, it reads from and writes to a `BrushConfigEditStore`, which acts as an in-memory, transactional buffer for all changes made by the brush.

The core logic is governed by an `ErodePreset` enum. Each preset defines the number of neighboring faces required to trigger a block removal (erosion) or a block addition (fill), as well as the number of iterations dedicated to each phase. This two-phase (erode, then fill) process allows for complex morphological transformations on terrain and structures.

## Lifecycle & Ownership
-   **Creation:** ErodeOperation is not instantiated directly using the `new` keyword in standard gameplay code. It is deserialized from a brush configuration file or data stream by the Hytale `Codec` system, specifically using its static `CODEC` field. This typically occurs when a player selects a scripted brush tool.
-   **Scope:** The object's lifetime is bound to a single, complete execution of a scripted brush. It is created, used across multiple iterations and block positions for one brush application, and then becomes eligible for garbage collection.
-   **Destruction:** As a standard Java object with a transient scope, it is automatically garbage collected once the parent brush executor completes its operation and releases all references to it. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
-   **State:** This class is mutable and stateful during its active lifecycle. Its behavior is dictated by two key fields:
    -   `erodePresetArg`: Configured upon creation and determines the rules for the entire operation.
    -   `iterationIndex`: A mutable integer updated by the brush system via `beginIterationIndex` before each pass. This field is critical for switching between the erosion and fill phases.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be driven by a single-threaded brush execution engine that calls its methods in a strict, sequential order. The `iterationIndex` field makes concurrent calls to `modifyBlocks` inherently unsafe, as it would lead to race conditions and unpredictable behavior. All interactions with this class must be confined to the world modification thread.

## API Surface
The public API is defined by its parent, `SequenceBrushOperation`, and serves as the contract with the brush execution engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBlocks(...) | boolean | O(1) | Core logic. Checks 6 neighbors of a single block and modifies the `BrushConfigEditStore` based on the current preset and iteration. |
| beginIterationIndex(int) | void | O(1) | Sets the internal iteration counter. Called by the brush engine before each pass over the brush volume. |
| getNumModifyBlockIterations() | int | O(1) | Returns the total number of passes required, calculated from the active `ErodePreset`. |
| modifyBrushConfig(...) | void | O(1) | Unused in this implementation. Fulfills the interface contract but performs no action. |

## Integration Patterns

### Standard Usage
An ErodeOperation is not used imperatively. It is defined declaratively within a brush's data configuration and executed by the engine. A developer would configure it, not call its methods.

*Conceptual Brush Definition (e.g., in JSON):*
```json
{
  "name": "Smooth Terrain Brush",
  "operations": [
    {
      "type": "erode",
      "ErodePreset": "Smooth"
    }
  ]
}
```

The brush system would then deserialize this configuration, creating an `ErodeOperation` instance and integrating it into the execution pipeline.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ErodeOperation()`. The object is not designed for manual setup and must be configured and created via the `CODEC` system to ensure its properties are correctly initialized.
-   **State Tampering:** Do not call `beginIterationIndex` manually. This method is strictly for the parent brush executor to control the operation's phase. Calling it out of sequence will corrupt the erosion/fill logic.
-   **Concurrent Execution:** Never share an instance of ErodeOperation across multiple threads. It must only be used by the single thread responsible for applying the brush modification.

## Data Pipeline
The ErodeOperation functions as a transformation stage within the larger brush data pipeline. It exclusively reads from and writes to a temporary data store, ensuring transactional integrity.

> Flow:
> Brush Executor begins pass -> Executor calls **ErodeOperation.beginIterationIndex** -> Executor iterates block at (x,y,z) -> Executor calls **ErodeOperation.modifyBlocks** -> Operation reads 6 neighbors from `BrushConfigEditStore` -> Applies `ErodePreset` rules -> Operation writes new material to `BrushConfigEditStore` -> Executor commits entire `BrushConfigEditStore` to the World Storage after all passes are complete.

