---
description: Architectural reference for AssetUndoRedoInfo
---

# AssetUndoRedoInfo

**Package:** com.hypixel.hytale.builtin.asseteditor.data
**Type:** Transient

## Definition
```java
// Signature
public class AssetUndoRedoInfo {
```

## Architecture & Concepts
The AssetUndoRedoInfo class is a stateful data container that underpins the undo and redo functionality within the Hytale asset editor. It is a direct implementation of the state-holding component of the **Command Pattern**.

This class does not contain any logic itself. Instead, it serves as a passive repository for sequences of `JsonUpdateCommand` objects. A higher-level controller or service, such as an `AssetEditorManager`, is responsible for populating, querying, and manipulating the stacks within this object in response to user actions. Its primary role is to decouple the history of operations from the execution of those operations, allowing for a clean and state-driven undo/redo system.

Each instance of AssetUndoRedoInfo corresponds to the complete command history for a single, active asset editing session.

### Lifecycle & Ownership
- **Creation:** An instance is created by a managing service when a new asset editing session is initiated. It is not intended for direct instantiation by general application code.
- **Scope:** The object's lifetime is strictly scoped to its parent editing session. It persists as long as the asset editor window for a specific asset is open.
- **Destruction:** The object is marked for garbage collection when the user closes the associated asset editor, and the parent session object is destroyed.

## Internal State & Concurrency
- **State:** This object is fundamentally **mutable**. Its entire purpose is to have its internal `undoStack` and `redoStack` collections modified by an external controller as the user performs and reverts actions.
- **Thread Safety:** This class is **not thread-safe**. The underlying `ArrayDeque` collections are not synchronized. All access and mutation of the `undoStack` and `redoStack` fields must be externally synchronized or, more commonly, confined to a single thread (e.g., the main UI or game thread). Failure to do so will result in race conditions and data corruption.

## API Surface
This class exposes its state via public fields, acting as a simple data structure. There is no method-based API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| undoStack | Deque<JsonUpdateCommand> | O(1) | The stack of commands representing actions that can be undone. The most recent action is at the top. |
| redoStack | Deque<JsonUpdateCommand> | O(1) | The stack of commands representing actions that have been undone and can be redone. |

## Integration Patterns

### Standard Usage
An external controller manages the lifecycle of commands, pushing them onto the `undoStack` after execution and moving them between stacks as the user invokes undo or redo actions.

```java
// In a hypothetical AssetEditorController
// Assume 'history' is the AssetUndoRedoInfo for the current session

// On a new user action (e.g., moving a vertex)
JsonUpdateCommand command = createNewCommandForAction();
execute(command);

// Add the command to the history and invalidate the redo stack
history.undoStack.push(command);
history.redoStack.clear();


// On user pressing "Undo"
if (!history.undoStack.isEmpty()) {
    JsonUpdateCommand commandToUndo = history.undoStack.pop();
    unexecute(commandToUndo); // Apply the inverse operation
    history.redoStack.push(commandToUndo);
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Instance:** Do not use a single AssetUndoRedoInfo instance for multiple asset editor windows. Each editing session must have its own unique history object to prevent state contamination.
- **Concurrent Modification:** Do not access or modify the `undoStack` or `redoStack` from a background thread while the main editor thread is also interacting with it. This will lead to unpredictable behavior and likely a `ConcurrentModificationException`.

## Data Pipeline
This class is not part of a linear data pipeline but rather a state repository in a user-driven control loop.

> Control Flow:
> User Action -> Editor Controller -> Creates `JsonUpdateCommand` -> Pushes to **AssetUndoRedoInfo**.undoStack

