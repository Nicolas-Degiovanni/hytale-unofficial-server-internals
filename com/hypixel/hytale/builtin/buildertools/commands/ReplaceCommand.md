---
description: Architectural reference for ReplaceCommand
---

# ReplaceCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Command Handler

## Definition
```java
// Signature
public class ReplaceCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts

The ReplaceCommand class is a server-side command handler within the Builder Tools plugin. It serves as the primary entry point for any player-initiated world modification involving the replacement of blocks within a selection. Architecturally, it embodies the Command pattern, translating high-level player text input into a concrete, executable task.

Its fundamental responsibility is to parse and validate arguments, determine the replacement strategy (e.g., simple, regex, substring), and then delegate the actual, potentially long-running, world modification to the asynchronous task queue managed by the BuilderToolsPlugin. This decoupling is critical for server performance, as it prevents large-scale block operations from blocking the main server thread.

The class contains complex branching logic to handle its various modes of operation, controlled by command flags. It interacts directly with the BlockTypeAssetMap to resolve block names and patterns into internal block IDs, forming the core of its data translation process. The use of a nested private class, ReplaceFromToCommand, is a structural pattern to handle different command usage variants (e.g., `/replace <to>` vs `/replace <from> <to>`) within a single logical unit.

## Lifecycle & Ownership

-   **Creation:** An instance of ReplaceCommand is created and registered by the server's command system when the BuilderToolsPlugin is loaded. This typically occurs once during server startup.
-   **Scope:** The registered command object is a long-lived singleton that persists for the entire server session. However, the parameters passed into its execute method, such as the CommandContext, are transient and scoped to a single command invocation.
-   **Destruction:** The object is dereferenced and eligible for garbage collection when the plugin is unloaded or the server shuts down.

## Internal State & Concurrency

-   **State:** The class instance holds immutable state defined during construction, specifically the definitions for its required arguments and flags (toArg, substringSwapFlag, regexFlag). The core execution logic within the execute and executeReplace methods is stateless, operating exclusively on the arguments provided in the CommandContext for each invocation.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be invoked exclusively from the main server thread. All interactions with world state, such as accessing the EntityStore, must occur on this thread. The primary concurrency mechanism is offloading the intensive block replacement logic to the BuilderToolsPlugin's work queue via `BuilderToolsPlugin.addToQueue`. This lambda submission is a thread-safe operation that schedules the world modification to be performed asynchronously, preventing server stalls.

**WARNING:** Any direct modification of world state within the execute method would introduce severe performance risks and must be avoided.

## API Surface

The public contract is defined by its role as a command handler. Direct programmatic invocation is not a supported use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ReplaceCommand() | constructor | O(1) | Registers the command name, description, permission level, and defines its argument structure. |
| execute(...) | void | O(1) | The entry point called by the command system. Parses context and dispatches a task to an asynchronous queue. The complexity of the dispatched task is variable, but this method returns immediately. |

## Integration Patterns

### Standard Usage

Developers do not interact with this class programmatically. It is invoked by the server's command system in response to player input. A player in Creative mode defines a selection and executes the command via the in-game chat.

*Example Player Interaction:*
1.  Player selects a region with a build tool.
2.  Player types: `/replace stone_bricks cobblestone`
3.  The server parses this, finds the ReplaceCommand handler, and invokes its `execute` method.
4.  The command logic queues a task to replace all `stone_bricks` with `cobblestone` in the selection.
5.  The player receives a confirmation message while the replacement occurs in the background.

### Anti-Patterns (Do NOT do this)

-   **Direct Invocation:** Never call the `execute` method directly from other plugin code. The command system is responsible for establishing the required context, including player permissions, entity references, and world state. Bypassing it will lead to unpredictable behavior and NullPointerExceptions.
-   **Synchronous World Edits:** Do not modify the class to perform block replacements synchronously within the `execute` method. The entire architecture relies on offloading this work to the BuilderToolsPlugin queue to maintain server tick rate.
-   **Stateful Logic:** Do not add instance fields to store data between command executions. Each command must be treated as an atomic, independent transaction.

## Data Pipeline

The flow of data for a replace operation begins with player input and ends with a world update, with this class acting as the central dispatcher.

> Flow:
> Player Chat Input -> Command Parser -> **ReplaceCommand.execute()** -> Argument & Flag Validation -> Task Lambda Creation -> BuilderToolsPlugin Queue -> Asynchronous World Edit -> World Storage Commit

