---
description: Architectural reference for UndoCommand
---

# UndoCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class UndoCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The UndoCommand class is a command-pattern implementation that serves as the primary user-facing entry point for the build tools history system. It acts as an adapter, translating a player's in-game chat command, such as `/undo`, into a deferred task processed by the core BuilderToolsPlugin.

This class is not responsible for the undo logic itself. Its sole responsibilities are:
1.  **Command Registration:** Declaring the command name (`undo`), alias (`u`), required permissions, and usage context (Creative mode).
2.  **Argument Parsing:** Utilizing a nested `UndoWithCountCommand` class to handle an optional integer argument specifying the number of actions to undo. This is a common pattern for creating overloaded command signatures.
3.  **Task Dispatching:** Upon successful execution, it retrieves the necessary player and entity context and enqueues a task on the BuilderToolsPlugin's work queue. This decouples the command invocation from the actual world modification, ensuring that history operations are processed in a controlled, sequential manner.

This design cleanly separates the concerns of command parsing and permission checking from the complex logic of history management and world state manipulation.

### Lifecycle & Ownership
-   **Creation:** An instance of UndoCommand is created by the server's Command System during plugin initialization. The system scans the plugin for classes extending `AbstractCommand` and registers them in a central command registry.
-   **Scope:** The registered command handler instance persists for the entire lifecycle of the BuilderToolsPlugin. The `execute` method is invoked each time a player runs the command, but the object itself is long-lived.
-   **Destruction:** The instance is garbage collected when the BuilderToolsPlugin is unloaded or the server shuts down, at which point the Command System clears its registry.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no mutable instance fields. All necessary state (the player, the world, command arguments) is provided via method parameters during the `execute` call. The nested `UndoWithCountCommand` holds an immutable reference to its argument definition.
-   **Thread Safety:** The class is **conditionally thread-safe**. As a stateless object, it can be safely referenced from any thread. However, its primary function is to dispatch a task to the `BuilderToolsPlugin` queue. The overall operation's safety is therefore dependent on the thread-safe implementation of that queue. The server's command system typically ensures that commands are executed on the main game thread, mitigating most concurrency risks at the point of invocation.

## API Surface
The public API is consumed exclusively by the server's Command System framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Framework-invoked method. Validates context and dispatches an undo task to the BuilderToolsPlugin queue. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. The standard interaction is a player executing the command in-game. A programmatic equivalent would involve using a command dispatcher service to simulate player input.

```java
// Hypothetical: Forcing a player to execute the undo command
// WARNING: This is for illustrative purposes only.
CommandSystem commandSystem = server.getCommandSystem();
Player targetPlayer = server.getPlayer("Notch");

// The system handles parsing, permissions, and execution
commandSystem.dispatch(targetPlayer, "undo 5");
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new UndoCommand()` in your code. Command objects are managed by the server's command registry and are not designed for manual creation or invocation.
-   **Manual Execution:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, argument parsing, and context injection, which will lead to NullPointerExceptions and unstable server state.
-   **Assuming Synchronous Execution:** The undo operation is asynchronous. This command only queues the task. Do not write code that assumes the world has been modified immediately after the command is dispatched.

## Data Pipeline
The flow of data for an undo operation begins with player input and terminates with a world state change, with this class acting as an early component in the chain.

> Flow:
> Player Chat Input (`/undo 5`) -> Server Command Parser -> **UndoCommand** -> BuilderToolsPlugin Task Queue -> History System Worker -> World State Modification

