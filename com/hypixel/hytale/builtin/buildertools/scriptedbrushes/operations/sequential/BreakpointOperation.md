---
description: Architectural reference for BreakpointOperation
---

# BreakpointOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Command

## Definition
```java
// Signature
public class BreakpointOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The BreakpointOperation is a specialized command within the Scripted Brush framework, designed exclusively for debugging complex brush logic. It functions as a server-side, conditional breakpoint that can pause the execution of a brush script, inspect its state, and provide feedback to the user or server console.

Architecturally, it is a single, self-contained step in a larger sequence of operations managed by the BrushConfigCommandExecutor. Its primary role is to interrupt the standard execution flow based on a set of predefined conditions. A key design feature is its composition with other components; it reuses the conditional logic from JumpIfCompareOperation, allowing for sophisticated, state-aware breakpoints (e.g., "break only if variable X is greater than 10").

This operation acts as a critical bridge between the automated execution of a brush and the manual intervention of a developer or designer, enabling interactive debugging of in-game content creation tools.

## Lifecycle & Ownership
-   **Creation:** BreakpointOperation instances are not created directly via a constructor in game logic. They are instantiated by the engine's serialization system, specifically through the static BuilderCodec field named CODEC. This occurs when the server loads a scripted brush definition from a configuration file. The CODEC is responsible for parsing the data and hydrating a new BreakpointOperation object.
-   **Scope:** The object's lifetime is bound to the parent BrushConfig that contains it. It persists in memory as part of the brush's operation list for as long as that brush definition is loaded. It is effectively a stateless data container between executions.
-   **Destruction:** The object is eligible for garbage collection when its parent BrushConfig is unloaded or replaced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The internal state of a BreakpointOperation (label, printMessage, condition, etc.) is configured upon deserialization from a data file. This state is treated as immutable during the object's lifetime. The operation itself is stateful only in the context of a single execution, as it reads the external, mutable state of the BrushConfig and BrushConfigCommandExecutor.
-   **Thread Safety:** This class is **not thread-safe**. Its primary method, modifyBrushConfig, is designed to be called from a single thread, typically the main server thread that manages the BrushConfigCommandExecutor. It directly interacts with and modifies the state of the executor and accesses entity components, which are not safe for concurrent access.

    **Warning:** Calling modifyBrushConfig from any thread other than the one managing the brush execution will lead to race conditions, inconsistent state, and potential server instability.

## API Surface
The public contract is dominated by the inherited execution method and the static CODEC used for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | Executes the breakpoint logic. Checks if breakpoints are enabled, evaluates the optional condition, and if triggered, outputs debug information and potentially pauses the executor. |
| CODEC | BuilderCodec | N/A | **Critical.** The static codec responsible for serializing and deserializing the operation from data files. This is the primary mechanism for creating instances. |

## Integration Patterns

### Standard Usage
A BreakpointOperation is not intended to be used directly in Java code. It is designed to be defined within a data file (e.g., JSON) that represents a scripted brush. The BrushConfigCommandExecutor is responsible for iterating through operations and executing them.

The following example shows the conceptual execution loop that would invoke this operation.

```java
// Conceptual code within BrushConfigCommandExecutor
BrushConfig config = loadBrushFromDisk("my_debug_brush.json");
BrushConfigCommandExecutor executor = new BrushConfigCommandExecutor(config);

// The executor's run loop implicitly calls the operation
executor.executeNextOperation(); // If this operation is a Breakpoint, its logic runs here
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BreakpointOperation()`. The object will be unconfigured and will not function correctly. All instances must be created via the data-driven CODEC system.
-   **External Invocation:** Do not call `modifyBrushConfig` from outside the managed lifecycle of a BrushConfigCommandExecutor. The method relies on the executor's internal state (e.g., current operation index, debug stepping mode) to function correctly.

## Data Pipeline
The BreakpointOperation acts as a conditional gate in a control flow rather than a step in a data transformation pipeline. Its primary function is to generate side effects (logging, state changes) based on the current execution context.

> Flow:
> BrushConfigCommandExecutor Loop -> **BreakpointOperation.modifyBrushConfig()** -> Evaluate Condition -> Path Selection
>
> **Path A (Condition False or Breakpoints Disabled):**
> No-op -> Executor proceeds to next operation
>
> **Path B (Condition True and Breakpoints Enabled):**
> 1. Send Message -> Player Chat
> 2. Log Message -> Server Console
> 3. Update Executor State -> Enter Debug Step Mode

