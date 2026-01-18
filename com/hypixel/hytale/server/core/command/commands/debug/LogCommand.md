---
description: Architectural reference for LogCommand
---

# LogCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Registered Handler

## Definition
```java
// Signature
public class LogCommand extends CommandBase {
```

## Architecture & Concepts
The LogCommand class is a concrete implementation of the Command Pattern, designed to provide a runtime, text-based interface for server administrators to control the server's logging verbosity. It serves as a direct bridge between the server's command processing system and the core HytaleLoggerBackend.

This command allows for dynamic adjustment of logging levels for any registered logger within the server, including a special "global" logger. Its primary function is for live debugging and diagnostics, enabling administrators to increase or decrease the amount of log output for specific systems without requiring a server restart. It can also persist these changes to the server's configuration file, making them permanent across sessions.

The class defines its own argument structure, including a custom argument type, LOG_LEVEL, which validates input against a predefined list of standard Java logging levels. This self-contained parsing logic makes it a robust and isolated component within the command framework.

## Lifecycle & Ownership
- **Creation:** A single instance of LogCommand is created by the server's command registration system during the server bootstrap sequence. It is not intended for manual instantiation.
- **Scope:** The object instance persists for the entire lifetime of the server. It is held as a registered handler within the central command map or registry.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only upon server shutdown, when the command registry is cleared.

## Internal State & Concurrency
- **State:** The LogCommand instance is effectively stateless regarding command execution. Its fields, such as loggerArg and levelArg, are final definitions of the command's structure and are immutable after construction. All state required for execution, such as the specific logger name and desired level, is provided externally via the CommandContext object for each invocation.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. The `executeSync` method signature strongly implies that the command system guarantees its execution on a synchronized, predictable thread, typically the main server thread. Any operations on shared resources, such as the HytaleServer configuration, are expected to be handled by those systems' own concurrency controls.

## API Surface
The public contract is almost exclusively defined by its role as a CommandBase implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(1) | Executes the command logic. Modifies logger state and potentially server configuration based on parsed arguments from the context. Sends feedback messages to the command source. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly in code. It is used by the server's command processor, which parses user input and dispatches it to the appropriate command handler. The standard interaction is through a text-based command line.

*Example User Interaction:*
```bash
# Sets the global logger to INFO and saves it to the config
/log global INFO --save

# Temporarily sets the network logger to FINEST for debugging
/log network FINEST

# Checks the current level of the global logger
/log global

# Resets the global logger's config entry, reverting to default
/log global --reset
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new LogCommand()`. The command system manages the lifecycle of command objects. Direct creation results in an object that is not registered and will never be executed.
- **Direct Invocation:** Do not call the `executeSync` method directly. This bypasses the entire argument parsing and validation pipeline, and requires manual construction of a CommandContext, which is complex and error-prone. All interaction should be routed through the server's command execution API.

## Data Pipeline
The flow for this command is initiated by user input and results in system side-effects and user feedback.

> Flow:
> Raw String (e.g., `/log global INFO`) -> Command System Parser -> Populated CommandContext -> **LogCommand.executeSync()** -> HytaleLoggerBackend.setLevel() -> HytaleServer.getConfig().setLogLevels() -> CommandContext.sendMessage() -> User Feedback Message

