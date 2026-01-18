---
description: Architectural reference for LoopRandomOperation
---

# LoopRandomOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol.loops
**Type:** Transient Component

## Definition
```java
// Signature
public class LoopRandomOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The LoopRandomOperation is a flow-control instruction within the Scripted Brush system. It does not directly modify the game world but instead manipulates the execution pointer of the brush itself. It functions as a conditional *jump* or *goto* statement, enabling sequences of operations to be repeated a random number of times.

This class is a concrete implementation of SequenceBrushOperation, meaning it is designed to be a single, stateful step in a larger, ordered list of brush instructions. Its primary role is to instruct the BrushConfigCommandExecutor to rewind its execution to a previously defined, named index. The number of times it performs this rewind is determined randomly at the start of the loop's execution, based on a configured min/max range.

The entire system is declarative. A user defines a brush's behavior, including loops, in a configuration file. The CODEC is responsible for deserializing this configuration into a sequence of operation objects, including instances of LoopRandomOperation.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by its static CODEC during the deserialization of a BrushConfig. This typically occurs when a server loads its brush definitions or when a player selects a scripted brush.
- **Scope:** The object instance persists as part of its parent BrushConfig for the duration of the server session or until the brush is redefined. However, its internal execution state, specifically the repetitionsRemaining field, is ephemeral and only relevant during a single, continuous application of the brush.
- **Destruction:** The object is garbage collected when its parent BrushConfig is unloaded. The resetInternalState method is invoked by the brush system between distinct brush applications to ensure that loop counters do not persist across separate user actions.

## Internal State & Concurrency
- **State:** Highly mutable. The core of its logic is the repetitionsRemaining field, which acts as a countdown timer for the loop. This field is initialized to an idle state (-1) and is actively modified during the execution of the modifyBrushConfig method. The configuration fields indexNameArg and repetitionsArg are set upon creation and are treated as immutable thereafter.

- **Thread Safety:** **This class is not thread-safe and must not be accessed concurrently.** It is designed to be operated by a single thread, typically the main server thread responsible for world modification. The internal state, repetitionsRemaining, is accessed and modified without any synchronization primitives. The use of ThreadLocalRandom further indicates an expectation of single-threaded execution within a given context. Concurrent calls to modifyBrushConfig would result in severe race conditions and unpredictable loop behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | The primary execution entry point. Decrements the loop counter and instructs the BrushConfigCommandExecutor to jump to the target index. |
| resetInternalState() | void | O(1) | Resets the internal loop counter to its idle state. Essential for ensuring a brush can be reused. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is designed to be defined declaratively within a brush configuration file and executed by the engine. The system interacts with it by iterating through a list of SequenceBrushOperation and calling modifyBrushConfig on each one.

```java
// CONCEPTUAL: How the BrushConfigCommandExecutor uses the operation.
// This code does not exist in this form but illustrates the pattern.

// 1. Executor retrieves the current operation in the sequence.
SequenceBrushOperation currentOp = brushConfig.getOperationAtIndex(currentIndex);

if (currentOp instanceof LoopRandomOperation) {
    // 2. The executor invokes the operation, passing a reference to itself.
    currentOp.modifyBrushConfig(entityStoreRef, brushConfig, this, accessor);

    // 3. The LoopRandomOperation, inside modifyBrushConfig, may call
    //    back into the executor to change the next instruction index.
    //    e.g., this.loadOperatingIndex("loop_start", false);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new LoopRandomOperation()`. The object will be unconfigured and will cause NullPointerExceptions or other undefined behavior. Always define operations via the brush configuration files so the CODEC can correctly initialize them.
- **State Tampering:** Do not manually call resetInternalState or modify the repetitionsRemaining field while a brush is being executed. This will corrupt the state of the loop and lead to either infinite loops or premature termination.
- **Asynchronous Execution:** Never invoke modifyBrushConfig from a separate thread. All brush operations must be executed sequentially on the main server thread to prevent world corruption and race conditions.

## Data Pipeline
LoopRandomOperation is a **control-flow component**, not a data-processing one. It does not transform data but rather redirects the flow of execution.

> Flow:
> BrushConfigCommandExecutor executes current instruction -> Instruction is **LoopRandomOperation** -> `modifyBrushConfig` is called -> **LoopRandomOperation** checks internal counter -> If counter > 0, calls `BrushConfigCommandExecutor.loadOperatingIndex` -> BrushConfigCommandExecutor's internal instruction pointer is reset to an earlier index.

