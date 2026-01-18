---
description: Architectural reference for ServerDumpCommand
---

# ServerDumpCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.server
**Type:** Component

## Definition
```java
// Signature
public class ServerDumpCommand extends CommandBase {
```

## Architecture & Concepts
The ServerDumpCommand class is a concrete implementation of the command pattern, designed to integrate into the server's command processing system. Its sole responsibility is to provide an in-game or console interface for server administrators to trigger a full dump of the server's runtime state for debugging and analysis.

This class acts as a thin adapter between the command system and the core dumping logic, which is encapsulated within the DumpUtil utility. It parses command-line arguments, specifically the presence of a JSON flag, and then orchestrates the call to the appropriate DumpUtil method. It is a diagnostic tool, not intended for use in standard gameplay logic.

## Lifecycle & Ownership
- **Creation:** Instantiated once by the server's command registration service during the bootstrap sequence. The system typically discovers all subclasses of CommandBase and creates a singleton instance of each to populate the command registry.
- **Scope:** The object instance persists for the entire lifetime of the server. It is a stateless component whose methods are invoked in response to user actions.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down and its command registry is cleared.

## Internal State & Concurrency
- **State:** The class holds a reference to a FlagArg instance, jsonFlag. This object defines the `--json` command-line flag. This state is configured in the constructor and is **immutable** for the remainder of the object's lifecycle. The class itself holds no mutable state related to command execution.
- **Thread Safety:** The primary execution method, executeSync, is designed to be called synchronously from the server's main logic thread or a dedicated command-handling thread. The class is inherently thread-safe due to its stateless nature regarding execution. However, the delegated DumpUtil methods perform I/O and may traverse large, complex object graphs, which is a blocking and potentially long-running operation. The framework ensures that executeSync is not invoked concurrently for the same command.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(N) | Executes the state dump. Complexity is proportional to N, the size of the server state being dumped. It checks for the presence of the JSON flag and delegates to DumpUtil. Sends status messages back to the caller via the CommandContext. Throws a wrapped IOException on file system errors. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and executed by the command system. A user, such as a server administrator, would trigger its execution via the server console or in-game chat.

*Console Input:*
```console
dump --json
```

*System-level invocation (conceptual):*
```java
// The CommandSystem finds and executes the command
// This code is internal to the server and not for external use.
CommandBase command = commandRegistry.find("dump");
CommandContext context = buildContextForIssuer(issuer, "--json");
command.execute(context); // This will internally call executeSync
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ServerDumpCommand()`. The command must be managed by the server's central command registry to function correctly.
- **Manual Execution:** Avoid calling `executeSync` directly. Doing so bypasses critical framework features like permission checks, argument parsing, and context creation, which can lead to unpredictable behavior and NullPointerExceptions.

## Data Pipeline
The data flow for this command is initiated by a user and results in a file on the server's disk.

> Flow:
> User Input (`/dump`) -> Command Parser -> **ServerDumpCommand.executeSync** -> DumpUtil -> Server State Serialization -> Filesystem Write -> CommandContext.sendMessage -> User Feedback

