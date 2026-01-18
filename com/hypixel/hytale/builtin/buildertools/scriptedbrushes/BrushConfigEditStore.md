---
description: Architectural reference for BrushConfigEditStore
---

# BrushConfigEditStore

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes
**Type:** Transient

## Definition
```java
// Signature
public class BrushConfigEditStore {
```

## Architecture & Concepts

The BrushConfigEditStore is a transactional, in-memory cache that serves as an intermediary between a scripted brush operation and the live world state. It provides an isolated "sandbox" environment for a single, complex brush stroke, ensuring that the operation is atomic, consistent, and reversible.

Its primary architectural role is to manage the state changes of a brush that may have multiple stages or require reading its own modifications before they are committed to the world. It achieves this by maintaining three distinct layers of block data:

1.  **Original State (before):** A snapshot of the world as it was *before* any modifications were made by this operation. This layer is populated on-demand the first time a block is targeted for a write, effectively creating the data required for an "undo" command.
2.  **Committed-Stage State (previous):** A cumulative record of all changes made and flushed in prior stages of the *same* brush stroke. This allows a multi-stage brush (e.g., place stone, then place ore on top of that stone) to see its own work from previous steps.
3.  **Current-Stage State (current):** A temporary buffer holding all block changes made in the current, active stage. These changes are not visible to subsequent read operations until they are explicitly committed via a flush.

By abstracting away direct world access, this class enables complex procedural generation and editing logic while preventing partial or corrupt data from being written to the world if an operation is interrupted.

### Lifecycle & Ownership

-   **Creation:** A new BrushConfigEditStore is instantiated by a higher-level tool controller or script executor at the beginning of a single, continuous brush application (e.g., when a player clicks and begins to drag a tool). It is not a persistent or shared service.
-   **Scope:** The object's lifetime is strictly bound to a single user action. It may persist across multiple server ticks during a prolonged operation (like painting a large area), but it is fundamentally a short-lived, single-use object.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as the brush operation concludes. The calling system is responsible for extracting the final `before` and `after` BlockSelection objects to apply the changes to the world and register an undo state. No references to the store should be held after this point.

## Internal State & Concurrency

-   **State:** This class is highly mutable and stateful by design. Its internal BlockSelection fields accumulate data throughout its lifecycle. The state represents a delta of changes to be applied to the world, not the state of the world itself.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, operated upon, and discarded entirely within the server's main game thread. Any concurrent access from other threads will result in race conditions, data corruption, and unpredictable world state. All interactions must be synchronized with the main server tick loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlock(x, y, z) | int | O(1) | Gets the block ID at a world position, respecting changes from previously flushed stages. |
| setBlock(x, y, z, blockId) | boolean | O(1) | Stages a block change in the current buffer. Returns false if the change is rejected by a mask or density rule. |
| setMaterial(x, y, z, material) | boolean | O(1) | A high-level wrapper for setting a block or fluid based on a Material object. |
| flushCurrentEditsToPrevious() | void | O(N) | Commits all staged changes from the current buffer to the previous buffer, making them visible to subsequent `getBlock` calls. N is the number of edits in the current stage. |
| getBefore() | BlockSelection | O(1) | Returns the complete selection of all original block states that were modified during the operation. Used for generating "undo" commands. |
| getAfter() | BlockSelection | O(1) | Returns the cumulative result of all flushed stages, representing the final desired world state. Used for generating "redo" commands. |

## Integration Patterns

### Standard Usage

The BrushConfigEditStore is the central component for executing a scripted brush. The controlling system orchestrates its lifecycle.

```java
// 1. Create the store for a new brush operation
BrushConfigEditStore editStore = new BrushConfigEditStore(packedPositions, config, world);

// 2. Execute brush logic, which reads from and writes to the store
// This might happen over multiple stages
for (BrushStage stage : brush.getStages()) {
    stage.execute(editStore);
    
    // 3. Flush edits between stages so the next stage can see the changes
    editStore.flushCurrentEditsToPrevious();
}

// 4. After the operation is complete, retrieve the results
BlockSelection originalState = editStore.getBefore();
BlockSelection finalState = editStore.getAfter();

// 5. Apply the changes to the world and register for undo/redo
world.applySelection(finalState);
undoManager.register(originalState, finalState);
```

### Anti-Patterns (Do NOT do this)

-   **Reference Hoarding:** Do not maintain a reference to a BrushConfigEditStore instance after its originating brush stroke is complete. This will cause a significant memory leak, as the internal `before` and `previous` selections can grow very large.
-   **Bypassing the Store:** Within a single brush operation, do not mix calls to the store with direct world modification. Doing so breaks the transactional guarantee and will lead to an inconsistent state, especially for undo operations.
-   **Omitting Flushes:** Forgetting to call `flushCurrentEditsToPrevious` between stages of a multi-stage brush is a common error. This will cause the second stage to operate on the original world state, ignoring all work done by the first stage.

## Data Pipeline

The flow of data through the BrushConfigEditStore is layered to ensure consistency and state separation. A write operation triggers a read from the world to populate the `before` cache, while a read operation checks internal caches before consulting the world.

> **Write Operation Flow:**
> `setBlock()` call -> BrushConfig Rule Check (Density, Mask) -> **BrushConfigEditStore** (Captures original block from World into `before` cache) -> Stages new block in `current` cache

> **Read Operation Flow:**
> `getBlock()` call -> **BrushConfigEditStore** (Checks `previous` cache) -> **BrushConfigEditStore** (Checks `accessor` for live World data if not in cache) -> Return Block ID

> **Stage Commit Flow:**
> `flushCurrentEditsToPrevious()` call -> **BrushConfigEditStore** (Merges `current` cache into `previous` cache) -> **BrushConfigEditStore** (Clears `current` cache)

