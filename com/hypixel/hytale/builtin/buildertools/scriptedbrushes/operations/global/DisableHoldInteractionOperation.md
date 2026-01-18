---
description: Architectural reference for DisableHoldInteractionOperation
---

# DisableHoldInteractionOperation

**Package:** `com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.global`
**Type:** Transient

## Definition
```java
// Signature
public class DisableHoldInteractionOperation extends GlobalBrushOperation {
```

## Architecture & Concepts
The DisableHoldInteractionOperation is a specialized, stateless component within the Scripted Brush framework. It functions as a conditional **control-flow gate**, not as an operation that modifies world data. Its sole purpose is to inspect the current brush's configuration and terminate the execution pipeline if a specific condition is met.

In the context of the broader Builder Tools system, brushes can be configured to execute as a chain of operations. This class is designed to be one of the first operations in that chain. It checks if the brush is currently being activated via a "hold" interaction (e.g., the user is holding down the mouse button for continuous application). If this is the case, it instructs the BrushConfigCommandExecutor to halt further processing of the operation chain for the current tick, effectively enforcing a "single-fire" behavior for the brush.

The static CODEC field indicates that this class is intended to be instantiated declaratively from configuration files or scripts, rather than being manually constructed in Java code. The system uses this codec to deserialize a brush's definition into a sequence of executable operations.

## Lifecycle & Ownership
- **Creation:** Instantiated by the engine's codec and serialization system when a scripted brush definition is loaded. It is not intended for manual creation via its constructor.
- **Scope:** The object's lifetime is extremely brief and is scoped to a single brush execution event. An instance is created, its `modifyBrushConfig` method is invoked, and it is then immediately eligible for garbage collection.
- **Destruction:** Managed by the Java Garbage Collector. No manual cleanup is required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is determined exclusively by the arguments passed to its methods.
- **Thread Safety:** Inherently **thread-safe**. As a stateless object, it can be safely used across multiple threads without synchronization. However, the objects it operates on, such as BrushConfig, may have their own distinct concurrency constraints.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Checks if the brush is in a hold-down interaction state. If true, it calls `exitExecution` on the provided executor to stop the operation pipeline. |

## Integration Patterns

### Standard Usage
This operation is not used imperatively in Java. Instead, it is declared as part of a brush's definition within a configuration or script file. The engine's deserializer then constructs the object and integrates it into the brush's execution pipeline.

A hypothetical brush definition might look like this:

```yaml
# Example Brush Definition
name: "Single Fire Placement Brush"
operations:
  - type: "DisableHoldInteractionOperation" # This operation is placed first
  - type: "SetBlockOperation"
    block: "hytale:stone"
  - type: "PlaySoundOperation"
    sound: "hytale:pop"
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new DisableHoldInteractionOperation()`. The engine's configuration loader is responsible for creating instances via the provided CODEC. Direct creation bypasses the intended declarative design.
- **Incorrect Ordering:** Placing this operation late in the execution chain is a critical error. If placed after world-modifying operations, it will fail to prevent them from running, defeating its purpose. It must be one of the first operations in the sequence.

## Data Pipeline
This component primarily influences the control flow of the brush execution pipeline rather than transforming data.

> Flow:
> Brush Activation Event -> Operation Pipeline Start -> **DisableHoldInteractionOperation** -> (Condition Met: Halt) OR (Condition Not Met: Continue) -> Subsequent World-Modifying Operations

