---
description: Architectural reference for HollowCommand
---

# HollowCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Command Definition

## Definition
```java
// Signature
public class HollowCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The HollowCommand class is a server-side command definition that implements the user-facing `/hollow` command. It serves as the primary interface between player input and the world modification logic for the "hollowing" builder tool operation.

Architecturally, this class is a stateless handler within the server's command processing system. Its sole responsibility is to define the command's syntax—including its name, arguments, and flags—and to translate a valid player command into a deferred task.

A critical design pattern employed here is the **Command-Queue-Executor** pattern. HollowCommand does not perform the computationally expensive world modification itself. Instead, upon successful execution, it validates the player's selection and arguments, then serializes the operation into a lambda function. This function is then submitted to the **BuilderToolsPlugin** work queue. This decouples the command parsing logic from the world modification engine, preventing command execution from blocking the main server thread and allowing for batching, prioritization, and safer world state manipulation.

This class relies heavily on the server's declarative argument parsing system. Fields like **blockTypeArg** and **thicknessArg** are not instance state; they are immutable definitions that instruct the command system how to parse and validate player input before the **execute** method is ever called.

## Lifecycle & Ownership
- **Creation:** A single instance of HollowCommand is created by the server's command registration system during the bootstrap phase, typically when the **BuilderToolsPlugin** is loaded. It is not instantiated on a per-player or per-command basis.

- **Scope:** The object is a long-lived singleton for the duration of the server's runtime. It persists as long as the parent plugin is active.

- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down or the **BuilderToolsPlugin** is unloaded, at which point the command registry is cleared.

## Internal State & Concurrency
- **State:** HollowCommand is fundamentally **stateless and immutable** after construction. The member fields are definitions for command arguments and are configured once in the constructor. They do not change during the object's lifetime. All state required for an operation is passed into the **execute** method via its parameters.

- **Thread Safety:** This class is inherently thread-safe. As a stateless definition, it can be safely registered and accessed by the command system without locks or synchronization. The **execute** method is invoked by the server's command processing thread. Any potential concurrency issues related to world modification are deferred to the **BuilderToolsPlugin** queue, which is responsible for managing its own threading and synchronization model.

## API Surface
The public API is exclusively consumed by the server's command system. Direct developer interaction is not intended.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Overrides **AbstractPlayerCommand**. Parses arguments from the context and queues a world modification task. Complexity is constant as it only enqueues a job, not executes it. |

## Integration Patterns

### Standard Usage
This class is not used directly in code by developers. It is invoked by the server's command dispatcher in response to a player executing the command in-game.

The standard interaction is via the game client's chat console:
```
/hollow stone 2 --floor --roof
```
This command instructs the system to hollow out the player's current selection, leaving a 2-block thick wall of stone, including a floor and a roof. The internal invocation by the command system would conceptually resemble the following:

```java
// Conceptual example of internal system invocation. DO NOT use this directly.
CommandContext context = createCommandContextForPlayer("/hollow stone 2 --floor --roof");
HollowCommand command = commandRegistry.get("hollow");

// The system validates permissions and then executes
if (player.hasPermission(command.getPermissionGroup())) {
    command.execute(context, ...);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new HollowCommand()`. The command system is responsible for the lifecycle of command objects. Direct instantiation will result in a non-functional command that is not registered with the server.

- **Direct Invocation:** Avoid calling the **execute** method directly. Doing so bypasses the command system's critical infrastructure, including permission checks, argument parsing, and context setup. This will lead to unpredictable behavior and likely throw a NullPointerException.

- **State Modification:** Attempting to modify the argument definitions (**blockTypeArg**, **thicknessArg**, etc.) after construction via reflection or other means is unsupported and will break the command's behavior for all users.

## Data Pipeline
The flow of data for a `/hollow` command follows a clear, asynchronous path from player input to world modification.

> Flow:
> Player Chat Input (`/hollow ...`) -> Server Network Packet -> Command Dispatcher -> **HollowCommand.execute()** -> Argument Validation -> **BuilderToolsPlugin.addToQueue(lambda)** -> World Modification Worker Thread -> World State Change -> Network Update to Client

