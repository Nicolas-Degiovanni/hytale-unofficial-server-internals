---
description: Architectural reference for JumpIfToolArgOperation
---

# JumpIfToolArgOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol
**Type:** Transient / Data Object

## Definition
```java
// Signature
public class JumpIfToolArgOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The JumpIfToolArgOperation is a flow-control instruction within the Scripted Brush execution engine. It functions as a conditional jump, analogous to an *if-goto* statement in lower-level programming. Its primary role is to alter the linear execution of a brush's operational sequence based on the current configuration of a player's held Builder Tool.

This class acts as a critical bridge between the player's user interface choices (e.g., toggling a checkbox, selecting a dropdown option on a tool's property panel) and the server-side logic of the brush. By inspecting the arguments stored on the active tool's ItemStack, it enables the creation of dynamic, multi-modal brushes whose behavior can be altered in real-time by the player without needing to switch to a different brush asset.

This operation is fundamental for building complex, non-linear build scripts that can react to user input, making builder tools significantly more powerful and versatile.

### Lifecycle & Ownership
-   **Creation:** Instances are not created programmatically via the *new* keyword. They are instantiated by the Hytale codec system during server startup or asset loading. The static CODEC field defines how to deserialize this object from a brush's configuration file (e.g., a JSON or HOCON asset).
-   **Scope:** The object's lifetime is bound to its parent BrushConfig. It persists in memory as part of a brush's definition as long as that brush asset is loaded.
-   **Destruction:** The object is eligible for garbage collection when its containing BrushConfig is unloaded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The internal state consists of four configuration fields: argNameArg, comparisonTypeArg, comparisonValueArg, and indexVariableNameArg. This state is populated once during deserialization and should be considered **immutable** during runtime execution. The object is a pure data-holder for the operation's parameters.
-   **Thread Safety:** This class is **not thread-safe**. The core logic in modifyBrushConfig accesses and manipulates live game state, including player components and the BrushConfigCommandExecutor. It must be executed exclusively on the main server thread that manages world updates to prevent race conditions, data corruption, and concurrency exceptions.

## API Surface
The public contract is exclusively for internal use by the Scripted Brush system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Executes the conditional jump logic. Reads the specified argument from the player's active tool, performs a comparison, and instructs the executor to change its program counter if the condition is met. Sets an error flag on the BrushConfig if the tool, item, or argument cannot be resolved. |

## Integration Patterns

### Standard Usage
This operation is not intended to be used directly in Java code. It is designed to be defined declaratively within a scripted brush asset file. The server's codec system will then parse this definition to create the object instance.

A conceptual configuration might look like this:

```json
// Example brush asset definition
{
  "name": "Conditional Placement Brush",
  "operations": [
    {
      "type": "Label",
      "Name": "Start"
    },
    {
      "type": "JumpIfToolArg",
      "ArgName": "use_special_block",
      "ComparisonType": "Equals",
      "ComparisonValue": "true",
      "StoredIndexName": "PlaceSpecial"
    },
    // ... operations for standard block placement
    {
      "type": "Jump",
      "StoredIndexName": "End"
    },
    {
      "type": "Label",
      "Name": "PlaceSpecial"
    },
    // ... operations for special block placement
    {
      "type": "Label",
      "Name": "End"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new JumpIfToolArgOperation()`. An unconfigured instance created this way will lack the necessary arguments to function and will likely cause errors or unpredictable behavior. Always define operations within asset files.
-   **State Mutation:** Do not modify the public fields (e.g., argNameArg) after the brush has been loaded. The state of an operation is considered immutable post-deserialization.
-   **Off-Thread Execution:** Never invoke modifyBrushConfig from an asynchronous task or a different thread. All brush operations must be executed synchronously on the main server thread.

## Data Pipeline
The execution of this operation follows a clear data-retrieval and control-flow pattern.

> Flow:
> BrushConfigCommandExecutor invokes operation -> **JumpIfToolArgOperation.modifyBrushConfig** -> Accesses Player Component via ComponentAccessor -> Reads active ItemStack from Player Inventory -> Resolves BuilderTool asset -> Decodes tool arguments from ItemStack -> Performs comparison logic -> If true, calls **BrushConfigCommandExecutor.loadOperatingIndex** -> Executor's internal program counter is updated -> Execution continues from new index.

