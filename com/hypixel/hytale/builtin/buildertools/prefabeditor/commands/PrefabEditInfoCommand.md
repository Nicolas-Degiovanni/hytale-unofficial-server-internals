---
description: Architectural reference for PrefabEditInfoCommand
---

# PrefabEditInfoCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditInfoCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PrefabEditInfoCommand class is a concrete implementation of the Command Pattern, designed to operate within the server's command processing system. It serves as a read-only interface for players to query the state of their current prefab editing session.

This command acts as a leaf node in the larger Builder Tools feature set. Its primary architectural role is to decouple the user input layer (the chat and command parser) from the internal state management of the prefab editor (PrefabEditSessionManager). It translates a player's request into a query against the session manager and formats the result into a user-friendly message. It does not modify any game state; it is a pure query operation.

## Lifecycle & Ownership
-   **Creation:** A single instance of PrefabEditInfoCommand is instantiated by the command registration system when the BuilderToolsPlugin is loaded. It is not created on a per-request basis.
-   **Scope:** The object instance persists for the entire lifecycle of the server or until the parent plugin is unloaded. It is effectively a singleton managed by the command framework. The parameters passed to the execute method, such as CommandContext, are request-scoped and transient.
-   **Destruction:** The instance is eligible for garbage collection when the server shuts down or the BuilderToolsPlugin is disabled, at which point the command registry releases its reference.

## Internal State & Concurrency
-   **State:** This class is stateless. It contains no mutable instance fields. The static Message fields are immutable and loaded once at class initialization. All state required for an operation is retrieved from external systems like the PrefabEditSessionManager via a player-specific key (UUID).

-   **Thread Safety:** The command is thread-safe under the assumption that it is executed on a per-player basis by a single thread at a time, which is the standard model for Hytale's command system. It relies on the external PrefabEditSessionManager to provide thread-safe access to session data. Direct, concurrent invocations of the execute method for the same player would be an engine-level contract violation and lead to undefined behavior.

## API Surface
The public contract is defined by its superclass, AbstractPlayerCommand. The primary entry point is the execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Queries the player's prefab editing session and sends a formatted message with details about the selected prefab. Throws assertion errors if core components are missing. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly. It is registered with the server's command system and triggered by player input. A programmatic invocation, if necessary, would be performed through the command system's dispatch mechanism, not by calling the execute method.

```java
// Conceptual example of the system dispatching the command
// Developers do not call this class directly.
CommandSystem commandSystem = server.getCommandSystem();
Player executingPlayer = ...;

// The system finds the registered command and invokes it
commandSystem.dispatch(executingPlayer, "editprefab info");
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new PrefabEditInfoCommand()`. The object is managed by the command system. Creating a new instance serves no purpose as it is not registered to receive commands.
-   **Stateful Implementation:** Do not add mutable instance fields to this class. A single instance is shared for all command executions, and adding per-player state would create severe race conditions and data corruption.
-   **Bypassing Session Manager:** Do not attempt to access or modify PrefabEditSession objects directly. All interactions must go through the PrefabEditSessionManager to ensure proper lifecycle and state management.

## Data Pipeline
This command facilitates a request-response data flow, where the input is a player command and the output is a chat message.

> Flow:
> Player Input (`/editprefab info`) -> Server Command Parser -> **PrefabEditInfoCommand** -> PrefabEditSessionManager (Query) -> Formatted Message -> Network Packet -> Player Chat UI

