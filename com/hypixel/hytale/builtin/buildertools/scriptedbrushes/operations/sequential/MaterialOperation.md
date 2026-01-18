---
description: Architectural reference for MaterialOperation
---

# MaterialOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Operation

## Definition
```java
// Signature
public class MaterialOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The MaterialOperation is a data-driven command object within the Scripted Brushes framework. It represents a single, atomic step in a sequence of brush modifications. Its primary architectural role is to translate a simple, human-readable material name from a configuration file into a `BlockPattern` object, which is the low-level data structure required by the core brush engine.

This class is not a service or a manager; it is a self-contained unit of configuration. The static `CODEC` field is the most critical aspect of its design, indicating that instances are intended to be created via deserialization from a data source (e.g., a JSON or HOCON brush definition file) rather than direct instantiation in code. This pattern allows designers and developers to define complex brush behaviors declaratively, without writing Java code.

As a subclass of SequenceBrushOperation, it adheres to a strict contract, exposing a single `modifyBrushConfig` method that the brush execution engine invokes to apply its specific change.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale `Codec` system during the parsing of a scripted brush definition. The `CODEC` static field acts as the factory, mapping configuration keys like "BlockType" to the internal state of the new object.
-   **Scope:** The lifecycle is extremely short. An instance exists only for the duration of its execution within a larger sequence of operations. It is created, its `modifyBrushConfig` method is called once, and then it is discarded.
-   **Destruction:** The object holds no persistent state or external references. It becomes eligible for garbage collection immediately after the `modifyBrushConfig` method returns.

## Internal State & Concurrency
-   **State:** The class holds a single mutable field, `blockTypeArg`. This state is set once during deserialization by the `CODEC` and is treated as immutable thereafter. The operation itself is stateless in the sense that it does not cache data or change its behavior across multiple invocations.
-   **Thread Safety:** This class is **not thread-safe** and is not designed to be. Brush operations are executed in a strict, synchronous sequence on the primary server thread responsible for world modifications. Any attempt to invoke this class from multiple threads will result in race conditions on the shared `BrushConfig` object and lead to world corruption.

## API Surface
The public API is minimal, consisting only of the contract method required by its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, ...) | void | O(1) | Applies the material configuration to the target BrushConfig. This is the sole entry point for the operation's logic. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in procedural code. It is designed to be defined declaratively in a brush script file and invoked by the engine. The following example illustrates the conceptual execution flow managed by the system.

```java
// The system, not the user, performs these steps.
// 1. A brush script is loaded and parsed by the Codec system.
//    This implicitly creates a MaterialOperation instance.
MaterialOperation operation = codec.decode(brushScriptNode);

// 2. The BrushSequencer invokes the operation on a BrushConfig.
BrushConfig targetConfig = new BrushConfig();
operation.modifyBrushConfig(entityRef, targetConfig, executor, accessor);

// 3. The targetConfig is now updated with a new BlockPattern.
//    The brush will now paint with "Rock_Stone".
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new MaterialOperation()`. This bypasses the `Codec` system, leaving the `blockTypeArg` field with its default value ("Rock_Stone") and ignoring any intended configuration.
-   **State Re-use:** Do not hold a reference to a MaterialOperation instance and attempt to re-use it. These objects are designed to be transient and should be treated as single-use commands.
-   **Concurrent Execution:** Never call `modifyBrushConfig` from any thread other than the main server thread. The underlying `BrushConfig` and world state are not protected against concurrent modification.

## Data Pipeline
The primary flow for this component is configuration data being transformed into an in-memory engine state.

> Flow:
> Brush Script File (e.g., JSON) -> Hytale Codec Deserializer -> **MaterialOperation Instance** -> BrushSequencer Engine -> `modifyBrushConfig` Call -> Updated `BrushConfig` Object -> World Edit Engine

