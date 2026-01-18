---
description: Architectural reference for BrushConfigListCommand
---

# BrushConfigListCommand

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.commands
**Type:** Transient

## Definition
```java
// Signature
public class BrushConfigListCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The BrushConfigListCommand is a server-side command component that provides a read-only view into a player's active brush configuration. It functions as a leaf node within the server's command processing tree, specifically serving the Builder Tools feature set.

Architecturally, this class embodies the Command Pattern. It encapsulates a single, specific request: to list the settings of the command-issuing player's brush. It acts as a bridge between the player's chat input and the internal state of the `BrushConfigCommandExecutor`, which manages the actual brush settings.

This command does not hold or modify any state. Its sole responsibility is to:
1.  Identify the player executing the command.
2.  Retrieve the player-specific tool settings via the static `ToolOperation` accessor.
3.  Query the `BrushConfigCommandExecutor` for its list of global and sequential operations.
4.  Format this data into a series of localized, human-readable messages.
5.  Transmit these messages back to the player.

This design effectively decouples the command invocation system from the complex internal state of the builder tools.

## Lifecycle & Ownership
-   **Creation:** An instance of BrushConfigListCommand is created and registered by its parent command system during server initialization or plugin loading. It is registered under the name "list" within a parent command structure (e.g., `/brush config`).
-   **Scope:** The object instance is effectively a singleton managed by the command framework, persisting for the entire server session. However, the execution context provided to its `execute` method is ephemeral, lasting only for the duration of a single command invocation.
-   **Destruction:** The instance is garbage collected when the server shuts down or when the command is programmatically unregistered from the command system.

## Internal State & Concurrency
-   **State:** BrushConfigListCommand is **stateless**. It contains no instance fields that persist between invocations. All data required for its operation is either passed as arguments to the `execute` method or retrieved from other stateful systems.

-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, its `execute` method is designed to be called by the server's main game thread or a designated command-processing thread. It interacts with the `ToolOperation` system, which manages shared, mutable state (player tool settings).

    **WARNING:** The thread safety of a command execution is dependent on the guarantees provided by the underlying data stores it accesses. It is assumed that systems like `ToolOperation` provide the necessary synchronization to prevent race conditions when accessing a specific player's settings.

## API Surface
The primary contract is with the command framework via the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N * M) | Executes the command logic. Retrieves and lists all brush operations (N) and their settings (M) for the invoking player. Sends formatted messages to the player. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically invoked by the server's command handler when a player executes the corresponding chat command.

The conceptual flow is managed entirely by the server framework:
1.  Player types `/brush config list` in chat.
2.  The server parses this input and dispatches it to the registered BrushConfigListCommand instance.
3.  The framework invokes the `execute` method with the appropriate context.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BrushConfigListCommand()`. The command system manages the lifecycle of command objects. Manual instantiation serves no purpose as the instance will not be registered to handle chat commands.
-   **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses critical framework functionality, including permission checks, context validation, and exception handling. Manually constructing the required `CommandContext`, `Store`, and `Ref` arguments is complex and error-prone.

## Data Pipeline
This command facilitates a one-way data flow from the server's internal state to the client's user interface.

> Flow:
> Player Chat Input (`/brush config list`) -> Server Command Parser -> **BrushConfigListCommand.execute()** -> `ToolOperation` Static Accessor -> `BrushConfigCommandExecutor` (State Query) -> `MessageFormat` (Data-to-Text Transformation) -> `PlayerRef.sendMessage()` -> Network Layer -> Client Chat UI

