---
description: Architectural reference for SetOperation
---

# SetOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient

## Definition
```java
// Signature
public class SetOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The SetOperation class is a fundamental, stateless command object within the Scripted Brush framework. It represents the most atomic world modification action: setting a single block or fluid at a specific coordinate.

Architecturally, it embodies the **Command Pattern**. Each instance is a self-contained, executable unit of work. It is designed to be composed into a sequence of operations managed by a parent Scripted Brush. The brush's execution engine iterates through a list of these operations, applying them to a target volume of blocks.

This class decouples the *intent* to set a block from the complex machinery of world modification. It does not know about player input, brush shapes, or transaction management; it only knows how to take a coordinate and a material from the current BrushConfig and write that change to a pending edit store. This separation of concerns makes it highly reusable for defining a vast array of custom building tools.

## Lifecycle & Ownership
- **Creation:** SetOperation instances are not created directly by developers using the *new* keyword. They are instantiated by the engine's serialization system via the static **CODEC** field. This typically occurs when a game asset defining a scripted brush is loaded from disk.

- **Scope:** The lifetime of a SetOperation instance is tied to the lifecycle of the parent BrushConfig object that contains it. It persists as part of the brush's definition for as long as that brush is loaded in memory.

- **Destruction:** The object is eligible for garbage collection when its parent BrushConfig is unloaded or redefined. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** The SetOperation class is **stateless and immutable**. It contains no instance fields to store data between calls. All necessary information, such as the target coordinates, the brush configuration, and the edit store, is provided as arguments to its methods.

- **Thread Safety:** This class is inherently **thread-safe**. Because it holds no state, a single instance can be safely executed by multiple threads simultaneously without risk of data corruption.

    **Warning:** While the SetOperation itself is thread-safe, the objects it interacts with, such as BrushConfig and BrushConfigEditStore, may not be. The calling context, typically the brush execution engine, is responsible for ensuring that access to these shared objects is properly synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBlocks(ref, config, executor, edit, x, y, z, accessor) | boolean | O(1) | Executes the primary logic. Retrieves the next material from the BrushConfig and writes it to the BrushConfigEditStore at the given coordinates. Always returns true. |
| modifyBrushConfig(ref, config, executor, accessor) | void | O(1) | A no-op method. This operation does not alter the state of the parent BrushConfig. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly in user code. It is designed to be defined within a data file (e.g., JSON) and executed by the Scripted Brush system. The following conceptual example illustrates how the *engine* would use an instance of this class within its execution loop.

```java
// Conceptual engine-level usage
// An instance of SetOperation is retrieved from the brush's operation list.
SequenceBrushOperation operation = myBrush.getOperation(i);

// The engine iterates through target coordinates and invokes the operation.
for (BlockCoordinate coord : targetVolume) {
    operation.modifyBlocks(
        entityStoreRef,
        brushConfig,
        commandExecutor,
        editStore,
        coord.getX(),
        coord.getY(),
        coord.getZ(),
        accessor
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SetOperation()`. Doing so bypasses the engine's configuration and serialization system, resulting in an object that is not properly integrated into the brush framework. All operations must be defined via data and loaded through the engine's CODEC system.

- **Stateful Extension:** Do not extend this class to add stateful instance variables. The stateless nature of brush operations is a core architectural principle that ensures predictability and thread safety.

## Data Pipeline
The SetOperation acts as a simple processor in the data flow from brush configuration to world state modification.

> Flow:
> Brush Configuration Data -> BuilderCodec Deserializer -> **SetOperation Instance** -> Brush Execution Engine -> `modifyBlocks` Call -> BrushConfigEditStore (Pending Changes) -> World Storage Commit

---

