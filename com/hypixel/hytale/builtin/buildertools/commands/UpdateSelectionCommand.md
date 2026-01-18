---
description: Architectural reference for UpdateSelectionCommand
---

# UpdateSelectionCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class UpdateSelectionCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The UpdateSelectionCommand class is a server-side command handler responsible for processing player requests to modify their builder tool selection area. It serves as a direct, programmatic interface for players to define a cuboid region by supplying six integer coordinates.

Architecturally, this class acts as a translator and a delegator. It translates a raw text command from a player into a structured, validated request. Its primary responsibility is not to perform the selection update itself, but to safely delegate this task to the core BuilderToolsPlugin. This is achieved by enqueuing a deferred task, a critical design pattern that decouples the immediate command execution from the potentially resource-intensive world modification logic. This ensures the server's main command processing thread remains responsive and centralizes all builder tool operations within the owning plugin.

This command is restricted to players in Creative mode, enforcing a clear separation of gameplay mechanics.

### Lifecycle & Ownership
- **Creation:** A singleton instance of UpdateSelectionCommand is instantiated by the server's Command System during the plugin loading phase. It is automatically discovered and registered based on its class definition.
- **Scope:** The registered instance persists for the entire server session, available to be invoked whenever a player executes the corresponding chat command.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down or the BuilderToolsPlugin is unloaded.

## Internal State & Concurrency
- **State:** The internal state of this class is effectively immutable after construction. It holds final references to six RequiredArg objects, which define the command's signature and argument parsing rules. No mutable state is maintained across executions.
- **Thread Safety:** This class is thread-safe. The execute method is invoked by a dedicated command processing thread. As the class itself is stateless, multiple invocations do not interfere with each other. The critical operation, `BuilderToolsPlugin.addToQueue`, is designed to be a thread-safe entry point for submitting tasks that modify world state, offloading the concurrency management to the plugin's internal queue processor.

**WARNING:** While this class is safe, developers must not assume that the lambda passed to the queue is executed immediately or on the same thread. It is a deferred, asynchronous operation.

## API Surface
The primary API is the `execute` method, which is invoked by the engine's command system. Direct invocation is strongly discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | protected void | O(1) | Parses coordinates from the context and enqueues a selection update task with the BuilderToolsPlugin. Throws exceptions on invalid argument types. |

## Integration Patterns

### Standard Usage
This command is intended to be used exclusively by players through the in-game chat console. The server's command system handles parsing, permission checks, and invocation.

```
// Player-invoked chat command
/updateselection 0 64 0 16 80 16
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new UpdateSelectionCommand()`. The command framework manages the lifecycle of all registered commands. Direct instantiation will result in a non-functional object that is not registered with the server.
- **Manual Execution:** Do not call the `execute` method directly. Doing so bypasses critical infrastructure, including permission validation, argument parsing, and context setup, which will lead to runtime errors and unpredictable server state.

## Data Pipeline
The flow of data for this command begins with player input and ends with a deferred task execution that modifies the player's state.

> Flow:
> Player Chat Input -> Server Command Parser -> Permission & Argument Validation -> **UpdateSelectionCommand.execute()** -> BuilderToolsPlugin.addToQueue(task) -> [Deferred] Task Runner -> Player Selection State Update

