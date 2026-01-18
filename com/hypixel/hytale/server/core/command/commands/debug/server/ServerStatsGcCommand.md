---
description: Architectural reference for ServerStatsGcCommand
---

# ServerStatsGcCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.server
**Type:** Transient

## Definition
```java
// Signature
public class ServerStatsGcCommand extends CommandBase {
```

## Architecture & Concepts
The ServerStatsGcCommand is a concrete implementation of the Command Pattern, designed to provide server administrators with real-time introspection into the Java Virtual Machine's memory management system. It functions as a diagnostic tool, bridging the gap between the in-game command system and the low-level Java Management Extensions (JMX) API.

This class does not participate in core game logic. Its sole responsibility is to query the JVM for garbage collection statistics and format them for human-readable output. It is registered with the server's central CommandSystem under the alias *gc* and is typically invoked manually for performance tuning and debugging memory-related issues.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the CommandSystem during server bootstrap. The system likely scans the classpath for all subclasses of CommandBase and registers them automatically.
- **Scope:** The object instance persists for the entire lifetime of the server process. As a stateless object, its memory footprint is negligible.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the CommandSystem is shut down as part of the server termination sequence. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** This class is entirely stateless. The only field, MESSAGE_COMMANDS_SERVER_STATS_GC_USAGE_INFO, is a static final constant, making it immutable. All data reported by the command is fetched on-demand from the ManagementFactory within the execution method.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, the `executeSync` method name strongly implies that the command framework guarantees its execution on the main server thread to prevent concurrency issues with the CommandContext or other game-state systems. Direct invocation from other threads is not a supported use case.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ServerStatsGcCommand() | constructor | O(1) | Registers the command's name ("gc") and its description key with the parent CommandBase. |
| executeSync(CommandContext) | void | O(N) | Iterates over all N registered GarbageCollectorMXBeans in the JVM, formatting and sending their statistics to the command's source. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is designed to be invoked exclusively through the server's command dispatcher, typically by a server administrator via the console or an in-game chat command. The framework handles parsing, routing, and execution.

A conceptual view of the framework's invocation:
```java
// This is a simplified representation of how the CommandSystem would
// find and execute this command. Do not replicate this pattern.

CommandContext context = ...; // Context from player or console
CommandBase command = commandRegistry.findCommand("gc");

if (command instanceof ServerStatsGcCommand) {
    command.executeSync(context);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ServerStatsGcCommand()`. The server's CommandSystem manages the lifecycle of all command objects. Manual instantiation will result in an object that is not registered and therefore cannot be executed.
- **Programmatic Invocation:** Avoid calling the `executeSync` method directly from other parts of the codebase. Doing so bypasses the CommandSystem's middleware, including permission checks, argument parsing, and thread safety guarantees.

## Data Pipeline
The data flow for this command is a simple, synchronous request-response cycle initiated by an external agent (a user or console).

> Flow:
> User Input (`/gc`) -> CommandSystem Parser -> **ServerStatsGcCommand.executeSync** -> JMX ManagementFactory Query -> Message Formatter -> CommandContext.sendMessage -> User Console Output

