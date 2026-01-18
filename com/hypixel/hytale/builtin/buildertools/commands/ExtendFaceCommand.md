---
description: Architectural reference for ExtendFaceCommand
---

# ExtendFaceCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Service Component

## Definition
```java
// Signature
public class ExtendFaceCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The ExtendFaceCommand class serves as a high-level command dispatcher, not a direct implementation of logic. It functions as a container for multiple command "variants" that all fall under the `/extendface` command name. This architectural pattern allows a single command to have multiple signatures, which the server's command system can resolve based on the arguments provided by the user.

Its primary responsibility is to act as a translation layer between the user-facing command system and the backend `BuilderToolsPlugin`. Upon execution, it does not perform any world modification itself. Instead, it validates and parses user input, then serializes the request into a lambda function. This function is then enqueued with the `BuilderToolsPlugin` for asynchronous processing. This decoupling is critical for server performance, preventing complex, potentially long-running world edit operations from blocking the main command processing thread.

The class utilizes two private inner classes, `ExtendFaceBasicCommand` and `ExtendFaceWithRegionCommand`, to define the specific argument contracts for each variant of the command.

## Lifecycle & Ownership
-   **Creation:** A single instance of ExtendFaceCommand is created by the `BuilderToolsPlugin` during the server's plugin initialization phase. It is then registered with the central `CommandSystem`.
-   **Scope:** The registered instance persists for the entire lifecycle of the server. It is effectively a singleton within the context of the command registry.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the `BuilderToolsPlugin` is unloaded or the server shuts down, at which point the `CommandSystem` clears its registrations.

## Internal State & Concurrency
-   **State:** This class is stateless. Its initial configuration, including its name and registered variants, is defined in the constructor and is immutable thereafter. The inner command classes are also stateless, serving only as definitions for argument parsing and execution logic.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. The command system guarantees that the `execute` method of its variants is called in a controlled, single-threaded manner per command invocation. The most critical operation, `BuilderToolsPlugin.addToQueue`, is a thread-safe method designed to marshal data from the command thread to a separate worker or the main server tick thread.

## API Surface
The public API of this class is designed for consumption by the Hytale command system, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ExtendFaceCommand() | constructor | O(1) | Configures the command name, permission group (Creative), and registers its two execution variants. |
| execute (internal) | void | O(1) | Invoked by the command system. Parses arguments and delegates the operation to the BuilderToolsPlugin queue. Throws exceptions on invalid argument types. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is registered once by its parent plugin and subsequently invoked by players through the in-game chat or server console. The correct integration is at the plugin level.

```java
// Example of how the BuilderToolsPlugin registers this command
// This code would exist within the plugin's onEnable() method.

CommandSystem commandSystem = server.getCoreApi().getCommandSystem();
commandSystem.register(new ExtendFaceCommand());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ExtendFaceCommand()` in your own logic. The class provides no functionality outside of its registration with the `CommandSystem`.
-   **Direct Execution:** Never invoke the `execute` method on the inner command classes directly. This will fail, as it requires a complete `CommandContext` and other engine-provided state that can only be constructed by the command system itself.
-   **Synchronous World Edits:** Do not modify this class to perform world edits within the `execute` method. The asynchronous queuing pattern is a deliberate design choice to protect server performance. Bypassing it will lead to server stalls and unresponsiveness.

## Data Pipeline
The primary function of this class is to act as an entry point in a larger data processing pipeline for world modification.

> Flow:
> Player Chat Input (`/extendface ...`) -> Server Network Layer -> Command System Parser -> **ExtendFaceCommand.execute** -> BuilderToolsPlugin Queue -> World Edit Worker -> Voxel State Change -> Network Replication to Clients

