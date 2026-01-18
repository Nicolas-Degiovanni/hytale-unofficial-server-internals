---
description: Architectural reference for ServerStatsMemoryCommand
---

# ServerStatsMemoryCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.server
**Type:** Transient

## Definition
```java
// Signature
public class ServerStatsMemoryCommand extends CommandBase {
```

## Architecture & Concepts
The ServerStatsMemoryCommand is a concrete implementation of the Command Pattern, designed to operate within the server's command processing system. Its primary function is to act as a diagnostic tool for server administrators, providing a real-time snapshot of both operating system and Java Virtual Machine memory utilization.

This class serves as an adapter between the high-level Hytale Command System and the low-level Java Management Extensions (JMX) API. It queries standard JMX beans (MemoryMXBean, OperatingSystemMXBean) to gather raw memory data. A key architectural detail is its conditional logic to probe for the Sun/Oracle-specific OperatingSystemMXBean implementation, which provides more granular details like physical and swap memory. If this specific bean is not present, the command gracefully degrades, reporting only the standard JVM memory metrics.

Data retrieved from JMX is not presented raw. Instead, it is processed through the server's localization and formatting utilities (Message, FormatUtil) to produce human-readable, localized output for the command issuer.

## Lifecycle & Ownership
- **Creation:** A single instance of ServerStatsMemoryCommand is created during server initialization by the central CommandRegistry. The registry is responsible for discovering and instantiating all command handlers.
- **Scope:** The object is a stateless singleton managed by the CommandRegistry. It persists for the entire duration of the server's runtime.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. It contains several static final Message fields which are immutable compile-time constants. It does not cache any data or maintain any state between invocations of its execution method. Each execution performs live queries against the JMX API.
- **Thread Safety:** The `executeSync` method implies it is designed to be executed on a synchronous, single-threaded command processing loop. The underlying JMX API calls are thread-safe. However, the command handler itself is not designed for concurrent invocation. The server's command system guarantees that each command execution is handled sequentially, preventing race conditions or interleaved output for a given user.

## API Surface
The public contract is almost exclusively defined by the `executeSync` method inherited from CommandBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(1) | Fetches OS and JVM memory data via JMX and sends formatted messages to the context's issuer. The operation is fast but involves system-level I/O. |

## Integration Patterns

### Standard Usage
This class is not intended to be used programmatically by other game systems. It is designed to be invoked exclusively by the server's command dispatcher in response to user input from the console or an in-game client with sufficient permissions.

The conceptual flow within the command system is as follows:

```java
// A user executes the command "/stats memory"
// The following is a conceptual representation of the system's actions.

// 1. The system parses the input and finds the corresponding command.
CommandBase command = commandRegistry.findCommand("memory");

// 2. A context for the execution is created.
CommandContext context = buildContextForIssuer(issuer);

// 3. The command is executed with the context.
// This internally calls the executeSync method on the ServerStatsMemoryCommand instance.
command.execute(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ServerStatsMemoryCommand()`. Command objects must be managed by the server's CommandRegistry to ensure they are properly registered, aliased, and discoverable.
- **Manual Execution:** Avoid calling `executeSync` directly. Doing so bypasses critical infrastructure such as permission checks, argument parsing, and asynchronous dispatching logic provided by the command system.
- **Stateful Modification:** Do not modify this class to hold state. A single instance is shared across all command executions, and adding mutable fields would introduce severe concurrency bugs.

## Data Pipeline
The flow of data for this command is unidirectional, originating from a user request and terminating with formatted output sent back to that user.

> Flow:
> User Input (`/stats memory`) -> Command System Parser -> **ServerStatsMemoryCommand** -> JMX API -> Raw Memory Data -> Message Formatter -> Formatted Output -> Command Context -> User Console/Chat

