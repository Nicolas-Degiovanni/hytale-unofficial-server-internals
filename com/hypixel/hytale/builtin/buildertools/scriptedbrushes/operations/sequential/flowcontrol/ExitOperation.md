---
description: Architectural reference for ExitOperation
---

# ExitOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol
**Type:** Transient / Command

## Definition
```java
// Signature
public class ExitOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The ExitOperation is a concrete implementation of the Command Pattern, representing a terminal flow-control instruction within the Scripted Brushes system. Its sole responsibility is to signal an immediate and unconditional halt to the execution of the current operation sequence.

Architecturally, it functions as a `break` or `return` statement for a brush script. When the BrushConfigCommandExecutor encounters an ExitOperation in its execution queue, it ceases processing any subsequent operations for the current brush application. This allows for the creation of brush scripts that can terminate prematurely without completing their full sequence, a critical feature for creating conditional or state-dependent building tools.

This class is entirely stateless; its behavior is defined by its type alone. It is discovered and instantiated by the engine's codec system when parsing brush configuration assets.

### Lifecycle & Ownership
- **Creation:** Instantiated by the `BuilderCodec` deserialization pipeline when a brush configuration asset is loaded into memory. It is never created manually by game logic. It becomes an immutable element within a `BrushConfig` object's operation list.
- **Scope:** The lifecycle of an ExitOperation instance is strictly bound to its parent `BrushConfig`. It persists as long as the brush definition is loaded.
- **Destruction:** The object is eligible for garbage collection when its parent `BrushConfig` is unloaded or the server shuts down. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** This object is stateless and immutable. It holds no internal data beyond the metadata provided in its constructor, which is constant for the lifetime of the application.
- **Thread Safety:** Inherently thread-safe. As a stateless command object, it can be referenced and its methods invoked from any thread without risk of data corruption. However, the `BrushConfigCommandExecutor` that invokes it is responsible for ensuring that the overall brush execution sequence is handled in a thread-safe manner, typically by confining it to the main world-update thread.

## API Surface
The public API is minimal, consisting only of the inherited method required to participate in the operation sequence.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | Instructs the BrushConfigCommandExecutor to terminate execution of the current operation stack. This is a terminal command. |

## Integration Patterns

### Standard Usage
This operation is not intended to be instantiated or called directly from Java code. It is designed to be declared within a data-driven brush configuration file (e.g., a JSON asset) and executed by the engine. The following example demonstrates how the *system* invokes the operation, not how a developer should use it.

```java
// PSEUDO-CODE: How the BrushConfigCommandExecutor processes operations
for (SequenceBrushOperation operation : brushConfig.getOperations()) {
    // The executor invokes the operation
    operation.modifyBrushConfig(ref, brushConfig, this, accessor);

    // The executor checks its internal state after the call
    if (this.isExecutionHalted()) {
        break; // Exit the loop
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ExitOperation()`. The Scripted Brush system relies on codec-driven instantiation to build its operation chains. Manual creation will result in an object that is not part of any managed execution sequence.
- **Manual Invocation:** Calling `modifyBrushConfig` directly on an instance is meaningless. The method's effect is entirely dependent on it being called by a `BrushConfigCommandExecutor`, which manages the execution state that this operation is designed to modify.

## Data Pipeline
ExitOperation does not process data. Instead, it alters the *control flow* of its parent system.

> Flow:
> BrushConfigCommandExecutor (Execution Loop) → Invokes `ExitOperation.modifyBrushConfig` → ExitOperation calls `BrushConfigCommandExecutor.exitExecution` → BrushConfigCommandExecutor sets internal "halt" flag → Execution Loop terminates.

