---
description: Architectural reference for SetDensity
---

# SetDensity

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Operation

## Definition
```java
// Signature
public class SetDensity extends SequenceBrushOperation {
```

## Architecture & Concepts
The SetDensity class is a concrete implementation of the **Command Pattern**, designed to operate within the Scripted Brush system. It does not perform direct world modification. Instead, its sole responsibility is to configure the execution context for subsequent operations within a brush sequence.

It functions as a state modifier for the active BrushConfig object, which acts as a shared context passed between all operations in a given brush script. By invoking SetDensity, a script can introduce a probabilistic element to block placement, effectively controlling the "density" of the brush's application. For example, setting a density of 50 means that subsequent block-setting operations in the sequence will only have a 50% chance of executing for any given block position.

This class is entirely data-driven, instantiated and configured by the engine's codec system when parsing a brush definition file.

## Lifecycle & Ownership
- **Creation:** An instance is created by the static BuilderCodec when deserializing a brush script. It is never instantiated directly in game logic. The framework reads a configuration value (e.g., "Density: 30") and uses the codec to construct a corresponding SetDensity object.
- **Scope:** The object's lifetime is exceptionally short and confined to a single method invocation. It is created, its modifyBrushConfig method is called by the brush executor, and it is immediately eligible for garbage collection.
- **Destruction:** Cleanup is handled by the Java garbage collector. As the object holds no persistent state and is not registered with any service, it is reclaimed as soon as the brush executor moves to the next operation in the sequence.

## Internal State & Concurrency
- **State:** The class holds a single mutable state field: density. This value is set by the codec during deserialization and is considered effectively immutable thereafter. The default value is 100, representing a 100% chance of execution.
- **Thread Safety:** This class is **not thread-safe** and must only be used from the main server thread that handles world updates. The Scripted Brush system is designed for single-threaded execution to prevent race conditions and ensure deterministic world generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, ...) | void | O(1) | Configures the provided BrushConfig instance by setting its density property. The input value is clamped to a safe range of 1 to 100. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is used declaratively within a brush script. The system's brush executor is responsible for instantiating and calling it. The following example illustrates how the *system* would use the object after it has been deserialized.

```java
// Hypothetical brush execution loop
BrushConfig currentConfig = new BrushConfig();
List<SequenceBrushOperation> operations = loadOperationsFromScript();

for (SequenceBrushOperation op : operations) {
    // If 'op' is an instance of SetDensity, this method is called.
    op.modifyBrushConfig(entityStoreRef, currentConfig, executor, accessor);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new SetDensity(). The object is useless without being properly configured by the codec system from a data source. Its state will be incomplete, leading to default (and likely unintended) behavior.
- **State Reuse:** Do not cache or reuse an instance of SetDensity. Each operation defined in a script corresponds to a new, distinct instance.

## Data Pipeline
The flow of data through this component is linear and unidirectional, modifying a shared state object.

> Flow:
> Brush Script Data -> BuilderCodec Deserializer -> **SetDensity Instance** -> BrushConfig State -> Subsequent Brush Operations

