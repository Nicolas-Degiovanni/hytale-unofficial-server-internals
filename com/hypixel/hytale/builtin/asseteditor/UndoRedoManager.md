---
description: Architectural reference for UndoRedoManager
---

# UndoRedoManager

**Package:** com.hypixel.hytale.builtin.asseteditor
**Type:** Singleton

## Definition
```java
// Signature
public class UndoRedoManager {
```

## Architecture & Concepts
The UndoRedoManager is the central state repository for command history within the Hytale Asset Editor. It is not a game-wide system, but is specifically scoped to tracking user-initiated changes to assets during an editing session.

Its primary architectural function is to decouple the state of an asset's modification history from the UI and the command execution logic. Instead of each editor panel managing its own undo/redo stacks, they query this central manager to retrieve the correct history object.

The core design is a map-based strategy where each unique AssetPath serves as a key to an AssetUndoRedoInfo object. This object contains the actual undo and redo stacks. This design enables multiple assets to be edited concurrently in separate tabs or windows, each maintaining an independent and isolated modification history. The use of fastutil's Object2ObjectOpenHashMap indicates a design preference for performance and low memory overhead over thread safety.

## Lifecycle & Ownership
- **Creation:** A single instance of UndoRedoManager is instantiated by the primary Asset Editor service or context provider when the editor subsystem is initialized. It is created once and only once.
- **Scope:** The manager's lifecycle is bound to the Asset Editor session. It persists as long as the editor is active, even if all individual asset editor windows are closed.
- **Destruction:** The instance is eligible for garbage collection only when the entire Asset Editor subsystem is shut down, typically upon returning to the main menu or exiting the client. Individual history stacks managed by this class must be manually cleared via clearUndoRedoStack when an asset is no longer being edited to prevent memory leaks.

## Internal State & Concurrency
- **State:** The UndoRedoManager is highly stateful and mutable. Its internal map, assetUndoRedoInfo, grows and shrinks as users open, edit, and close assets. It acts as a live cache of all command histories for the current session.
- **Thread Safety:** **This class is not thread-safe.** All methods mutate the internal map and must be invoked exclusively from the main application thread (the UI or Editor thread). The underlying data structure, Object2ObjectOpenHashMap, is not synchronized. Any concurrent access from background threads will result in undefined behavior, including potential data corruption or ConcurrentModificationExceptions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOrCreateUndoRedoStack(AssetPath) | AssetUndoRedoInfo | O(1) avg | Retrieves the history for an asset. Creates a new, empty history if one does not exist. This is the primary entry point. |
| getUndoRedoStack(AssetPath) | AssetUndoRedoInfo | O(1) avg | Retrieves the history for an asset. Returns null if no history exists for the given path. |
| putUndoRedoStack(AssetPath, AssetUndoRedoInfo) | void | O(1) avg | Directly inserts or overwrites the history for an asset. Primarily used for advanced state restoration. |
| clearUndoRedoStack(AssetPath) | AssetUndoRedoInfo | O(1) avg | Removes the history for an asset, freeing its memory. Returns the removed history object. **CRITICAL** for preventing memory leaks. |

## Integration Patterns

### Standard Usage
The manager is always retrieved from a central service context. An editor panel then gets the specific history stack for the asset it manages and pushes commands to it.

```java
// In an Asset Editor Panel's initialization
UndoRedoManager undoManager = editorContext.getService(UndoRedoManager.class);
AssetPath currentAsset = this.getAssetPath();
this.undoRedoInfo = undoManager.getOrCreateUndoRedoStack(currentAsset);

// When a user performs an action
IEditorCommand command = new VoxelPaintCommand(...);
command.execute();
this.undoRedoInfo.pushUndo(command);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new UndoRedoManager()`. This creates an orphaned instance that is not tracked by the editor's service architecture, leading to a completely disconnected and non-functional undo/redo system.
- **State Sharing:** Do not share a single AssetUndoRedoInfo instance across multiple AssetPaths. Each asset must have its own unique history stack.
- **Forgetting Cleanup:** Failure to call `clearUndoRedoStack` when an asset editor tab is closed will cause a memory leak. The command history for that asset will be retained in the manager's map for the remainder of the session.

## Data Pipeline
The UndoRedoManager does not process data in a traditional pipeline. Instead, it serves as a central repository in an interaction-driven workflow.

> Flow:
> User Action in Editor UI -> Editor creates `IEditorCommand` -> Command is executed, modifying asset state -> Command is passed to `AssetUndoRedoInfo` stack (retrieved from **UndoRedoManager**) -> User requests Undo -> UI retrieves **UndoRedoManager** -> Gets `AssetUndoRedoInfo` -> Pops command and executes its inverse.

