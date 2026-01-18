---
description: Architectural reference for RunCommandOperation
---

# RunCommandOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient

## Definition
```java
// Signature
public class RunCommandOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The RunCommandOperation is a concrete implementation of the Command Pattern, designed to operate within the Scripted Brush framework. It functions as a single, serializable step in a sequence of brush operations, enabling world modification tools to execute server commands.

Architecturally, this class serves as a critical bridge between a declarative configuration system and the server's imperative command execution engine. Its primary design feature is the static **CODEC** field. This is a standard Hytale engine pattern where components, entities, and operations are defined as data (typically in JSON or a similar format) and are deserialized into live Java objects at runtime. The CODEC for RunCommandOperation defines how to construct an instance from a configuration file, specifically mapping the `CommandToRun` key to the internal `commandArg` field.

This class encapsulates the logic for dynamic string substitution, transforming a command template into a fully-formed command string using runtime context provided by the active BrushConfig and BrushConfigCommandExecutor.

## Lifecycle & Ownership
- **Creation:** RunCommandOperation is instantiated exclusively by the engine's serialization system via its static **CODEC**. Developers must not instantiate this class directly using the `new` keyword. It is created when a scripted brush definition containing a `runCommand` operation is loaded from an asset file.
- **Scope:** The object is short-lived. An instance is created as part of a larger brush definition structure. It is held by the parent brush configuration but is effectively used for a single invocation of `modifyBrushConfig` during a brush application.
- **Destruction:** The object holds no native resources and does not require explicit cleanup. It is managed entirely by the Java Garbage Collector and becomes eligible for collection after the brush operation completes.

## Internal State & Concurrency
- **State:** The primary state is the `commandArg` string, which contains the command template. This state is set once upon deserialization and is treated as immutable. The class is stateless concerning its execution; it does not modify its own fields during the `modifyBrushConfig` call.
- **Thread Safety:** This class is **not thread-safe** and must only be used on the main server thread. The `modifyBrushConfig` method interacts directly with the global CommandManager and accesses entity components, which are not designed for concurrent access. Calling this method from a worker thread will lead to race conditions and severe server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(N) | Processes and executes the configured command. N is the length of the command string. Throws NullPointerException if the player component is missing. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in typical game logic code. It is designed to be defined in a data file and executed by the Scripted Brush system. The following example illustrates how the engine's `BrushConfigCommandExecutor` would invoke this operation.

```java
// Engine-level code within the BrushConfigCommandExecutor
// operations is a List<SequenceBrushOperation> loaded from config

for (SequenceBrushOperation operation : operations) {
    // The engine dispatches the call to the correct operation type
    operation.modifyBrushConfig(entityRef, currentBrushConfig, this, componentAccessor);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new RunCommandOperation()`. The object will be in an invalid state without its `commandArg` field being set by the codec system. This will result in no command being executed.
- **Off-Thread Execution:** Never invoke `modifyBrushConfig` from an asynchronous task or a separate thread. All interactions with the CommandManager and entity components must be synchronized with the main server tick.
- **State Mutation:** Do not attempt to retrieve and modify the `commandArg` field via reflection after initialization. The operation's definition is expected to be static once loaded.

## Data Pipeline
The flow of data from configuration to execution is a key aspect of this system's design.

> Flow:
> Brush Asset File (JSON) -> Engine Codec System -> **RunCommandOperation Instance** -> BrushConfigCommandExecutor -> `modifyBrushConfig` Invocation -> CommandManager -> Server Command Execution

