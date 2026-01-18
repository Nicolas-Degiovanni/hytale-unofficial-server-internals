---
description: Architectural reference for RandomizeDimensionsOperation
---

# RandomizeDimensionsOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.dimensions
**Type:** Transient Command

## Definition
```java
// Signature
public class RandomizeDimensionsOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The RandomizeDimensionsOperation is a concrete implementation of the **Strategy Pattern**, designed to function as a single, atomic step within a larger scripted brush execution pipeline. Its sole responsibility is to mutate the dimensional properties—specifically width and height—of a BrushConfig object.

This class is a fundamental component of the server-side Builder Tools framework. It allows content creators and level designers to introduce procedural variation into brush tools without writing imperative code. The operation is defined declaratively in data files (e.g., HOCON or JSON) and instantiated at runtime by the engine's serialization system via its static CODEC field. This design decouples the complex behavior of a scripted brush from its simple, composable parts.

It acts as a state mutator within a well-defined sequence, receiving the current brush state, modifying it, and passing it along to the next operation in the chain.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly using the constructor. Instead, they are instantiated by the Hytale Codec system when a scripted brush definition is loaded from a resource pack or configuration file. The static CODEC field dictates how the object is constructed from the source data.
- **Scope:** The object's lifetime is bound to the lifetime of its parent scripted brush configuration. It is effectively a configuration value that encapsulates behavior. It persists as long as the brush tool definition is loaded in memory.
- **Destruction:** The object is marked for garbage collection when the parent brush definition is unloaded. This typically occurs when a server shuts down, a world is closed, or game assets are reloaded.

## Internal State & Concurrency
- **State:** The internal state consists of the RelativeIntegerRange fields for width and height. This state is configured once upon deserialization and is intended to be treated as immutable for the remainder of the object's lifecycle. The class itself does not cache any runtime data or results between executions.
- **Thread Safety:** This class is **not thread-safe**. The core method, modifyBrushConfig, directly mutates the passed-in BrushConfig object. The Builder Tools execution engine guarantees that all operations within a single brush sequence are executed serially on the main server thread.
    - **WARNING:** Attempting to invoke this operation from multiple threads on a shared BrushConfig instance will result in severe data corruption and unpredictable brush behavior. All brush modifications must be synchronized by the calling system.

## API Surface
The primary public contract is the inherited modifyBrushConfig method. The public fields exist for reflective access by the Codec system and should not be modified directly at runtime.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, ...) | void | O(1) | Mutates the shape width and height of the provided BrushConfig. This is the sole entry point for the operation's logic. |

## Integration Patterns

### Standard Usage
This operation is not intended for direct, imperative use in code. It is designed to be used declaratively within a scripted brush definition file. The engine's systems handle its lifecycle and execution.

A conceptual brush definition might look like this:

```yaml
# Example Scripted Brush Definition
name: "Random Sizer Brush"
sequence: [
    {
        # This block is deserialized into a RandomizeDimensionsOperation instance
        type: "RandomizeDimensions"
        WidthRange: "~-2..~2"  # Relative range: current width +/- 2
        HeightRange: "5..10"   # Absolute range: between 5 and 10
    },
    {
        # ... other operations like PlaceBlockOperation would follow
        type: "PlaceBlock"
        block: "hytale:stone"
    }
]
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new RandomizeDimensionsOperation()`. The object will lack the proper configuration, which is injected by the Codec system during asset loading.
- **Runtime State Mutation:** Do not modify the `widthRangeArg` or `heightRangeArg` fields after the brush has been loaded. This can lead to non-deterministic behavior and breaks the declarative model. Configuration is static.
- **External Invocation:** Do not call `modifyBrushConfig` from outside the context of the scripted brush execution engine. The engine provides the necessary state and transactional context for the operation to succeed safely.

## Data Pipeline
The flow of data and control for this operation is linear and unidirectional, managed entirely by the scripted brush system.

> Flow:
> Brush Definition File -> Hytale Codec System -> **RandomizeDimensionsOperation Instance** -> Brush Execution Engine -> `modifyBrushConfig` Invocation -> Mutated BrushConfig State -> Subsequent Brush Operations

