---
description: Architectural reference for ExpandCommand
---

# ExpandCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class ExpandCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ExpandCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's command dispatch system. It serves as the primary entry point for players to manipulate a build tool selection volume via text commands.

Architecturally, this class acts as a translator and a delegator. It translates the raw string input from a player into a structured, deferred world modification task. It does **not** perform the selection expansion itself. Instead, it calculates the required parameters (direction and distance) and enqueues a functional task into the BuilderToolsPlugin. This decouples the user-facing command interface from the core world-editing logic, allowing the BuilderToolsPlugin to manage, schedule, and execute modifications in a controlled, potentially asynchronous, and transactional manner.

Its inheritance from AbstractPlayerCommand ensures it adheres to the server's standard command lifecycle, including permission checks, argument parsing, and context injection.

## Lifecycle & Ownership
- **Creation:** A single instance of ExpandCommand is instantiated by the server's command registry, typically during the loading phase of the BuilderToolsPlugin. It is registered under the name "expand".
- **Scope:** The command instance is a singleton that persists for the entire server session, or until the owning plugin is unloaded. The execution context and state for any given command invocation are transient and scoped only to the `execute` method call.
- **Destruction:** The object is eligible for garbage collection when the server shuts down or the BuilderToolsPlugin is disabled, at which point the command registry releases its reference.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The fields `distanceArg` and `axisArg` are immutable definitions for the command argument parser, configured once during construction. All operational state is passed into the `execute` method via its parameters, primarily the CommandContext.
- **Thread Safety:** The `execute` method is designed to be called exclusively by the server's main thread as part of the command processing loop. It is not thread-safe for concurrent invocations. The critical hand-off to `BuilderToolsPlugin.addToQueue` is the primary mechanism for ensuring that the subsequent world modification is executed in a thread-safe context managed by the plugin's own concurrency model.

## API Surface
The primary API is the command's registration, not direct method invocation. The `execute` method is the functional core but is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ExpandCommand() | constructor | O(1) | Initializes the command name, description, permission group, and argument parsers. |
| execute(...) | protected void | O(1) | Parses arguments from context, determines expansion vector, and enqueues a deferred task in the BuilderToolsPlugin. Throws exceptions on invalid arguments. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked automatically by the server's command system when a player with creative mode permissions executes the command in chat.

**Player Input:**
```
/expand 10
/expand 5 up
```

**System-level Invocation (Conceptual):**
```java
// The server's command dispatcher invokes the command
// This is a conceptual representation and should not be replicated.
CommandSystem commandSystem = server.getCommandSystem();
CommandContext context = commandSystem.createContextFromPlayerInput("/expand 10", player);
ExpandCommand command = commandSystem.getCommand("expand", ExpandCommand.class);

// The system calls execute, injecting necessary ECS and world state
command.execute(context, store, ref, playerRef, world);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ExpandCommand()`. The command system manages the lifecycle of registered command instances. Creating your own instance will result in a non-functional object that is not registered to handle any chat input.
- **Manual Invocation:** Do not call the `execute` method directly. This bypasses the server's entire command processing pipeline, including critical permission checks and context-aware argument parsing. This will lead to unpredictable behavior and likely cause NullPointerExceptions.
- **Assuming Synchronous Execution:** The command enqueues a task for later execution. Any logic that depends on the result of the expansion must not immediately follow the command's execution. Instead, it should hook into the event system or the BuilderToolsPlugin's post-execution callbacks, if available.

## Data Pipeline
The flow of data for an expansion operation begins with player input and terminates with a change to the world state, with ExpandCommand acting as an early-stage processor.

> Flow:
> Player Chat Input (`/expand 10 up`) -> Server Command Parser -> **ExpandCommand.execute** -> BuilderToolsPlugin Task Queue -> World Edit Processor -> World State Change

