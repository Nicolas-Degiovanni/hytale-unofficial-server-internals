---
description: Architectural reference for ServerStatsCpuCommand
---

# ServerStatsCpuCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.server
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ServerStatsCpuCommand extends CommandBase {
```

## Architecture & Concepts
The ServerStatsCpuCommand is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. Its sole responsibility is to act as a bridge between a user-invoked action and the Java Management Extensions (JMX) API, specifically for querying CPU and process metrics.

This class encapsulates a single, synchronous, read-only operation. It is considered a "debug" command, providing server administrators with real-time performance diagnostics.

A key architectural feature is its graceful degradation capability. The command attempts to query the proprietary `com.sun.management.OperatingSystemMXBean` for detailed metrics like process-specific CPU load. If this extended bean is unavailable (e.g., when running on a non-Oracle/OpenJDK JVM), the command falls back to providing only the universally available metrics, such as system load average and uptime, preventing runtime errors and ensuring baseline functionality across different environments.

## Lifecycle & Ownership
-   **Creation:** A single instance of ServerStatsCpuCommand is instantiated by the server's `CommandManager` or an equivalent bootstrap service during server startup. It is discovered alongside all other `CommandBase` implementations and registered into a central command map.
-   **Scope:** Application-scoped. The singleton instance persists for the entire lifecycle of the server, held as a strong reference by the command registration system.
-   **Destruction:** The object is marked for garbage collection when the server shuts down and the `CommandManager` clears its command registry.

## Internal State & Concurrency
-   **State:** This object is effectively stateless. The `Message` fields are static, final, and immutable references to translation keys. All operational state, such as the command sender and context, is passed into the `executeSync` method and is confined to the method's stack frame. It performs no caching.
-   **Thread Safety:** This class is **not thread-safe** and must not be treated as such. The `executeSync` method name is a strict contract indicating that the command system will invoke it from a designated synchronous execution thread, typically the main server game loop or a dedicated command processing thread. Concurrent invocations would lead to race conditions within the `CommandContext` and its underlying network streams.

## API Surface
The public API is minimal, primarily consisting of the constructor for framework instantiation and the overridden execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ServerStatsCpuCommand() | constructor | O(1) | Initializes the command with its name and description key. For framework use only. |
| executeSync(CommandContext) | void | O(1) | Executes the CPU statistics query. This is a blocking, synchronous call. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use by other services. It is designed to be invoked exclusively by the server's command dispatch system in response to user input (e.g., a console administrator typing `stats cpu`). The following conceptual example illustrates how the framework interacts with the object.

```java
// Conceptual: How the Command Dispatcher invokes the command
// NOTE: Do not replicate this pattern in application code.

CommandBase command = commandRegistry.getCommand("cpu");
if (command != null) {
    // The framework provides the necessary context (sender, args, etc.)
    CommandContext context = createCommandContextForSender(sender);
    command.execute(context); // Base class routes to executeSync
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ServerStatsCpuCommand()`. The command system manages the lifecycle of command objects. Direct instantiation will result in an un-registered command that is never used.
-   **Direct Invocation:** Avoid calling the `executeSync` method directly. Bypassing the server's command dispatcher will circumvent critical infrastructure such as permission checks, logging, and context setup.
-   **Output Parsing:** Do not build automated systems that parse the string output of this command. The output is for human consumption, is subject to change, and relies on localization that will vary. For programmatic access to these metrics, use the JMX APIs directly.

## Data Pipeline
The flow for this command is initiated by user input and terminates with a network message sent back to the user. It does not transform or pass data to other internal systems.

> Flow:
> User Input -> Command Parser -> Command Dispatcher -> **ServerStatsCpuCommand.executeSync** -> Java ManagementFactory -> OperatingSystemMXBean -> Message Formatter -> CommandContext -> Network Subsystem -> Client

