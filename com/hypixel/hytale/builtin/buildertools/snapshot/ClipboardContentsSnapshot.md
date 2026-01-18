---
description: Architectural reference for ClipboardContentsSnapshot
---

# ClipboardContentsSnapshot

**Package:** com.hypixel.hytale.builtin.buildertools.snapshot
**Type:** Transient

## Definition
```java
// Signature
public class ClipboardContentsSnapshot implements ClipboardSnapshot<ClipboardContentsSnapshot> {
```

## Architecture & Concepts
The ClipboardContentsSnapshot is an immutable data-holding object that implements the **Memento design pattern**. Its sole purpose is to capture, store, and restore the state of a player's clipboard, which is represented by a BlockSelection object.

This class is a fundamental component of the Builder Tools undo/redo system. It does not represent the *live* clipboard itself, but rather a frozen, point-in-time record of its contents. By creating a snapshot before a destructive operation, the system gains the ability to revert the clipboard to its previous state.

Its design emphasizes safety and predictability. The internal BlockSelection is deeply cloned upon creation, ensuring that a snapshot is completely isolated from any subsequent modifications to the original selection object it was created from. This prevents complex state corruption bugs common in state management systems.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by the Builder Tools command system, typically via the static factory method ClipboardContentsSnapshot.copyOf. This occurs immediately before a command modifies the player's active clipboard selection.
- **Scope:** Short-lived. An instance typically exists for as long as it is held within an undo/redo stack. It is not a persistent entity and does not survive server restarts or player sessions.
- **Destruction:** The object is eligible for garbage collection as soon as it is no longer referenced, for example, when an undo stack is cleared or a player's builder session ends.

## Internal State & Concurrency
- **State:** **Immutable**. The internal BlockSelection field is final and is assigned a deep copy of the source selection during construction. Once created, a ClipboardContentsSnapshot instance can never be altered.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It can be safely passed across threads without synchronization. However, its usage is almost exclusively confined to the main server thread as part of the game logic loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| restoreClipboard(...) | ClipboardContentsSnapshot | O(1) | Swaps the current BuilderState selection with this snapshot's selection. Critically, it returns a new snapshot of the state *before* the swap, enabling redo functionality. |
| copyOf(selection) | static ClipboardContentsSnapshot | O(N) | **Primary Factory Method.** Creates a new, isolated snapshot by performing a deep clone of the provided BlockSelection. N is the number of blocks in the selection. |

## Integration Patterns

### Standard Usage
The primary pattern involves capturing state before an operation and using the snapshot to restore it later, typically as part of an undo command.

```java
// In a builder tool command execution handler...

// 1. Capture the state of the clipboard BEFORE modifying it.
ClipboardContentsSnapshot undoSnapshot = ClipboardContentsSnapshot.copyOf(builderState.getSelection());

// 2. Add this snapshot to the undo stack for the player.
playerUndoStack.push(undoSnapshot);

// 3. Perform the operation that modifies the live selection.
BlockSelection newSelection = createNewSelectionFromWorld();
builderState.setSelection(newSelection);
```

To perform an undo, the snapshot is popped from the stack and used to restore the state.

```java
// In an undo command handler...
ClipboardContentsSnapshot snapshotToRestore = playerUndoStack.pop();

// The restoreClipboard method returns the state that is being overwritten.
// This is captured and pushed to the redo stack.
ClipboardContentsSnapshot redoSnapshot = snapshotToRestore.restoreClipboard(..., builderState, ...);
playerRedoStack.push(redoSnapshot);
```

### Anti-Patterns (Do NOT do this)
- **Shallow Copying:** Never use `new ClipboardContentsSnapshot(selection)`. This creates a snapshot that shares the same BlockSelection reference as the live state. If the live selection is later modified, the snapshot becomes corrupted. Always use the static `ClipboardContentsSnapshot.copyOf` factory method.
- **Ignoring Return Value:** The `restoreClipboard` method returns the state it just overwrote. Discarding this return value breaks the undo/redo chain, making a "redo" operation impossible.

## Data Pipeline
This class acts as a temporary data container within the Builder Tools command processing flow. It does not actively process data but rather holds it for later retrieval.

> **Copy Operation Flow:**
> Player Input (Copy Command) -> Command Handler -> **ClipboardContentsSnapshot.copyOf(builderState.getSelection())** -> Push to Undo Stack -> BuilderState updated with new selection

> **Undo Operation Flow:**
> Player Input (Undo Command) -> Command Handler -> Pop from Undo Stack -> **snapshot.restoreClipboard(builderState)** -> Returned snapshot pushed to Redo Stack

