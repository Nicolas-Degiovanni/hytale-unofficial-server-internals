---
description: Architectural reference for JumpIfBlockTypeOperation
---

# JumpIfBlockTypeOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol
**Type:** Transient Command Object

## Definition
```java
// Signature
public class JumpIfBlockTypeOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The JumpIfBlockTypeOperation is a fundamental flow-control instruction within the Scripted Brush subsystem. It functions as a conditional jump, analogous to a GOTO statement in lower-level programming, enabling non-linear execution paths for complex building and terraforming tasks.

This class is not a service or a manager; it is a pure data-driven command. Its primary role is to encapsulate the logic for a single conditional check: "Does the block at a specific offset from the current brush origin match a given BlockMask?" If the condition is met, it instructs the brush executor to alter its instruction pointer, jumping to a predefined labeled index within the sequence of operations.

Instances of this class are not created manually in code. Instead, they are deserialized from data files (e.g., JSON definitions of a brush) via the static CODEC field. This declarative approach allows designers and developers to construct intricate brush behaviors without writing Java code. The operation is entirely dependent on the context provided by a BrushConfigCommandExecutor, which manages the world state, the current brush position, and the execution index.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale Codec system during the deserialization of a Scripted Brush definition. The public constructor exists to satisfy the reflection requirements of the BuilderCodec, not for direct developer use.

-   **Scope:** The object's lifetime is extremely short and is scoped to a single execution of a brush script. It is held within a list of SequenceBrushOperation objects inside a parent configuration.

-   **Destruction:** The object is eligible for garbage collection as soon as the brush execution completes and the parent BrushConfig is no longer referenced. It holds no persistent resources that require manual cleanup.

## Internal State & Concurrency
-   **State:** The internal state of this class is **Mutable**. Its public fields, such as offsetListArg and blockMaskArg, are populated directly by the codec upon creation. This state is considered "write-once" and should not be modified after deserialization. During execution, the state is read-only.

-   **Thread Safety:** This class is **NOT thread-safe**. The public mutable fields and lack of synchronization primitives make it inherently unsafe for concurrent access. The Scripted Brush execution engine **must** guarantee that all operations on a given brush occur on a single thread in a strictly sequential manner. Any attempt to execute brush logic in parallel will result in race conditions and catastrophic world corruption.

## API Surface
The primary contract is the inherited modifyBrushConfig method, which is invoked by the brush executor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(N) | Executes the block check. N is the number of offsets. If any offset matches the BlockMask, it commands the executor to jump to the stored index name. Throws errors if the brush origin is not set. |

## Integration Patterns

### Standard Usage
This operation is not intended to be invoked directly. It is defined declaratively as part of a larger sequence and processed by an executor. The following conceptual example illustrates how the system processes it.

```java
// Conceptual executor loop
BrushConfigCommandExecutor executor = getExecutor();
BrushConfig config = executor.getConfig();

// The executor retrieves the current operation from its list
JumpIfBlockTypeOperation operation = (JumpIfBlockTypeOperation) config.getOperationAtIndex(executor.getCurrentIndex());

// The executor invokes the operation, passing in the required context
operation.modifyBrushConfig(worldRef, config, executor, componentAccessor);

// The operation may have modified the executor's internal index,
// causing the next iteration of the loop to fetch a different operation.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new JumpIfBlockTypeOperation()`. The object will be in an invalid state with null fields, as it bypasses the codec-based initialization. This will cause immediate NullPointerExceptions when `modifyBrushConfig` is called.

-   **Out-of-Context Execution:** Calling `modifyBrushConfig` manually is an error. The method relies entirely on the state managed by the BrushConfig and BrushConfigCommandExecutor arguments. Calling it with mock or null objects will lead to unpredictable behavior.

-   **State Mutation:** Do not modify the public fields (offsetListArg, blockMaskArg, indexVariableNameArg) after the object has been created by the codec. This state is considered immutable post-initialization.

## Data Pipeline
The flow of data and control for this operation is linear and reactive. It reads from the world state to make a decision that influences the control flow of its parent executor.

> Flow:
> Brush Definition (Data File) -> Hytale Codec -> **JumpIfBlockTypeOperation Instance** -> BrushConfigCommandExecutor invokes `modifyBrushConfig` -> Reads World State via `getEdit()` -> Compares against BlockMask -> If match, calls `executor.loadOperatingIndex()` -> Executor updates its internal instruction pointer.

