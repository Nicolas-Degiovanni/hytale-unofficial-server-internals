---
description: Architectural reference for UseBrushMaskOperation
---

# UseBrushMaskOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.masks
**Type:** Transient / Data Transfer Object

## Definition
```java
// Signature
public class UseBrushMaskOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The UseBrushMaskOperation is a specific, atomic command within the Scripted Brush framework. It does not represent a long-lived service but rather a single, configurable step in a sequence of build tool operations. Architecturally, it embodies the Command Pattern, encapsulating a request to modify a single boolean property—`useBrushMask`—within a target BrushConfig object.

Its primary architectural significance comes from the static **CODEC** field. This class is designed to be instantiated and configured declaratively, typically from an asset file (e.g., JSON or HOCON), rather than imperatively in Java code. The Hytale Codec system uses this static definition to deserialize the configuration into a hydrated UseBrushMaskOperation object at runtime.

This component acts as a simple data carrier whose sole purpose is to be "executed" by a higher-level brush processing system, which iterates through a list of SequenceBrushOperation instances and applies them sequentially to a BrushConfig.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **Codec** system during the deserialization of a Scripted Brush asset. The static CODEC field acts as the factory for this class. Direct instantiation via its public constructor is strongly discouraged.
- **Scope:** The object's lifetime is bound to the lifecycle of the parent Scripted Brush definition that contains it. It is created when the brush is loaded into memory and persists as part of that definition.
- **Destruction:** The object is eligible for garbage collection when its parent Scripted Brush definition is unloaded. It manages no native resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The core state is the public `useBrushMask` boolean field. This state is set once during deserialization by the CODEC system and should be treated as immutable thereafter. While the field is technically public and mutable, altering it post-creation violates the design intent of static, data-driven brush definitions.
- **Thread Safety:** This class is inherently thread-safe for reads. As a simple data holder with no internal logic that modifies its own state, it can be safely referenced from any thread. However, the execution of its `modifyBrushConfig` method is **not** inherently thread-safe, as it mutates an external BrushConfig object. The calling system, which manages the brush execution pipeline, is responsible for ensuring that operations on a given BrushConfig are properly synchronized to prevent race conditions.

## API Surface
The public API is minimal, consisting of the inherited contract from SequenceBrushOperation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Mutates the passed BrushConfig object, setting its `useBrushMask` property to the value stored in this operation. This is a direct, synchronous state change. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly. It is automatically executed by the Scripted Brush engine as part of a larger sequence. A developer's interaction is typically limited to defining it within a brush asset file.

The conceptual execution flow within the engine resembles the following:

```java
// Hypothetical engine code for executing a brush
BrushConfig activeConfig = new BrushConfig();
List<SequenceBrushOperation> operations = loadBrushOperationsFromAsset("my_brush.json");

// The engine iterates and applies each operation
for (SequenceBrushOperation op : operations) {
    // If 'op' is an instance of UseBrushMaskOperation, its logic is applied here
    op.modifyBrushConfig(entityStoreRef, activeConfig, commandExecutor, accessor);
}

// The activeConfig is now modified and ready for use
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new UseBrushMaskOperation()`. This bypasses the data-driven configuration system and creates an unconfigured, default operation. All instances must be created via the Codec system from an asset definition.
- **Post-Creation State Mutation:** Do not modify the public `useBrushMask` field after the brush has been loaded. This can lead to unpredictable behavior and breaks the assumption that brush definitions are static and reusable.

## Data Pipeline
The flow of data for this component begins with a static asset definition and ends with a mutated in-memory object.

> Flow:
> Brush Asset File -> Hytale Codec Deserializer -> In-memory **UseBrushMaskOperation** instance -> Brush Execution Engine -> `modifyBrushConfig` call -> Mutated BrushConfig state

