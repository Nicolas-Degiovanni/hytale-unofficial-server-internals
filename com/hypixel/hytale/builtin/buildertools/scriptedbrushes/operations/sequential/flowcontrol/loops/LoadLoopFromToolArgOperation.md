---
description: Architectural reference for LoadLoopFromToolArgOperation
---

# LoadLoopFromToolArgOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol.loops
**Type:** Transient State Object

## Definition
```java
// Signature
public class LoadLoopFromToolArgOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The LoadLoopFromToolArgOperation is a control-flow instruction within the Scripted Brush system. It functions as a dynamic, stateful loop controller, analogous to a `for` loop in a traditional programming language. Its primary purpose is to repeat a sequence of brush operations a variable number of times.

This class is not a standalone service; it is a node within a directed sequence of operations managed by a BrushConfigCommandExecutor. Its key architectural distinction is that the loop's iteration count is not defined within the brush script itself. Instead, it is dynamically read from the arguments associated with the player's currently equipped BuilderTool ItemStack. This design decouples the brush's logic from its configuration, allowing players to modify the behavior of a tool (e.g., the number of times a pattern repeats) without editing the underlying brush script.

Internally, it manipulates the program counter of the BrushConfigCommandExecutor by instructing it to jump back to a previously defined index, effectively creating a loop.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale Codec system during the deserialization of a brush configuration asset. The static CODEC field defines how the object is constructed from its serialized form. It is never instantiated directly in game logic.
-   **Scope:** The object's lifetime is bound to its parent BrushConfig. However, its internal state, specifically the repetitionsRemaining field, is transient and only valid for the duration of a single, continuous brush execution. The `resetInternalState` method is invoked between distinct brush uses to ensure state is not leaked across operations.
-   **Destruction:** The object is eligible for garbage collection when its parent BrushConfig is unloaded.

## Internal State & Concurrency
-   **State:** This class is highly mutable and stateful. Its core logic revolves around the `repetitionsRemaining` integer field, which tracks the current state of the loop. It transitions from an idle state (-1), to an initialized state (read from tool arguments), and is decremented with each pass through the loop.
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. Scripted Brush operations are executed sequentially within a single-threaded context managed by the BrushConfigCommandExecutor. Any concurrent access to `modifyBrushConfig` or modification of `repetitionsRemaining` would corrupt the loop's state and lead to undefined behavior.

## API Surface
The public contract is designed for interaction with the brush execution system, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | Executes the core loop logic. Reads tool arguments on the first pass and decrements the counter, jumping the executor's instruction pointer on subsequent passes. |
| resetInternalState() | void | O(1) | Resets the internal loop counter to its idle state. Critical for ensuring predictable behavior between brush uses. |

## Integration Patterns

### Standard Usage
This operation is never invoked directly. It is defined within a brush script's sequence of operations. The BrushConfigCommandExecutor is responsible for invoking `modifyBrushConfig` at the correct time during the script's execution.

A conceptual brush script definition might look like this:

```yaml
# This is a conceptual representation
- type: StoreIndexOperation
  name: "loop_start"
- type: SomeOtherBrushOperation
  # ...
- type: LoadLoopFromToolArgOperation
  StoredIndexName: "loop_start"
  ArgName: "repeat_count"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new LoadLoopFromToolArgOperation()`. The object must be configured and instantiated via the framework's CODEC system to be part of a valid brush.
-   **State Leakage:** Failure to call `resetInternalState` between separate executions of the same brush will cause the loop counter to persist, resulting in incorrect behavior on the next use. The framework handles this, but custom executors must respect this contract.
-   **Invalid Argument Types:** The tool argument specified by `ArgName` must be an Integer. The operation will raise a brush error if the argument is missing or is not a valid integer, halting the execution of the brush.

## Data Pipeline
This component acts as a control-flow gate rather than a data transformer. Its "data" is the player's intent and the tool's configuration, which it uses to alter the execution flow.

> Flow:
> Player Input (Use Tool) -> Server invokes BrushConfigCommandExecutor -> Executor reaches **LoadLoopFromToolArgOperation** -> Operation reads Player and ItemStack state -> Retrieves `ArgData` from BuilderTool -> Initializes or decrements internal counter -> **LoadLoopFromToolArgOperation** commands Executor to jump to a new instruction index -> Executor continues from new index.

