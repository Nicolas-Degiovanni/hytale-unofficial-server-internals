---
description: Architectural reference for PrefabEditTeleportCommand
---

# PrefabEditTeleportCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditTeleportCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PrefabEditTeleportCommand class is a server-side component that functions as a user-facing entry point within the Builder Tools plugin's command system. It translates a player's chat command, such as `/prefab tp`, into a server-side action.

This class acts as a lightweight controller. It does not contain complex business logic. Instead, its primary responsibility is to:
1.  Receive a command execution request from a specific player.
2.  Query the central **PrefabEditSessionManager** to validate the player's current state.
3.  Delegate the core action—opening a user interface—to the player's **PageManager** component.

It serves as a crucial bridge between the server's command parsing system and the player-specific UI framework, ultimately triggering the display of the **PrefabTeleportPage** on the client.

### Lifecycle & Ownership
-   **Creation:** A single instance of PrefabEditTeleportCommand is instantiated by the server's command registration system when the **BuilderToolsPlugin** is loaded. It is then registered under the parent `prefab` command.
-   **Scope:** The command object instance persists for the entire server session, held within the central command registry. The `execute` method, however, is invoked on a per-call basis and has a transient lifetime.
-   **Destruction:** The object is dereferenced and eligible for garbage collection when the plugin is unloaded or the server shuts down, at which point the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is stateless. It maintains no mutable instance fields. All required context, such as the player reference and world state, is provided as arguments to the `execute` method. The static fields are immutable **Message** objects used for localization, which are safe by design.
-   **Thread Safety:** The class is inherently thread-safe. Command execution is dispatched by the server's main game loop, ensuring that calls to `execute` for any given player are serialized. As the object holds no state, concurrent execution from different players on different threads (if the engine were to support it) would not pose a risk.

## API Surface
The public contract is defined by its role as a command, primarily through the inherited `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | The command's entry point. Validates if the player is in a prefab editing session and has loaded prefabs. Opens the teleportation UI or sends an error message to the player. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly in code. It is automatically executed by the server's command system when a player with appropriate permissions types the corresponding command into the chat.

The typical user flow is:
1.  A player enters `/prefab tp` or `/prefab teleport` into the game client's chat window.
2.  The server's command dispatcher identifies the command and routes the request to the registered PrefabEditTeleportCommand instance.
3.  The `execute` method is called with the context of the invoking player.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabEditTeleportCommand()`. The command system handles its lifecycle. Creating an instance manually serves no purpose as it will not be registered to handle chat commands.
-   **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses the server's permission checks, context validation, and error handling that are managed by the command dispatcher.

## Data Pipeline
The flow of data and control for this command begins with user input and terminates with a UI update on the client.

> Flow:
> Player Chat Input (`/prefab tp`) -> Server Command Dispatcher -> **PrefabEditTeleportCommand.execute()** -> PrefabEditSessionManager (State Validation) -> Player.PageManager.openCustomPage() -> Network Packet (UI Open Request) -> Client UI System -> Render PrefabTeleportPage

