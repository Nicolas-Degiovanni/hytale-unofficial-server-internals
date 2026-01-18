---
description: Architectural reference for SequenceBrushOperation
---

# SequenceBrushOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.system
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class SequenceBrushOperation extends BrushOperation {
```

## Architecture & Concepts
The SequenceBrushOperation class is an abstract base class that forms a fundamental component of the Scripted Brush system. It represents a single, atomic step within a larger sequence of world modification actions. This class embodies the **Strategy Pattern**, where each concrete implementation defines a specific behavior (e.g., place block, replace block, run command) that can be composed into a complex tool.

Its primary role is to provide a standardized contract for operations that can be executed by the brush engine. The engine iterates through a list of these operations, invoking them sequentially to apply a brush's effect to the world.

A critical architectural feature is the **doesOperateOnBlocks** flag. This boolean, set during construction, serves as a performance hint to the brush execution engine. The engine can use this flag to optimize its execution path, for example, by skipping the expensive process of acquiring world region locks for operations that do not modify block data.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of SequenceBrushOperation are instantiated by the Scripted Brush parser or a factory when a brush configuration is loaded. They are not created directly by gameplay logic but are part of the tool's definition.
- **Scope:** An instance's lifetime is tied to the execution context of a specific brush. It is typically a short-lived object, existing only for the duration of a single player action (e.g., a single click of a tool).
- **Destruction:** The object is eligible for garbage collection as soon as the brush execution sequence completes and all references are dropped. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The base class holds one piece of immutable state: the **doesOperateOnBlocks** flag. Concrete implementations may introduce their own state, but they are strongly encouraged to be stateless to ensure predictable and repeatable behavior.
- **Thread Safety:** **This class is not thread-safe.** Instances are designed to be created, configured, and executed exclusively on the server's main thread. The **modifyBlocks** method is passed a direct reference to the EntityStore, and any concurrent access would lead to world corruption, race conditions, and server instability.

**WARNING:** All implementations of this class must assume they are operating in a single-threaded, synchronous environment. Do not introduce asynchronous operations or access shared mutable state without external locking.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBlocks(...) | boolean | O(N) | The core execution method. Applies the operation's logic at the given coordinates. Subclass implementation determines complexity. |
| beginIterationIndex(int) | void | O(1) | A lifecycle hook called by the engine before an iteration begins. Useful for operations that need to reset state between iterations. |
| getNumModifyBlockIterations() | int | O(1) | Informs the engine how many times **modifyBlocks** should be called for a single point. Defaults to 1. |
| doesOperateOnBlocks() | boolean | O(1) | Returns true if the operation modifies block data, allowing the engine to perform optimizations. |

## Integration Patterns

### Standard Usage
This is an abstract class and cannot be instantiated directly. A developer must extend it to define a new brush behavior. The brush engine will then invoke the implemented methods.

```java
// 1. Define a concrete operation
public class PlaceStoneOperation extends SequenceBrushOperation {
    public PlaceStoneOperation() {
        // Name, description, and doesOperateOnBlocks = true
        super("Place Stone", "Replaces the target block with stone.", true);
    }

    @Override
    public boolean modifyBlocks(
        Ref<EntityStore> entityStoreRef,
        BrushConfig brushConfig,
        BrushConfigCommandExecutor executor,
        BrushConfigEditStore edit,
        int x, int y, int z,
        ComponentAccessor<EntityStore> accessor
    ) {
        // World modification logic goes here
        // This is a simplified example
        EntityStore store = entityStoreRef.get();
        store.setBlock(x, y, z, STONE_BLOCK_ID);
        return true; // Indicates a change was made
    }
}

// 2. The brush engine uses the operation (conceptual)
BrushConfig myBrush = loadBrushFromScript(); // Contains a PlaceStoneOperation
myBrush.executeAt(player.getPosition());
```

### Anti-Patterns (Do NOT do this)
- **Retaining World State:** Do not store references to the EntityStore or other world state objects as fields within the class. This can lead to memory leaks and use of stale data. All necessary state is passed into the **modifyBlocks** method.
- **Blocking Operations:** The **modifyBlocks** method is executed on the main server thread. Performing file I/O, network requests, or other long-running tasks within this method will cause severe server lag or crashes.
- **Ignoring Iteration Hooks:** For complex, multi-iteration operations, failing to implement **beginIterationIndex** and **getNumModifyBlockIterations** can lead to incorrect or unpredictable behavior.

## Data Pipeline
The SequenceBrushOperation acts as a processor within the server-side world modification pipeline.

> Flow:
> Player Input -> Server Command -> Scripted Brush Executor -> **SequenceBrushOperation.modifyBlocks** -> EntityStore (World State Change) -> World Save & Network Sync -> Client Render Update

