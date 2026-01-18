---
description: Architectural reference for BlockSelectionSnapshot
---

# BlockSelectionSnapshot

**Package:** com.hypixel.hytale.builtin.buildertools.snapshot
**Type:** Transient

## Definition
```java
// Signature
public class BlockSelectionSnapshot implements SelectionSnapshot<BlockSelectionSnapshot> {
```

## Architecture & Concepts
The BlockSelectionSnapshot is a data-holding object that functions as a Memento within the Builder Tools plugin system. Its primary role is to capture a point-in-time state of a three-dimensional region of blocks, known as a BlockSelection.

This class is a cornerstone of the undo and redo functionality for world editing. It acts as an immutable record of a selection's contents *before* a modification occurs. By creating a snapshot, the system can later use the `restore` method to revert the world to the state captured within the snapshot.

Architecturally, it decouples the undo history management (likely a stack within BuilderToolsPlugin) from the complex logic of world modification. The snapshot itself contains both the data (the blocks) and the behavior (the `restore` method) required to revert a change.

### Lifecycle & Ownership
- **Creation:** Snapshots are exclusively created via the static factory method `BlockSelectionSnapshot.copyOf(selection)`. This is typically invoked by a higher-level tool or command handler immediately before it modifies the world. The `copyOf` method performs a deep clone of the provided BlockSelection, ensuring the snapshot is completely isolated from any subsequent changes to the original selection object.
- **Scope:** The lifetime of a BlockSelectionSnapshot is tied directly to the undo/redo history stack. It persists as long as the operation it represents is available to be undone or redone.
- **Destruction:** The object is marked for garbage collection when it is removed from the undo/redo history. This occurs when the history buffer is full, a new action overwrites it, or the game session ends.

## Internal State & Concurrency
- **State:** The BlockSelectionSnapshot is an immutable object. Its single field, `selection`, is declared as `final`. The critical design choice is in the `copyOf` factory, which ensures the encapsulated BlockSelection is a *clone*, not a reference to the original. This prevents state corruption if the original BlockSelection object is later modified.
- **Thread Safety:** While the object's state is immutable and safe to read from any thread, its primary method, `restore`, performs world modification. All world modification operations are fundamentally unsafe to perform outside of the main server thread.

    **WARNING:** Invoking the `restore` method from any worker thread will lead to race conditions, chunk corruption, and server instability. All restoration logic must be scheduled to execute on the main game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| restore(ref, player, world, accessor) | BlockSelectionSnapshot | O(N) | Re-applies the captured block data to the world. N is the number of blocks in the selection. Returns a *new* snapshot representing the state of the world *before* this restoration, which is essential for redo functionality. |
| copyOf(selection) | static BlockSelectionSnapshot | O(N) | The sole entry point for creating a snapshot. It performs a deep clone of the selection to guarantee state isolation. N is the number of blocks in the selection. |
| getBlockSelection() | BlockSelection | O(1) | Provides read-only access to the internal BlockSelection data. |

## Integration Patterns

### Standard Usage
The BlockSelectionSnapshot is used as part of a command pattern or undo/redo system. A command that modifies the world first creates a snapshot, performs its action, and then pushes the snapshot onto an undo stack.

```java
// A player is about to delete a selection of blocks
BlockSelection selectionToDelete = player.getSelection();

// 1. Create a snapshot of the area BEFORE the change
BlockSelectionSnapshot undoSnapshot = BlockSelectionSnapshot.copyOf(selectionToDelete);
undoHistory.push(undoSnapshot);

// 2. Perform the world modification
world.clearBlocks(selectionToDelete);
```

To perform an undo, the snapshot is popped from the stack and its `restore` method is called.

```java
// Player triggers the undo command
if (!undoHistory.isEmpty()) {
    BlockSelectionSnapshot snapshotToRestore = undoHistory.pop();

    // 3. Restore the world. The returned snapshot is for the redo stack.
    BlockSelectionSnapshot redoSnapshot = snapshotToRestore.restore(ref, player, world, accessor);
    redoHistory.push(redoSnapshot);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BlockSelectionSnapshot(selection)`. This constructor bypasses the critical `cloneSelection` step. The resulting snapshot would share a reference with the original BlockSelection, leading to a corrupted and useless undo history if the original selection is modified. Always use the static `BlockSelectionSnapshot.copyOf` factory.
- **Asynchronous Restoration:** Do not call `restore` from a separate thread. World modifications must be synchronized with the server's main tick loop to prevent catastrophic state corruption.

## Data Pipeline
The flow of data for a typical undo operation is managed by a higher-level system, with the snapshot acting as the data container.

> Flow (Creating an Undo Record):
> Player Action (e.g., Fill Command) -> BuilderToolsPlugin captures current `BlockSelection` -> **BlockSelectionSnapshot.copyOf()** -> Snapshot pushed to Undo Stack

> Flow (Performing an Undo):
> Player Input (Undo Command) -> Undo Stack pops **BlockSelectionSnapshot** -> `snapshot.restore(world)` -> World State is reverted -> Returned snapshot is pushed to Redo Stack

