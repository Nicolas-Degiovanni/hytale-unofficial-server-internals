---
description: Architectural reference for BrushOperation
---

# BrushOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.system
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class BrushOperation {
```

## Architecture & Concepts
The BrushOperation class is the abstract foundation for the **Command Pattern** that powers Hytale's Scripted Brush system. It represents a single, atomic, and configurable action that can be performed by a builder's tool, such as setting blocks, applying a mask, or executing control flow logic.

This architecture is designed for extensibility and data-driven configuration. Concrete implementations, like SetOperation or JumpIfBlockTypeOperation, are registered in a central, static registry. A "scripted brush" is not defined in code but as a sequence of these operations in a data file, typically a JSON asset. The engine deserializes this asset at runtime, using the registry to instantiate a corresponding list of BrushOperation objects.

This design decouples the brush execution logic from the operation definitions, allowing designers and developers to create complex tool behaviors by simply composing different operations in a data file without recompiling the engine. The system acts as a small, domain-specific virtual machine where each BrushOperation is an instruction, and the BrushConfig acts as the machine's memory or registers.

## Lifecycle & Ownership
- **Creation:** Instances are created by the engine's serialization system when a BrushConfig is loaded from an asset. The static BRUSH_OPERATION_REGISTRY maps a string identifier from the data file (e.g., "set", "mask") to a constructor reference (e.g., SetOperation::new), which is then invoked. Direct instantiation by developers is an anti-pattern.
- **Scope:** The lifecycle of a BrushOperation instance is strictly bound to its parent BrushConfig. It persists in memory as part of a list of operations for as long as the brush configuration is active.
- **Destruction:** Instances are marked for garbage collection when the owning BrushConfig is unloaded or replaced. There is no explicit destruction or cleanup method that must be called.

## Internal State & Concurrency
- **State:** The base class itself holds immutable metadata (name, description) and a mutable map of its settings, registeredOperationSettings. This map is populated during the creation phase and is intended to be read-only during execution. Concrete subclasses may introduce their own internal state, which can be reset via the resetInternalState hook.
- **Thread Safety:** **CRITICAL:** BrushOperation instances are **not thread-safe**. They are designed for synchronous, sequential execution on the main server thread. The primary method, modifyBrushConfig, directly manipulates shared, mutable game state such as the EntityStore and BrushConfig. Any attempt to execute operations concurrently will result in race conditions, data corruption, and unpredictable world state. The use of a ConcurrentHashMap for the registry only ensures thread safety during the initial registration phase at engine startup, not during operation execution.

## API Surface
The public API is primarily intended for consumption by the Scripted Brush execution engine, not for direct calls by game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | Variable | **Core Method.** Executes the operation's logic, modifying the BrushConfig and/or world state. Complexity depends entirely on the implementation, from O(1) for state changes to O(N) for world edits, where N is the volume of the brush. |
| createBrushSetting(...) | BrushOperationSetting | O(1) | Factory method used by subclasses in their constructor to define configurable parameters. |
| resetInternalState() | void | O(1) | Hook for stateful operations to reset themselves, typically called before a new brush stroke begins. |
| preExecutionModifyBrushConfig(...) | void | O(1) | A lifecycle hook called by the executor immediately before modifyBrushConfig. |

## Integration Patterns

### Standard Usage
The standard method for using BrushOperation is declarative, not imperative. A developer or designer defines a sequence of operations and their settings within a data asset file. The engine handles the instantiation and execution.

The following conceptual example illustrates how a simple "set stone" brush would be defined.

```yaml
# Conceptual brush_script.yaml asset
# This file is loaded by the BrushConfig system.

name: "Stone Placer Brush"
operations:
  - id: "material" # Corresponds to BRUSH_OPERATION_REGISTRY key
    settings:
      material: "hytale:stone"
  - id: "shape"
    settings:
      shape: "sphere"
      radius: 5
  - id: "set"
    # This operation has no settings; it uses the material and shape
    # set by previous operations in the BrushConfig.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SetOperation()`. The system relies on the data-driven factory pattern via the BRUSH_OPERATION_REGISTRY for correct initialization and lifecycle management.
- **Concurrent Execution:** Never invoke modifyBrushConfig from multiple threads or for the same BrushConfig concurrently. This will corrupt world data. All brush execution must be serialized.
- **Stateful Mismanagement:** Do not design operations that carry state between separate brush uses (e.g., between two separate player clicks) without properly using the resetInternalState hook. This can lead to unpredictable behavior.

## Data Pipeline
The BrushOperation acts as a processor in the data pipeline that transforms a player action into a world modification.

> Flow:
> Brush Script Asset -> Engine Deserializer -> List of BrushOperation instances -> BrushConfigCommandExecutor -> **BrushOperation.modifyBrushConfig** -> Modified BrushConfig State & World State (EntityStore)

