---
description: Architectural reference for SaveIndexOperation
---

# SaveIndexOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.saveandload
**Type:** Transient Command

## Definition
```java
// Signature
public class SaveIndexOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The SaveIndexOperation is a foundational control-flow component within the Scripted Brush system. It does not directly modify the game world. Instead, its sole purpose is to act as a named "bookmark" or "label" within a sequence of brush operations.

During the pre-execution phase of a scripted brush, this operation captures its own numerical position (index) within the operation list and stores it in a variable map managed by the BrushConfigCommandExecutor. This stored index can then be retrieved by subsequent operations, such as a jump or loop command, to alter the normal sequential flow of execution.

This class is a critical enabler for creating complex brush logic, allowing for iterative behaviors and conditional branching without requiring direct script compilation. It is a pure state-management command that operates on the execution context, not the world state.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale `CODEC` system during the deserialization of a brush definition from a configuration file. The static `CODEC` field defines the mapping between the configuration data (e.g., a JSON or HOCON file) and the in-memory object.
-   **Scope:** The object's lifetime is bound to its parent BrushConfig. It exists as an immutable element within the brush's operation sequence for as long as the brush definition is loaded.
-   **Destruction:** The object is marked for garbage collection when its parent BrushConfig is unloaded or replaced. There is no manual destruction or cleanup required.

## Internal State & Concurrency
-   **State:** The internal state, specifically the `variableNameArg`, is mutable only during the initial deserialization process via the `CODEC`. After instantiation, the object is effectively immutable. It holds configuration data but does not cache runtime state.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking mechanisms. All brush operations are designed to be executed serially on a single, world-specific thread. Concurrent modification or access from multiple threads will lead to undefined behavior and state corruption within the BrushConfigCommandExecutor.

## API Surface
The public API is minimal, as the primary interaction is a callback from the brush execution system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| preExecutionModifyBrushConfig(executor, index) | void | O(1) | The core function. Injects the current operation index into the executor's state map under the key defined by `variableNameArg`. |

## Integration Patterns

### Standard Usage
This operation is not used directly in Java code by developers. Instead, it is declared within a brush's data definition file. The system then instantiates and executes it as part of the brush's lifecycle.

A conceptual brush definition might look like this:
```yaml
# Conceptual Brush Definition
name: "Looping Pillar Brush"
operations:
  - type: "SaveIndexOperation"
    StoredIndexName: "loop_start" # Bookmark this location
  - type: "PlaceBlockOperation"
    block: "hytale:stone"
    offset: "0, 1, 0" # Move up one block
  - type: "JumpIfCounterOperation" # A hypothetical partner operation
    targetIndexName: "loop_start"
    counter: "pillar_height"
    condition: "less_than_10"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SaveIndexOperation()`. The `variableNameArg` field will be uninitialized, causing the operation to store the index under the meaningless name "Undefined". Objects must be created via the `CODEC` deserializer.
-   **Orphaned Bookmarks:** Defining a SaveIndexOperation without a corresponding operation to read the stored index (e.g., a jump or loop) is useless. It adds minor overhead and pollutes the executor's state map for no benefit.

## Data Pipeline
This component operates on control-flow data, not world or entity data. Its pipeline is concerned with capturing and storing its position within an execution sequence.

> Flow:
> Brush Definition File -> **CODEC Deserializer** -> In-Memory `SaveIndexOperation` Instance -> BrushConfigCommandExecutor -> **SaveIndexOperation.preExecutionModifyBrushConfig()** -> Executor State Map (e.g., `{"my_label": 5}`)

