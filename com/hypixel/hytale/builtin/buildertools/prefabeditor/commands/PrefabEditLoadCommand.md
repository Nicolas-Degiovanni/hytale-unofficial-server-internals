---
description: Architectural reference for PrefabEditLoadCommand
---

# PrefabEditLoadCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditLoadCommand extends AbstractAsyncPlayerCommand {
```

## Architecture & Concepts
The PrefabEditLoadCommand class is an implementation of the Command Pattern, serving as the primary entry point for initiating a prefab editing session from a player-invoked chat command. It acts as a translator, converting raw string-based player input into a structured, validated request that can be processed by the core builder tools systems.

This command is a critical bridge between the server's command processing system and the **PrefabEditSessionManager**. It is solely responsible for defining the command's syntax, parsing its arguments, and delegating the complex logic of prefab loading and world manipulation to the appropriate manager.

The command exposes two distinct operational modes:
1.  **Interactive Mode:** When invoked with no arguments, it opens the PrefabEditorLoadSettingsPage UI for the player. This provides a guided, user-friendly way to configure the loading parameters without requiring knowledge of the command's syntax.
2.  **Parameterized Mode:** When invoked with arguments, it bypasses the UI and immediately triggers the asynchronous loading process. This mode is intended for power users and automation.

By extending AbstractAsyncPlayerCommand, it signals to the command system that its primary execution logic is potentially long-running and must be performed off the main server thread to prevent server stalls.

### Lifecycle & Ownership
-   **Creation:** A single instance of PrefabEditLoadCommand is instantiated by the server's command registry when the BuilderToolsPlugin is loaded. It is registered as the handler for the `load` subcommand within the `/prefab edit` command namespace.
-   **Scope:** The object instance persists for the entire server session as a singleton handler. However, the *execution* of the command is transient; a new execution context is created for each invocation and is discarded upon completion.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down or the associated plugin is unloaded.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its member fields are argument definitions (e.g., RequiredArg, DefaultArg) which are configured once in the constructor and are not modified thereafter. All state relevant to a specific command execution is passed in via the `CommandContext` parameter and encapsulated within a new PrefabEditorCreationSettings object.

-   **Thread Safety:** The class is inherently thread-safe due to its stateless design. The primary logic in `executeAsync` is designed to be run on a worker thread, managed by the server's command execution engine. It returns a `CompletableFuture`, adhering to the server's standard asynchronous programming model. All mutations of world state are delegated to the PrefabEditSessionManager, which is responsible for its own thread safety and synchronization with the main game loop.

## API Surface
The public contract of this class is its registration as a command, not direct method invocation. The following method is the callback invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context, store, ref, playerRef, world) | CompletableFuture<Void> | O(N) | Asynchronously initiates a prefab editing session. Complexity is proportional to the size (N) of the prefabs being loaded. Throws no exceptions directly but handles errors by sending messages to the player. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly from code. It is invoked by a player through the in-game chat console.

```
# Example: Load a single prefab from the asset directory with default settings
/prefab edit load asset my_awesome_castle

# Example: Load all prefabs in a subfolder recursively with custom spacing
/prefab edit load asset structures/village --recursive --spacing 25 --stackingAxis Z
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabEditLoadCommand()`. The command system manages the lifecycle of command objects. Direct instantiation will result in an object that is not registered to handle any chat commands.
-   **Manual Invocation:** Do not call the `executeAsync` method directly. Doing so bypasses the server's critical command processing pipeline, including argument parsing, permission checks, and context setup.

## Data Pipeline
The flow of data for a parameterized command invocation follows a clear, delegated path from player input to world modification.

> Flow:
> Player Chat Input (`/prefab edit load...`) -> Server Command Parser -> **PrefabEditLoadCommand** -> PrefabEditorCreationSettings (DTO) -> PrefabEditSessionManager.loadPrefabAndCreateEditSession -> Asynchronous World Generation Task -> Player Teleport & Session Activation

