---
description: Architectural reference for SaveBrushConfigOperation
---

# SaveBrushConfigOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.saveandload
**Type:** Transient

## Definition
```java
// Signature
public class SaveBrushConfigOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The SaveBrushConfigOperation is a concrete implementation of the Command Pattern, representing a single, serializable action within the Scripted Brush system. It does not contain complex logic itself; instead, it serves as a data-driven instruction that holds a single parameter: the name under which to save a snapshot of the current brush state.

Its primary architectural role is to decouple the definition of a brush's behavior from the execution logic. This class is designed to be instantiated by the engine's codec system from a data file (e.g., a JSON asset defining a brush). The static CODEC field is the key to this system, providing the blueprint for deserializing a configuration block into a valid SaveBrushConfigOperation object.

During the execution of a brush sequence, this operation delegates the actual snapshotting work to the BrushConfigCommandExecutor, which manages the state and storage of brush configurations.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the Hytale codec framework via its static CODEC field when a scripted brush asset is loaded. It is never intended to be constructed manually in game logic.
- **Scope:** The object's lifetime is bound to the parent scripted brush definition. It exists as one command in a sequence of operations.
- **Destruction:** The instance is eligible for garbage collection when the brush asset it belongs to is unloaded from memory.

## Internal State & Concurrency
- **State:** Mutable. The core state is the public String field, variableNameArg. This field is populated by the codec during deserialization. The object itself is a simple data container.
- **Thread Safety:** This class is not thread-safe and must not be accessed from multiple threads. It is designed to be created, configured, and executed exclusively on the server's main game thread as part of a brush execution sequence. No synchronization primitives are used.

## API Surface
The public API is minimal, consisting primarily of the method required to fulfill the SequenceBrushOperation contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(N) | Instructs the BrushConfigCommandExecutor to create and store a deep copy of the provided BrushConfig. The complexity N is proportional to the size of the BrushConfig object being snapshotted. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly from Java code. Instead, it is defined declaratively within a brush asset file. The engine's systems handle its lifecycle and execution.

A conceptual representation in a configuration file would look like this:

```yaml
# Example of a brush definition asset
sequence: [
    {
        # The engine uses the 'type' to find the correct codec
        type: "SaveBrushConfig",

        # The codec uses this key to populate the variableNameArg field
        StoredName: "my_brush_snapshot"
    },
    # ... other operations
]
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new SaveBrushConfigOperation()`. The object will be unconfigured and will not function correctly within the brush system. The engine must instantiate it via its codec to ensure proper state.
- **Manual Invocation:** Avoid calling the `modifyBrushConfig` method directly. The parent SequenceBrushOperation is responsible for invoking its child operations in the correct order and with the correct execution context.

## Data Pipeline
The primary function of this class is to trigger a data snapshotting process. The data does not flow *through* this object, but rather this object acts as the trigger for the flow.

> Flow:
> Brush Asset File -> Engine Codec System -> **SaveBrushConfigOperation Instance** -> Brush Sequence Executor -> `modifyBrushConfig` Invocation -> BrushConfigCommandExecutor -> In-Memory Snapshot Map (String -> BrushConfig)

