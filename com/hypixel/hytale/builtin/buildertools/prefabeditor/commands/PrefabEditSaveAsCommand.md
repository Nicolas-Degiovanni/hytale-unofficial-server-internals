---
description: Architectural reference for PrefabEditSaveAsCommand
---

# PrefabEditSaveAsCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditSaveAsCommand extends AbstractAsyncPlayerCommand {
```

## Architecture & Concepts
The PrefabEditSaveAsCommand class is a command handler responsible for persisting a player's in-game prefab editing session to a file. It serves as a user-facing entry point into the prefab serialization pipeline, triggered when a player executes the `/editprefab saveas` command.

Architecturally, this class acts as a controller that orchestrates interactions between several core systems:
*   **Command System:** It registers itself with the server's command dispatcher to listen for specific player input.
*   **Prefab Session Management:** It queries the PrefabEditSessionManager to retrieve the state of the player's current editing session.
*   **World State:** It interacts with the player's current selection state to define the boundaries of the data to be saved.
*   **Serialization & Persistence:** It configures and invokes the PrefabSaver utility, which handles the complex logic of serializing block and entity data into the Hytale prefab JSON format and writing it to disk.

A critical design aspect is its inheritance from AbstractAsyncPlayerCommand. This ensures that the potentially long-running file I/O operation is executed on a separate worker thread, preventing the main server game loop from stalling and maintaining server performance.

## Lifecycle & Ownership
- **Creation:** A single prototype instance of this class is created and registered with the server's command system when the BuilderToolsPlugin is loaded. It is not instantiated per-player or per-command-execution.
- **Scope:** The object's lifecycle is tied to the plugin's lifecycle. However, its operational scope is transient; it holds no state between command invocations. All necessary context is provided via the arguments to the executeAsync method.
- **Destruction:** The prototype instance is destroyed and garbage collected when the BuilderToolsPlugin is unloaded by the server.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. The command argument definitions (e.g., fileNameArg, noEntitiesArg) are immutable configurations set during construction. All runtime data is sourced from the CommandContext, the World, or external state managers like PrefabEditSessionManager.
- **Thread Safety:** The class itself is not designed for concurrent access. The server's command execution framework guarantees that the executeAsync method is invoked safely on a worker thread. Any interaction with shared game state, such as the PrefabEditSessionManager, relies on the thread safety of those external components. This class contains no explicit locking mechanisms.

## API Surface
The primary public contract of this class is the command it exposes to players, not a programmatic API for other systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(...) | CompletableFuture<Void> | O(N) | Asynchronously executes the save logic. Complexity is O(N) where N is the number of blocks and entities in the selection, dominated by serialization and file I/O. Throws no checked exceptions but handles internal errors by sending failure messages to the player. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked programmatically. A developer or player interacts with it via the in-game chat console.

*Example command execution:*
```
// Saves the current selection to server/prefabs/structures/my_castle.prefab.json
/editprefab saveas structures/my_castle

// Saves the selection, overwriting any existing file, and excluding entities
/editprefab saveas structures/my_castle --overwrite --noEntities
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabEditSaveAsCommand()` in your code. The command system handles its lifecycle. Manual instantiation provides no mechanism for execution.
- **Blocking on Future:** If you were to somehow get a reference to the returned CompletableFuture, never call `.get()` or `.join()` on it from the main server thread. This would block the game loop and negate the benefits of the asynchronous design.
- **Path Traversal:** The command includes internal security checks to prevent saving files outside of designated prefab directories. Do not attempt to bypass these checks by manipulating the file name argument with sequences like `../`.

## Data Pipeline
This command initiates a data flow that transforms in-memory world state into a persistent file on disk.

> Flow:
> Player Chat Input (`/editprefab saveas ...`) -> Server Command Dispatcher -> **PrefabEditSaveAsCommand** -> PrefabEditSessionManager (State Retrieval) -> PrefabSaver (Serialization) -> Filesystem I/O (Write `.prefab.json`) -> CompletableFuture Completion -> Message sent to Player Client

