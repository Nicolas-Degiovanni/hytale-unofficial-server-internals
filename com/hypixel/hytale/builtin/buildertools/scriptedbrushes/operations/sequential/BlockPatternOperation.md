---
description: Architectural reference for BlockPatternOperation
---

# BlockPatternOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient

## Definition
```java
// Signature
public class BlockPatternOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The BlockPatternOperation is a concrete implementation of the Command Pattern, representing a single, serializable instruction within a larger scripted brush pipeline. Its sole responsibility is to encapsulate a BlockPattern and apply it to a BrushConfig during the execution of a brush sequence.

This class is a fundamental component of the server-side Builder Tools system. It enables the dynamic, data-driven configuration of building brushes. The static CODEC field is the cornerstone of this design, allowing the Hytale engine to deserialize this specific operation from a generic script or configuration file. This decouples the brush's behavior from its implementation, allowing designers and players to define complex brush logic without writing code.

It acts as a state modifier within a chain of operations. When a scripted brush is used, an executor iterates through its list of operations, calling modifyBrushConfig on each one in sequence. This class specifically targets the material or pattern aspect of the brush's configuration.

### Lifecycle & Ownership
- **Creation:** An instance is created by the BuilderCodec system when deserializing a scripted brush definition from a data source (e.g., a prefab asset). It is not intended for direct manual instantiation in typical gameplay code.
- **Scope:** The lifetime of a BlockPatternOperation instance is tied to its parent scripted brush definition. It is effectively a value object that exists as long as the script that contains it is loaded. During execution, its role is ephemeral—it is used once to modify the BrushConfig and then discarded until the next brush execution.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources or persistent state, so it is cleaned up once the parent brush definition is unloaded.

## Internal State & Concurrency
- **State:** The class holds a single, mutable state field: blockPatternArg. This field is populated during deserialization by the CODEC. After initialization, the object should be treated as immutable. It does not cache any data or maintain connections to other systems.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and executed exclusively on the main server thread that processes world interactions and builder tool actions. Accessing or modifying an instance from multiple threads will result in race conditions and undefined brush behavior.

## API Surface
The public API is minimal, focusing on the execution of its core task.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Applies the stored BlockPattern to the target BrushConfig. This is the primary entry point for the operation's logic and is called by the brush execution system. |

## Integration Patterns

### Standard Usage
This class is not meant to be invoked directly by developers. It is automatically managed and executed by the scripted brush system. A higher-level executor, such as a parent SequenceBrushOperation, iterates through a list of configured operations and applies them sequentially to a transient BrushConfig object.

```java
// Conceptual example of how a brush executor might use this operation.
// Developers do not write this code; it is part of the engine.

BrushConfig config = new BrushConfig();
List<SequenceBrushOperation> operations = scriptedBrush.getOperations();

// The executor iterates and applies each operation
for (SequenceBrushOperation op : operations) {
    // If 'op' is an instance of BlockPatternOperation, this call
    // will set the pattern on the 'config' object.
    op.modifyBrushConfig(entityStoreRef, config, commandExecutor, accessor);
}

// The fully modified 'config' is then used to perform the world edit.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid `new BlockPatternOperation()`. The class is designed to be configured via its CODEC from data files. Manual creation bypasses the intended data-driven workflow and can lead to improperly configured brushes.
- **State Mutation:** Do not modify the `blockPatternArg` field after the operation has been loaded into a brush sequence. Operations should be treated as immutable once configured to ensure predictable and repeatable brush behavior.

## Data Pipeline
The primary flow for this class involves deserialization from a data source, followed by execution as part of a larger command sequence.

> Flow:
> Brush Asset (Data File) -> BuilderCodec Deserialization -> **BlockPatternOperation Instance** -> Scripted Brush Executor -> `modifyBrushConfig` Call -> Live BrushConfig State Update

