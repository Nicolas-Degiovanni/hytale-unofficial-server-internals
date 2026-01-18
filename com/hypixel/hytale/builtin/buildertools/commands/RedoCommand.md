---
description: Architectural reference for RedoCommand
---

# RedoCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class RedoCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The RedoCommand class is a server-side command handler responsible for processing the player-initiated `/redo` action. It serves as a critical bridge between the Command System, which interprets user input, and the BuilderToolsPlugin, which manages world modification history.

This class follows the Command design pattern, encapsulating a request as an object. It extends AbstractPlayerCommand, inheriting the necessary framework to be automatically discovered and registered by the server. This base class also guarantees that the command's execution context will include a valid player reference and world access.

A key architectural choice is the delegation of the core logic. RedoCommand does not perform the "redo" operation itself. Instead, it validates the request and queues a deferred task with the BuilderToolsPlugin. This decouples the command parsing from the world-mutating logic, ensuring that all builder tool operations are processed centrally and in a thread-safe manner during the appropriate game tick phase.

The class also demonstrates a common pattern for handling command variants through the use of a nested class, RedoWithCountCommand, which manages the signature for `/redo <count>`.

## Lifecycle & Ownership
-   **Creation:** A single instance of RedoCommand is created by the server's command registry during plugin loading or server bootstrap. The system scans for classes extending command base types and instantiates them.
-   **Scope:** The command object is a long-lived singleton for the duration of the server's runtime. It is held within a central command map and is not garbage collected until the server shuts down or the parent plugin is unloaded.
-   **Destruction:** The instance is destroyed when the command registry is cleared during server shutdown.

## Internal State & Concurrency
-   **State:** RedoCommand is stateless. It contains no mutable instance fields. All necessary information for its operation (the player, the world, command arguments) is passed into the execute method at the time of invocation. This design makes the object inherently reusable and simple.
-   **Thread Safety:** The RedoCommand object itself is thread-safe due to its stateless nature. The execute method is invoked by the server's main thread. Crucially, it avoids concurrency issues by not modifying world state directly. The call to BuilderToolsPlugin.addToQueue hands off the operation to a system designed to execute tasks serially on the correct game thread, preventing race conditions with other world modifications.

## API Surface
The primary contract is the inherited execute method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | Entry point for the command. Validates context and queues a redo operation with the BuilderToolsPlugin using a default count of 1. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and executed by the server's command system in response to player chat input. The pattern it demonstrates is the correct way to integrate with the builder tools history system.

The effective result of this command is queuing a task, a pattern developers should follow for similar systems.
```java
// This is what RedoCommand does internally,
// and is the correct pattern for deferring builder tool actions.

// Assume playerComponent and playerRefComponent are available
BuilderToolsPlugin.addToQueue(
    playerComponent,
    playerRefComponent,
    (r, s, componentAccessor) -> s.redo(r, 1, componentAccessor)
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new RedoCommand()`. The command system manages the lifecycle of command objects. Manual instantiation serves no purpose as the object would not be registered to handle any chat commands.
-   **Direct Execution:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, argument parsing, and context injection, which will lead to NullPointerExceptions and inconsistent state.
-   **Implementing Redo Logic:** Do not replicate the redo logic within a command. The command's sole responsibility is to act as a thin translation layer. All business logic must be delegated to the appropriate service, in this case, the BuilderToolsPlugin, to maintain central control and thread safety.

## Data Pipeline
The flow of data and control for a redo command is unidirectional, starting from player input and resulting in a world state change.

> Flow:
> Player Input (`/redo 5`) → Server Command Parser → **RedoCommand** → BuilderToolsPlugin Task Queue → Server Game Tick → HistoryManager.redo() → World State Mutation & Network Sync

