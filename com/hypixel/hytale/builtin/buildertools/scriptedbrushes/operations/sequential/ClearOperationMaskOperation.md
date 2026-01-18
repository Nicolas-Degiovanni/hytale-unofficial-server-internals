---
description: Architectural reference for ClearOperationMaskOperation
---

# ClearOperationMaskOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ClearOperationMaskOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The ClearOperationMaskOperation is a stateless, single-purpose command object within the Scripted Brush framework. It represents one discrete step in a sequence of modifications applied to a brush's configuration during its use.

Its primary architectural role is to manage the state of a brush's effective area. The system distinguishes between two types of masks:
1.  **Tool Mask:** The base shape and area of effect defined by the brush tool itself.
2.  **Operation Mask:** A temporary, additive, or subtractive mask applied by a preceding operation within a script. This allows for complex, multi-stage brush behaviors where the area of effect changes dynamically.

This operation's sole function is to discard the current *Operation Mask*, reverting the brush's effective area to the original *Tool Mask*. It is a reset mechanism, ensuring that subsequent operations in the sequence are not affected by masks set by prior ones. It is fundamental for creating clean state transitions within complex brush scripts.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly via the constructor. They are instantiated by the engine's serialization system, specifically through the public static BuilderCodec field named CODEC. This occurs when the server loads a brush configuration asset (e.g., a JSON file) that declaratively lists this operation as part of a sequence.
-   **Scope:** The object's lifetime is exceptionally brief and tied to a single execution of a brush script. It is created, its modifyBrushConfig method is invoked once by the BrushConfigCommandExecutor, and it is then immediately eligible for garbage collection. It holds no state beyond its initial configuration.
-   **Destruction:** Managed entirely by the Java garbage collector. There are no native resources or explicit cleanup methods required.

## Internal State & Concurrency
-   **State:** This object is **immutable and stateless**. It contains no mutable fields and does not cache any data. Its behavior is entirely determined by the arguments passed to its methods.
-   **Thread Safety:** The class itself is inherently thread-safe. However, it operates upon a mutable BrushConfig object. The caller, typically the BrushConfigCommandExecutor, is responsible for ensuring that the entire sequence of brush operations is executed in a thread-safe context, preventing concurrent modifications to the BrushConfig instance.

## API Surface
The public contract is minimal, consisting of the deserialization entry point and the core execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | Mutates the passed BrushConfig instance by clearing its operation-specific mask. |
| CODEC | BuilderCodec | N/A | The static deserializer used by the engine to instantiate this object from data. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is designed to be used declaratively within a brush configuration file. The engine's brush system deserializes and executes it as part of a larger command sequence.

A conceptual representation in a configuration file might look like this:

```yaml
# Example pseudo-code for a brush asset
name: "ComplexFillBrush"
sequence:
  - operation: "SetOperationMask" # First, limit the area to a sphere
    shape: "sphere"
    radius: 5
  - operation: "Fill" # Perform a fill operation within that sphere
    block: "hytale:stone"
  - operation: "ClearOperationMask" # Reset the mask for the next step
  - operation: "Overlay" # Apply an overlay using the brush's original shape
    block: "hytale:grass"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new ClearOperationMaskOperation()`. The object is useless outside the context of the brush execution system which provides the necessary BrushConfig state.
-   **Manual Invocation:** Do not call the modifyBrushConfig method directly. The state of the BrushConfig and the context provided by the BrushConfigCommandExecutor are critical for correct behavior. Bypassing the executor can lead to unpredictable brush state.

## Data Pipeline
This component acts as a state modifier within a control-flow pipeline, not a data-transformation pipeline. Its execution directly mutates the state of the active BrushConfig.

> Flow:
> Brush Asset File -> Engine Deserializer (using CODEC) -> `ClearOperationMaskOperation` Instance -> BrushConfigCommandExecutor -> **`modifyBrushConfig()`** -> BrushConfig State is Mutated

