---
description: Architectural reference for JumpIfClickType
---

# JumpIfClickType

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol
**Type:** Transient Operation

## Definition
```java
// Signature
public class JumpIfClickType extends SequenceBrushOperation {
```

## Architecture & Concepts
The JumpIfClickType class is a fundamental flow-control instruction within the Scripted Brush system. It functions as a conditional jump, analogous to a `GOTO` or `JMP` instruction in low-level programming, enabling dynamic execution paths based on user input.

Its sole responsibility is to inspect the current interaction type (e.g., a primary or secondary mouse click) stored within the active BrushConfig and, if it matches a predefined condition, command the BrushConfigCommandExecutor to alter its internal execution pointer. This redirects the flow of subsequent brush operations to a labeled index within the operation sequence.

This class is never executed in isolation. It is designed to be a single step in a larger, ordered list of SequenceBrushOperation objects that collectively define a brush's behavior. The static CODEC field is a critical architectural element, indicating that instances are created via data-driven deserialization from a brush's configuration file, not through direct instantiation in game logic.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the engine's Codec framework when a Scripted Brush configuration is loaded into memory. The framework uses the static CODEC definition to parse the configuration data and construct the object.
-   **Scope:** The object's lifetime is bound to its parent BrushConfig. It persists as part of the in-memory representation of the brush's operation list and is effectively stateless between individual executions.
-   **Destruction:** The object is marked for garbage collection when the parent BrushConfig is unloaded, such as when a server shuts down or a user deselects the brush.

## Internal State & Concurrency
-   **State:** The internal state, consisting of `indexVariableNameArg` and `clickTypeArg`, is **immutable** post-deserialization. The object serves as a read-only data carrier for its execution logic and holds no mutable runtime data.

-   **Thread Safety:** This class is inherently thread-safe. Its immutable state and lack of side-effects within its own scope ensure that it can be safely processed. The overarching Scripted Brush system is responsible for ensuring that the `modifyBrushConfig` method is not invoked concurrently for the same brush instance, thereby guaranteeing the safety of the mutable parameters passed to it.

## API Surface
The public contract is focused entirely on its role as an operation within the brush execution loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Evaluates the current interaction type. If it matches the configured `clickTypeArg`, instructs the executor to jump to the operation index specified by `indexVariableNameArg`. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly through Java code. Instead, it is declared within a brush's data configuration file. The engine's Scripted Brush system parses this configuration and executes the operation.

The following pseudo-configuration demonstrates its intended use:

```yaml
# Example brush_definition.hocon
operations = [
    # ... other operations
    {
        # This operation checks the click type.
        type: "JumpIfClickType",
        ClickType: "Left",
        StoredIndexName: "OnLeftClickBehavior"
    },
    {
        # This block executes ONLY on a right-click, because the
        # JumpIfClickType instruction was not triggered.
        type: "PaintBlock",
        block: "hytale:stone"
    },
    {
        # Unconditionally jump to the end to avoid falling through.
        type: "Jump",
        StoredIndexName: "EndExecution"
    },
    {
        # A label that the first operation can jump to.
        label: "OnLeftClickBehavior",
        type: "EraseBlock"
    },
    {
        label: "EndExecution",
        type: "NoOp"
    }
    # ... other operations
]
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new JumpIfClickType()`. The object will lack the necessary configuration and will not be registered within an execution sequence, rendering it useless. All instances must be created via the engine's CODEC deserializer.
-   **Runtime State Mutation:** Do not attempt to modify the `indexVariableNameArg` or `clickTypeArg` fields after the object has been created. These are intended to be immutable configuration values. Modifying them at runtime leads to unpredictable and non-deterministic brush behavior.

## Data Pipeline
This component primarily manipulates the control flow, not a data stream. The flow represents the sequence of events leading to a change in the brush executor's instruction pointer.

> Flow:
> User Input (Primary/Secondary Click) -> Engine captures as `InteractionType` -> `BrushConfig` is updated with InteractionType -> `BrushConfigCommandExecutor` invokes current operation -> **JumpIfClickType.modifyBrushConfig()** reads InteractionType -> `BrushConfigCommandExecutor.loadOperatingIndex()` is called -> Execution pointer is moved to a new operation.

