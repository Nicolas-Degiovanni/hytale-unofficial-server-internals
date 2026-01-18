---
description: Architectural reference for PersistentDataOperation
---

# PersistentDataOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.saveandload
**Type:** Transient Command

## Definition
```java
// Signature
public class PersistentDataOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The PersistentDataOperation is a concrete implementation of the Command Pattern, designed to operate within the Scripted Brush system. Its primary function is to provide stateful behavior to otherwise stateless brush execution sequences. It allows a brush to maintain and manipulate simple integer variables that persist between distinct uses of the tool.

This class acts as a data-driven command object. It is not intended for direct, procedural instantiation in code. Instead, it is defined declaratively within a brush configuration file and brought to life by the engine's serialization system (Codec).

Architecturally, it decouples the definition of a stateful operation (e.g., "add 5 to the variable named *height*") from the execution context that manages the state itself. The PersistentDataOperation holds the *what* (the variable name, the operation, the modifier), while the BrushConfigCommandExecutor, provided at runtime, holds the *actual state* (the map of variable names to integer values). This makes the operation itself reusable and stateless.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the static `CODEC` field during the deserialization of a Scripted Brush configuration file. The engine reads the brush definition and uses the codec to construct a corresponding in-memory graph of operation objects.
-   **Scope:** The lifetime of a PersistentDataOperation instance is bound to its parent BrushConfig. It persists in memory as long as the brush definition is loaded.
-   **Destruction:** The object is marked for garbage collection when its parent BrushConfig is unloaded, for example, when a game asset pack is reloaded or the server shuts down. There is no manual destruction method.

## Internal State & Concurrency
-   **State:** The internal state of this object (variableNameArg, operationArg, modifierArg) is **effectively immutable after deserialization**. While the fields are not declared final, they are populated once by the CODEC and are not designed to be modified at runtime. This class does not manage any runtime state itself; it is a pure operator on state held externally.

-   **Thread Safety:** This class is **not thread-safe** and must not be treated as such. The `modifyBrushConfig` method is designed to be executed on a single, controlled game thread (e.g., the main server thread). Its collaborator, BrushConfigCommandExecutor, does not employ synchronization for its internal state, and concurrent calls to `modifyBrushConfig` will result in race conditions and unpredictable behavior.

    **WARNING:** Any attempt to execute brush operations from multiple threads will corrupt brush state.

## API Surface
The public API is minimal, exposing only the execution entry point. Configuration is handled declaratively via the CODEC system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, config, executor, accessor) | void | O(1) | Executes the defined operation. Reads the target variable from the executor, applies the arithmetic modification, and writes the new value back. This is the primary entry point for the command. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly from Java code. It is configured declaratively within a data file that defines a scripted brush. The engine handles its lifecycle and execution.

The following conceptual example illustrates how this operation would be defined in a brush configuration file.

```yaml
# Example Brush Definition (Conceptual)
name: "Incremental Tower Brush"
sequence:
  - type: "PersistentDataOperation"
    StoredName: "current_height"
    Operation: "ADD"
    Modifier: 1
  - type: "PlaceBlockOperation"
    block: "stone"
    offset: { y: "@{current_height}" } # Use the variable in a subsequent operation
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new PersistentDataOperation()`. The object will be unconfigured and will not be part of any execution pipeline. The system relies entirely on the `CODEC` for proper initialization.
-   **Runtime State Mutation:** Do not modify the public fields (`variableNameArg`, `operationArg`, `modifierArg`) after the object has been deserialized. This violates the declarative nature of the system and can lead to unpredictable behavior.
-   **External Execution:** Do not acquire an instance of this class and call `modifyBrushConfig` from your own code. It must only be executed by the Scripted Brush system, which provides the correct stateful `BrushConfigCommandExecutor` context.

## Data Pipeline
PersistentDataOperation acts as a controller within a control-flow pipeline, not a data-transformation pipeline. It modifies a shared state context that is then read by subsequent operations in the sequence.

> Flow:
> Brush Config File (YAML/JSON) -> **Hytale Codec Deserializer** -> In-Memory `PersistentDataOperation` Instance -> Brush Execution Engine -> `modifyBrushConfig` call -> **BrushConfigCommandExecutor (State is Mutated)** -> Subsequent Brush Operation (Reads Mutated State)

