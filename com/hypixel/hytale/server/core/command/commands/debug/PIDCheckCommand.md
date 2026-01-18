---
description: Architectural reference for PIDCheckCommand
---

# PIDCheckCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Singleton

## Definition
```java
// Signature
public class PIDCheckCommand extends CommandBase {
```

## Architecture & Concepts
The PIDCheckCommand is a server-side debug command that provides an interface for administrators to verify the running status of a process on the host machine. It is a concrete implementation of the abstract CommandBase, designed to be discovered and registered by the server's command system at startup.

Architecturally, this class serves as a leaf node in the command processing tree. It translates a user-facing text command into a system-level query using the ProcessUtil helper. Its primary function is to act as a diagnostic tool, especially in single-player or locally hosted environments, to confirm that an associated client process is still active. The command's logic branches based on the provided arguments, either checking a specific Process ID (PID) or automatically checking the known client PID in a single-player session.

## Lifecycle & Ownership
- **Creation:** A single instance of PIDCheckCommand is created by the server's command registration service during the server bootstrap sequence. The framework is responsible for its instantiation.
- **Scope:** The object is stateless and its lifetime is tied to the server's runtime. It persists from the moment it is registered until the server shuts down.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the command registry is cleared during server shutdown.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The instance fields, singleplayerFlag and pidArg, are argument *definitions* configured immutably in the constructor. They do not hold state related to any specific command execution; rather, they are used to parse state from the CommandContext object passed into executeSync.
- **Thread Safety:** The primary entry point, executeSync, is designed to be called synchronously from the main server thread. The implementation does not modify any shared mutable state and is therefore safe within its intended single-threaded execution model. Direct invocation from other threads is not supported and would violate the command system's threading contract. The underlying call to ProcessUtil.isProcessRunning is expected to be thread-safe.

## API Surface
The public contract is defined by the CommandBase superclass. The constructor is public for instantiation by the framework, and executeSync is the command's execution logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PIDCheckCommand() | constructor | O(1) | Initializes the command name and argument definitions. |
| executeSync(context) | void | O(1) | Executes the PID check. Logic is dispatched based on arguments present in the CommandContext. |

## Integration Patterns

### Standard Usage
This command is not intended to be used programmatically. It is invoked by a user, typically a server administrator, through the server console or in-game chat. The command system handles parsing, routing, and invocation.

*Example User Input:*
```text
/pidcheck 12345
/pidcheck --singleplayer
```

The framework then populates a CommandContext and invokes the command instance:
```java
// Framework-level pseudo-code
CommandContext context = commandParser.parse("/pidcheck 12345");
CommandBase command = commandRegistry.get("pidcheck");
command.executeSync(context); // Invokes the PIDCheckCommand instance
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PIDCheckCommand()`. An instance created this way will not be registered with the server's command system and will never be executed in response to user input.
- **Direct Invocation:** Avoid calling the executeSync method directly. This bypasses the command system's parsing, permission checks, and context-building logic, which can lead to NullPointerExceptions or incorrect behavior.

## Data Pipeline
The flow of data for this command begins with user input and ends with a message sent back to the user's client. The PIDCheckCommand acts as the central processing component in this specific flow.

> Flow:
> User Command String -> Server Command Parser -> CommandContext Creation -> **PIDCheckCommand.executeSync** -> ProcessUtil -> OS Process Query -> Message Object -> CommandContext.sendMessage -> Network Layer -> Client Chat UI

