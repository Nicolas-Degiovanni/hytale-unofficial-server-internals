---
description: Architectural reference for LoopOperation
---

# LoopOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol.loops
**Type:** Transient Operation

## Definition
```java
// Signature
public class LoopOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The LoopOperation is a fundamental flow-control primitive within the Scripted Brushes system. It does not directly modify the world, but instead manipulates the execution pointer of the BrushConfigCommandExecutor. Its primary function is to enable repetitive sequences of brush operations, acting as a "goto" statement with a built-in counter.

This class is designed to be defined declaratively within a brush configuration script and then deserialized at runtime. It forms a directed, potentially cyclic graph of operations that allows for the creation of complex, procedural building patterns without requiring redundant operation definitions. The core mechanism involves instructing the command executor to jump to a previously defined, named index in the operation sequence.

**WARNING:** The system relies on a well-defined sequence with named indices. A LoopOperation pointing to a non-existent index name will result in a runtime failure of the brush execution.

### Lifecycle & Ownership
-   **Creation:** LoopOperation instances are not created directly via their constructor. They are instantiated exclusively by the Hytale `CODEC` system during the deserialization of a BrushConfig. The static `CODEC` field defines how to map configuration data (e.g., from a JSON file) to the `indexNameArg` and `repetitionsArg` fields.
-   **Scope:** The object's lifetime is tightly coupled to its parent BrushConfig. It exists as an element within the BrushConfig's internal list of operations and persists as long as the brush configuration is loaded.
-   **Destruction:** The object is eligible for garbage collection when its parent BrushConfig is unloaded or replaced. The `resetInternalState` method can be called to revert its execution state without destroying the object, allowing a brush script to be re-run.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. Its core internal state is the `repetitionsRemaining` counter. This field is initialized to an idle state (-1), set to the configured number of repetitions on its first execution, and decremented on each subsequent pass. This statefulness is essential for the loop to terminate correctly. The `indexNameArg` and `repetitionsArg` fields are configured at creation and are treated as immutable thereafter.

-   **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed to be managed and executed by a single BrushConfigCommandExecutor within a sequential, single-threaded context. Any external or multi-threaded modification of its internal state, particularly `repetitionsRemaining`, will corrupt the brush's execution flow and lead to undefined behavior, such as infinite loops or premature termination.

## API Surface
The public API is minimal, designed for interaction with the Scripted Brush execution engine, not for general developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | Executes the core loop logic. Manages the internal repetition counter and instructs the BrushConfigCommandExecutor to change its operating index. |
| resetInternalState() | void | O(1) | Resets the internal loop counter to its initial idle state. This is critical for re-executing a brush script. |

## Integration Patterns

### Standard Usage
A LoopOperation is never used in isolation. It is defined as part of a larger brush script and executed by the engine. The developer's interaction is purely declarative.

```java
// Conceptual representation of how the engine uses the operation.
// Developers do NOT write this code.

// Engine retrieves the current operation from the BrushConfig
SequenceBrushOperation currentOp = brushConfig.getCurrentOperation();

if (currentOp instanceof LoopOperation) {
    // The engine invokes the operation, passing its own context
    currentOp.modifyBrushConfig(
        entityStoreRef,
        brushConfig,
        commandExecutor,
        componentAccessor
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new LoopOperation()`. The object will be unconfigured, lacking the critical `indexNameArg` and `repetitionsArg` values, causing runtime failures. Always define loops within the brush's data configuration to be loaded by the `CODEC`.
-   **State Tampering:** Do not manually call `resetInternalState` during a brush execution. This method is for the controlling system to call between full executions of a script.
-   **Invalid Configuration:** Defining a loop with `repetitionsArg` greater than `MAX_REPETITIONS` (100) or less than 0 will cause the brush to enter an error state upon execution.

## Control Flow
The LoopOperation redirects the flow of control within the executor rather than processing data.

> Flow:
> BrushConfigCommandExecutor executes operations sequentially -> Reaches **LoopOperation** instance -> `modifyBrushConfig` is called -> **LoopOperation** decrements its internal counter -> If counter > 0, it calls `commandExecutor.loadOperatingIndex(indexNameArg)` -> The command executor's instruction pointer is reset to a previous operation -> The sequence repeats.

