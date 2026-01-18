---
description: Architectural reference for PrefabEditModifiedCommand
---

# PrefabEditModifiedCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditModifiedCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PrefabEditModifiedCommand is a server-side command processor that integrates with the Builder Tools plugin. It serves as a user-facing entry point for players to query the state of their active prefab editing session. Its sole responsibility is to identify and report which loaded prefabs have been altered but not yet saved.

This class acts as a read-only interface to the PrefabEditSessionManager. It does not mutate any game state or session data. Instead, it retrieves the session associated with the executing player, filters the list of loaded prefabs for any marked as "dirty", and formats a response to be sent back to the player's chat console. This provides a crucial feedback mechanism for builders, allowing them to track their unsaved work without needing to inspect each prefab individually.

## Lifecycle & Ownership
-   **Creation:** An instance of PrefabEditModifiedCommand is created and registered by the server's command system when the parent BuilderToolsPlugin is loaded. It is not instantiated on a per-command-execution basis.
-   **Scope:** The command instance is a long-lived object, persisting for the entire duration that the BuilderToolsPlugin is active on the server.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the BuilderToolsPlugin is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is entirely stateless. It contains no instance fields that hold mutable data. All necessary information is either passed as arguments to the execute method or retrieved from external, stateful services like the PrefabEditSessionManager.
-   **Thread Safety:** The class instance is inherently thread-safe due to its stateless design. However, the execute method is designed to be invoked exclusively by the server's main thread as part of the command processing pipeline. Direct, concurrent invocation of the execute method from other threads is unsupported and would lead to race conditions with the underlying game state and session managers.

## API Surface
The public contract is defined by its role as a command handler. Direct programmatic invocation is not an intended use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the command logic. N is the number of prefabs loaded in the player's session. It fetches the session, filters for modified prefabs, and sends a formatted list to the player. |

## Integration Patterns

### Standard Usage
This command is not intended to be invoked programmatically. It is triggered by a player with appropriate permissions executing the corresponding command in the game's chat console.

1.  A player enters the command (e.g., `/prefab modified`) into chat.
2.  The server's command system parses the input and identifies PrefabEditModifiedCommand as the handler.
3.  The system invokes the execute method, providing the full context of the player and world.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabEditModifiedCommand()`. The command system is responsible for the lifecycle of command objects.
-   **Manual Execution:** Avoid calling the `execute` method directly from other plugin code. This bypasses critical infrastructure such as permission checks, context validation, and standardized error handling provided by the command system. To query prefab state, use the PrefabEditSessionManager API instead.

## Data Pipeline
The flow for this command is initiated by user input and results in a message sent back to the user.

> Flow:
> Player Chat Input -> Server Command Parser -> **PrefabEditModifiedCommand.execute()** -> PrefabEditSessionManager -> PrefabEditSession State -> CommandContext.sendMessage() -> Network Message -> Player Chat UI

