---
description: Architectural reference for PrefabEditSaveUICommand
---

# PrefabEditSaveUICommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditSaveUICommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PrefabEditSaveUICommand is a server-side command handler that acts as a user-facing entry point for the Prefab Editor's save functionality. It is a component of the Builder Tools plugin and integrates directly with the server's core command processing system.

Its sole responsibility is to bridge player input (a typed chat command) with the server's UI framework. Upon execution, it retrieves the player's current editing context from the PrefabEditSessionManager and, if valid, constructs and displays the PrefabEditorSaveSettingsPage. This class is a classic example of the Command pattern, encapsulating a request as an object, thereby decoupling the invoker (the command system) from the receiver (the UI and session management systems).

## Lifecycle & Ownership
- **Creation:** A single instance is created and registered by the BuilderToolsPlugin during its initialization phase. The server's central CommandSystem owns this instance.
- **Scope:** The command object is a long-lived singleton that persists for the entire server session, or until the owning plugin is unloaded. The execution context provided to its methods, however, is ephemeral and scoped to a single command invocation.
- **Destruction:** The object is garbage collected when the BuilderToolsPlugin is disabled or the server shuts down, at which point the CommandSystem releases its reference.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no mutable instance fields. All required context, such as the player reference and world state, is passed as arguments to the execute method. The static Message fields are immutable constants loaded at class initialization.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. The execute method is designed to be called by the server's main thread, which has exclusive access to the game state components (Store, Ref, World) passed into it. Operations on the PrefabEditSession are delegated to the PrefabEditSessionManager, which is responsible for its own thread safety.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | The primary entry point. Validates the player's edit session and opens the save settings UI. Sends failure messages to the player via the CommandContext if validation fails. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically registered and executed by the server's command system in response to player input. The standard interaction is a player typing the command in the game client.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabEditSaveUICommand()`. The command system manages the lifecycle of command objects. Manual instantiation serves no purpose.
- **Manual Invocation:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, context setup, and error handling provided by the server's CommandSystem.

## Data Pipeline
The command initiates a UI flow based on server-side state. It does not process a continuous stream of data but rather acts as a trigger.

> Flow:
> Player Chat Command -> Command System -> **PrefabEditSaveUICommand.execute()** -> PrefabEditSessionManager lookup -> Player.PageManager.openCustomPage() -> Server-to-Client UI Packet

