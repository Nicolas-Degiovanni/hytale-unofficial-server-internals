---
description: Architectural reference for StopCommand
---

# StopCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server
**Type:** Component

## Definition
```java
// Signature
public class StopCommand extends CommandBase {
```

## Architecture & Concepts
The StopCommand class is a concrete implementation of the Command Pattern, designed to handle a critical server administration task: shutting down the server process. It serves as a direct interface between user or console input and the core server lifecycle management system, represented by HytaleServer.

This class is a leaf node within the server's command hierarchy. Its single responsibility is to parse a specific command string ("stop" or "shutdown"), interpret any provided flags, and trigger the appropriate server shutdown sequence. It decouples the command input layer (console, RCON, in-game chat) from the server's internal shutdown logic, ensuring a consistent and controlled termination process.

The inclusion of the FlagArg for a "crash" state demonstrates how the command system integrates with argument parsing to provide nuanced control over server operations, allowing administrators to simulate a crash for debugging or testing purposes.

## Lifecycle & Ownership
- **Creation:** A single instance of StopCommand is created by the central CommandManager during the server's bootstrap phase. The manager scans for and instantiates all classes extending CommandBase to build its command registry.
- **Scope:** The StopCommand object is a long-lived component. It persists for the entire runtime of the server, held as a reference within the CommandManager's registry.
- **Destruction:** The object is dereferenced and garbage collected only when the HytaleServer process terminates and the JVM is shut down. It has no explicit destruction or cleanup logic.

## Internal State & Concurrency
- **State:** StopCommand is effectively stateless and immutable after construction. Its internal field, crashFlag, is a definition object whose configuration is fixed in the constructor. The state relevant to any single execution (e.g., whether the crash flag was provided) is contained entirely within the CommandContext object passed to the executeSync method, not stored in the StopCommand instance itself.
- **Thread Safety:** This class is thread-safe under its intended execution model. The `executeSync` method name strongly implies that the command system dispatches it on a synchronized main server thread. The method's logic is inherently safe as it reads from immutable fields and makes a call to HytaleServer, which is expected to manage its own concurrency for critical operations like shutdown.

## API Surface
The public contract is primarily defined by its constructor for registration and the overridden `executeSync` method for execution by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| StopCommand() | constructor | O(1) | Initializes the command's name, alias, and flag definitions. |
| executeSync(context) | void | O(1) | Triggers the server shutdown process. This is a terminal operation for the server. |

## Integration Patterns

### Standard Usage
A developer does not typically interact with this class directly. It is instantiated and managed by the command system. The standard interaction is a user typing the command into a server console or chat. The following code illustrates how the system registers the command.

```java
// Example of command registration during server startup
CommandManager commandManager = server.getService(CommandManager.class);
commandManager.register(new StopCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Execution:** Never instantiate and call `executeSync` manually. Doing so bypasses the command system's permission checks, argument parsing, and context setup, which can lead to an uncontrolled and unlogged shutdown.
- **Subclassing for Modification:** Do not extend this class to alter its behavior. If custom shutdown logic is required, it should be implemented through server-level lifecycle hooks, not by modifying this specific command handler.

## Data Pipeline
The StopCommand acts as the final step in a user-initiated data flow that results in a server-wide state change.

> Flow:
> User Input (`/stop --crash`) -> Network/Console Handler -> Command Parser -> **StopCommand**.executeSync() -> HytaleServer.shutdownServer(ShutdownReason.CRASH) -> Server Shutdown Sequence

