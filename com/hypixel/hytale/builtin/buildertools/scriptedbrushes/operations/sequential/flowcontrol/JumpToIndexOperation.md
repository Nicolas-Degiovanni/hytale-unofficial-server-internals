---
description: Architectural reference for JumpToIndexOperation
---

# JumpToIndexOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol
**Type:** Transient Command Object

## Definition
```java
// Signature
public class JumpToIndexOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The JumpToIndexOperation is a concrete implementation of the Command Pattern, designed to introduce non-linear control flow into the Scripted Brush system. It functions as a `GOTO` statement, altering the execution sequence of brush operations.

Architecturally, this class is a node in a sequence of operations managed by a BrushConfig. Its sole responsibility is to instruct the BrushConfigCommandExecutor—the effective "virtual machine" for scripted brushes—to change its internal program counter. It directs the executor to jump to a previously labeled instruction index.

This operation is meaningless in isolation. It relies on a corresponding operation, such as a StoreIndexOperation, to have previously saved an execution index under a named label. The JumpToIndexOperation then uses this named label, stored in its `variableNameArg` field, to look up the target index and perform the jump. This enables the creation of loops and conditional branches within a declarative brush script.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale `CODEC` system during the deserialization of a BrushConfig. A game designer or developer defines this operation declaratively in a data file (e.g., JSON), and the static `CODEC` field handles its instantiation and configuration from that data.
- **Scope:** The object's lifetime is bound to the parent BrushConfig that contains it. It is a stateless, value-like component within a larger configuration graph.
- **Destruction:** The object is marked for garbage collection when its parent BrushConfig is unloaded or destroyed. It holds no external resources that require explicit cleanup.

## Internal State & Concurrency
- **State:** Effectively immutable after deserialization. The `variableNameArg` field is populated once by the `CODEC` and is not intended to be modified at runtime. The operation itself is stateless; its execution produces a side effect on the state of the BrushConfigCommandExecutor.
- **Thread Safety:** This object is inherently thread-safe due to its immutable nature. However, the context in which it operates is **not**. All brush operations modifying a world or a BrushConfig state must be executed on a single, designated thread (typically the main server thread) to prevent severe race conditions and world corruption. The BrushConfigCommandExecutor is not designed for concurrent execution.

## API Surface
The public contract is minimal, consisting of the execution method called by the brush system's interpreter.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Instructs the provided BrushConfigCommandExecutor to modify its internal execution index to the one associated with `variableNameArg`. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation in user code. It is executed by the BrushConfigCommandExecutor as part of a sequence. The following conceptual example illustrates how the executor would process it.

```java
// Executed internally by the Scripted Brush engine
BrushConfigCommandExecutor executor = getActiveExecutor();
JumpToIndexOperation jumpOp = configuredJumpOperation; // Loaded from config

// The engine invokes the operation
jumpOp.modifyBrushConfig(entityRef, config, executor, accessor);

// The executor's internal program counter is now changed
executor.proceedToNextOperation(); // Will now execute from the new index
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new JumpToIndexOperation()`. The object will be unconfigured and useless. It must be instantiated and configured declaratively through the `CODEC` system when defining a brush.
- **Runtime Modification:** Do not attempt to modify the `variableNameArg` field after the object has been created. This violates the declarative design of the system and can lead to unpredictable behavior.
- **Jumping to an Unsaved Index:** Executing this operation with a `variableNameArg` that does not correspond to a previously stored index is an error. This will cause the BrushConfigCommandExecutor to fail, halting the brush script.

## Data Pipeline
As a control-flow operation, this class does not process data. Instead, it redirects the flow of execution.

> Flow:
> Brush Configuration File (JSON/Data) -> Hytale CODEC Deserializer -> **JumpToIndexOperation Instance** -> BrushConfigCommandExecutor invokes `modifyBrushConfig` -> Executor's Internal Program Counter is updated -> Execution continues from new index

