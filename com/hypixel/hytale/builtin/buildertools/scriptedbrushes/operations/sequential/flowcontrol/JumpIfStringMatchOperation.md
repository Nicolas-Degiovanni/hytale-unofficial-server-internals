---
description: Architectural reference for JumpIfStringMatchOperation
---

# JumpIfStringMatchOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol
**Type:** Transient

## Definition
```java
// Signature
public class JumpIfStringMatchOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The JumpIfStringMatchOperation is a command object that implements flow control within the Scripted Brush system. It functions as a conditional *goto* statement, allowing a sequence of brush operations to branch based on string equality. This enables the creation of complex, state-driven building tools and macros without requiring a full-fledged embedded scripting language.

Architecturally, this class is a single, self-contained instruction in a larger sequence managed by a `BrushConfigCommandExecutor`. When a scripted brush is executed, the executor iterates through a list of `SequenceBrushOperation` instances. When it encounters a JumpIfStringMatchOperation, it delegates execution to its `modifyBrushConfig` method. The operation then performs its logic and, if its condition is met, instructs the executor to change its internal instruction pointer to a new, labeled index within the sequence.

This pattern decouples the flow control logic from the executor itself, allowing new conditional operations to be added to the system without modifying the core execution loop.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly via the `new` keyword. They are deserialized and instantiated by the Hytale `CODEC` system when a `BrushConfig` is loaded from storage. The static `CODEC` field defines how to map configuration data (e.g., from a JSON file) to the object's fields.
-   **Scope:** The object's lifetime is bound to its parent `BrushConfig`. It persists in memory as part of the brush's definition for as long as that brush is loaded.
-   **Destruction:** The object is marked for garbage collection when its containing `BrushConfig` is unloaded or redefined. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The object's state (`indexVariableNameArg`, `sideOneArg`, `sideTwoArg`) is mutable upon creation by the `CODEC` system. After initialization, this state is intended to be treated as immutable for the remainder of its lifecycle. The core execution logic in `modifyBrushConfig` reads this state but does not alter it.

-   **Thread Safety:** This class is **not thread-safe** and must only be used from the main server thread. Brush operations are executed sequentially within the server's tick loop. The parameters passed to `modifyBrushConfig`, such as `Ref<EntityStore>`, provide access to core world state that is not protected for concurrent access. Any attempt to execute this operation from an asynchronous task will lead to race conditions and world corruption.

## API Surface
The public API is minimal, consisting primarily of the override for the execution contract. The public fields are an implementation detail of the `CODEC` system and should not be accessed directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | Executes the conditional jump logic. Compares its internal string arguments and instructs the `BrushConfigCommandExecutor` to change its execution index if they match case-insensitively. |

## Integration Patterns

### Standard Usage
This operation is not used directly in Java code. It is defined declaratively as part of a brush's configuration data. The engine's `CODEC` system parses this data to create the object instance.

A conceptual configuration might look like this:

```yaml
# This is a conceptual representation, not actual Hytale config format.
operations:
  - type: StoreString
    name: "block_type"
    value: "stone"
  - type: JumpIfStringMatch
    StoredIndexName: "handle_stone"
    LeftSideOfStatement: "var:block_type"
    RightSideOfStatement: "stone"
  - type: PlaceBlock
    block: "dirt"
  - type: Jump
    StoredIndexName: "end"
  - type: Label # This is the target for the jump
    name: "handle_stone"
  - type: PlaceBlock
    block: "stone"
  - type: Label
    name: "end"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new JumpIfStringMatchOperation()`. The object will be unconfigured and useless. It must be instantiated by the `CODEC` system from a valid brush definition.
-   **Manual Execution:** Do not call the `modifyBrushConfig` method directly. This bypasses the state management and instruction pointer of the `BrushConfigCommandExecutor`, leading to incorrect and unpredictable behavior.
-   **Post-Creation State Mutation:** Modifying the public fields like `sideOneArg` after the brush has been loaded is not supported. This can break the intended logic of the brush script.

## Data Pipeline
This component primarily manipulates the *control flow* rather than a data stream. Its role in the pipeline is to redirect the execution path of its parent executor.

> Flow:
> `BrushConfigCommandExecutor` (executing operation at index N) -> Invokes `modifyBrushConfig` on **JumpIfStringMatchOperation** -> Internal string comparison evaluates to true -> **JumpIfStringMatchOperation** invokes `brushConfigCommandExecutor.loadOperatingIndex("label")` -> `BrushConfigCommandExecutor` updates its internal instruction pointer to index M -> Execution resumes at operation M in the next step.

