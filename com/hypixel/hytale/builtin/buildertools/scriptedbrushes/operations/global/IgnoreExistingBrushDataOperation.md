---
description: Architectural reference for IgnoreExistingBrushDataOperation
---

# IgnoreExistingBrushDataOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.global
**Type:** Transient Operation

## Definition
```java
// Signature
public class IgnoreExistingBrushDataOperation extends GlobalBrushOperation {
```

## Architecture & Concepts
The IgnoreExistingBrushDataOperation is a stateless, command-like object within the Scripted Brushes system. It serves as a configuration directive rather than a world-modification tool. Its sole function is to alter the behavior of the BrushConfigCommandExecutor during the execution of a brush script.

When included in a brush's operation sequence, this class instructs the executor to discard any brush settings that might already be present on the tool item being used. This ensures that the brush script executes in a clean, predictable state, defined entirely by the script itself, rather than a combination of the script and pre-existing tool data.

As a subclass of GlobalBrushOperation, it affects the overall context of the brush execution, not the specific blocks or entities being targeted. It is a foundational component for creating reliable and self-contained scripted brushes.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale codec system during the deserialization of a brush script asset. The static CODEC field defines how the engine should construct this object from data. It is never instantiated manually in game logic.
-   **Scope:** The object is ephemeral. Its lifetime is confined to a single method call within the brush execution pipeline. It is created, its purpose is served, and it is immediately discarded.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as the `modifyBrushConfig` method completes. No persistent references are held by the engine.

## Internal State & Concurrency
-   **State:** This class is immutable and stateless. It contains no mutable fields and its behavior is constant. The name and description are set once in the constructor and never change.
-   **Thread Safety:** The class is inherently thread-safe due to its immutability. However, it is designed to be executed within a single-threaded context, typically the main server thread responsible for world modification. The method `modifyBrushConfig` directly mutates the state of the passed-in BrushConfigCommandExecutor, which is not designed for concurrent access.

## API Surface
The public API consists of a single method invoked by the brush execution system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Sets a boolean flag on the BrushConfigCommandExecutor, forcing it to ignore pre-existing tool data for the remainder of the current brush execution. |

## Integration Patterns

### Standard Usage
This operation is not intended to be used directly from Java code. It is designed to be declared within a data-driven brush script definition, which the engine then parses and executes.

A conceptual data representation might look like this:

```yaml
# Example: my_reset_brush.yaml
name: "Reset and Place Brush"
operations:
  - type: "IgnoreExistingBrushDataOperation" # This class is instantiated here
  - type: "SetBlockOperation"
    block: "hytale:stone"
  - type: "SphereBrushShape"
    radius: 5
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new IgnoreExistingBrushDataOperation()` in your code. It has no effect outside the context of a brush execution pipeline managed by the engine's codec and command systems.
-   **Incorrect Ordering:** Placing this operation late in a script is ineffective. It must be one of the first operations to ensure all subsequent operations execute with a clean configuration. Any operations defined before it may still use the tool's original data.

## Data Pipeline
This class acts as a control-flow modifier rather than a data processor. It intercepts the configuration pipeline to alter the state of the executor.

> Flow:
> Brush Script Asset -> Engine Codec Deserialization -> **IgnoreExistingBrushDataOperation Instance** -> Brush Pipeline Execution -> `modifyBrushConfig` is called -> BrushConfigCommandExecutor state is mutated -> Subsequent operations execute under the new "ignore" directive.

