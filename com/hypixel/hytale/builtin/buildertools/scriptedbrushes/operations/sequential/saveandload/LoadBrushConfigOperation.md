---
description: Architectural reference for LoadBrushConfigOperation
---

# LoadBrushConfigOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.saveandload
**Type:** Transient

## Definition
```java
// Signature
public class LoadBrushConfigOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The LoadBrushConfigOperation is a data-driven command object that represents a single, discrete step within the Scripted Brush system. Its primary architectural function is to encapsulate the parameters required to restore a previously saved brush configuration, known as a snapshot.

This class adheres to a Command Pattern. It does not contain the logic for loading the configuration itself. Instead, it holds the *intent* and *parameters* for the operation—specifically, the name of the snapshot to load and which parts of it to apply. The actual work is delegated to the `BrushConfigCommandExecutor` during execution. This design decouples the definition of a brush sequence (which can be serialized from a file) from the runtime execution environment.

The static `CODEC` field is the most critical architectural feature. It integrates this class into Hytale's data serialization framework, allowing engine systems to instantiate and configure `LoadBrushConfigOperation` objects directly from game data files without hardcoded logic.

## Lifecycle & Ownership
- **Creation:** This object is almost exclusively instantiated by the engine's `Codec` system during the deserialization of a scripted brush definition. Manual instantiation is strongly discouraged as it bypasses the intended data-driven workflow.
- **Scope:** The object is ephemeral. Its lifetime is scoped to the execution of a single step within a brush's operational sequence. It exists only to be configured and then immediately executed.
- **Destruction:** Once the `modifyBrushConfig` method completes, the object has served its purpose and is eligible for garbage collection. It holds no persistent state and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state consists of `variableNameArg` and `dataSettingFlagArg`. This state is mutable upon creation and during the deserialization process. During its execution via `modifyBrushConfig`, this state should be considered read-only.
- **Thread Safety:** This class is **not thread-safe**. The public fields can be modified from any thread. However, the Scripted Brush system is designed to execute operations sequentially on a single, controlled thread. Concurrency issues will only arise if this class is used outside of its intended framework.

**WARNING:** Modifying the state of this object from another thread while it is being executed will result in undefined and likely catastrophic behavior for the brush system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(N) | Delegates to the provided executor to load a named snapshot. Complexity depends on the executor's implementation and the number of parameters (N) being loaded. |

## Integration Patterns

### Standard Usage
This operation is not intended to be invoked directly in Java code. It is designed to be defined within a data file (e.g., a JSON asset) and executed by the Scripted Brush engine. The engine is responsible for providing the correct context and executor.

A conceptual Java-based invocation would look like this:

```java
// Engine-level code, not typical user code.
// Assume 'operation' is an instance deserialized from a config file.
LoadBrushConfigOperation operation = ...;

// The engine provides the context and executor at runtime.
BrushConfigCommandExecutor executor = getExecutorForPlayer(player);
BrushConfig targetConfig = getActiveBrushConfig(player);

// The engine invokes the operation on the target config.
operation.modifyBrushConfig(entityStoreRef, targetConfig, executor, accessor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new LoadBrushConfigOperation()`. The object will be unconfigured and useless. It must be created and populated by the engine's deserialization pipeline to be valid.
- **State Mutation:** Do not modify `variableNameArg` or `dataSettingFlagArg` after the operation has been passed to the execution engine. The state is considered sealed once execution begins.
- **Executor Misuse:** This class relies entirely on the `BrushConfigCommandExecutor` for its logic. Attempting to use it without a valid, fully-initialized executor will lead to system instability or no-op behavior.

## Data Pipeline
The primary flow for this component is from a static data definition to a runtime state change.

> Flow:
> Brush Script Asset (JSON/NBT) -> Engine `Codec` Deserializer -> **LoadBrushConfigOperation Instance** -> Scripted Brush Executor -> `modifyBrushConfig` Call -> `BrushConfigCommandExecutor` -> Live `BrushConfig` State is updated

