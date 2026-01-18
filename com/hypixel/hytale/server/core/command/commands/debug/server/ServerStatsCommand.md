---
description: Architectural reference for ServerStatsCommand
---

# ServerStatsCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.server
**Type:** Structural Component / Command Group

## Definition
```java
// Signature
public class ServerStatsCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The ServerStatsCommand class is a structural component within the server's command processing system. It does not implement any direct server logic itself. Instead, it functions as a **Composite** in the Command design pattern, acting as a container or namespace for a group of related sub-commands.

Its primary architectural role is to organize debugging and administrative commands under a single, discoverable entry point: *stats*. By extending AbstractCommandCollection, it inherits the necessary machinery to register and delegate execution to its children, such as ServerStatsCpuCommand and ServerStatsMemoryCommand.

This approach decouples the command hierarchy from the execution logic, promoting modularity and maintainability. New statistical commands can be added as sub-commands without altering the top-level command registry.

### Lifecycle & Ownership
- **Creation:** Instantiated a single time during the server bootstrap sequence. A central CommandRegistry service is responsible for discovering and registering all command implementations, including this one.
- **Scope:** Application-scoped. The instance is created once and held in memory by the CommandRegistry for the entire lifetime of the server process.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only upon server shutdown when the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** The state of this object is effectively **immutable** after construction. The list of sub-commands is populated within the constructor and is not intended for modification during runtime. The state is managed by the parent AbstractCommandCollection.

- **Thread Safety:** This class is inherently thread-safe. As its internal state is established at construction and not mutated, it can be safely accessed by the command system's dispatcher thread without synchronization. Any potential concurrency concerns reside within the execution logic of its sub-commands, not within this container class.

## API Surface
The public contract is minimal and exclusively focused on instantiation. All other functionality is inherited from AbstractCommandCollection and utilized internally by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ServerStatsCommand() | Constructor | O(k) | Initializes the command group with the name "stats" and registers its constituent sub-commands. Complexity is dependent on k, the number of sub-commands added. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by application code. It is designed to be discovered and managed exclusively by the server's command registration and dispatching system. The standard interaction is initiated by a server administrator via the console.

A system-level registration process would resemble the following:
```java
// Example of how the CommandRegistry might register this command group
CommandRegistry registry = serverContext.getCommandRegistry();
registry.register(new ServerStatsCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not instantiate this class outside of the server's initial command registration phase. Creating orphaned instances serves no purpose as they will not be wired into the command dispatcher.
- **Runtime Modification:** Do not retrieve this object from the registry to dynamically add or remove sub-commands. The command hierarchy is expected to be static after server startup. Such modifications can lead to unpredictable behavior and are unsupported.

## Data Pipeline
The ServerStatsCommand acts as a routing node in the data pipeline for server commands. It receives a parsed command request and delegates it to the appropriate sub-command for execution.

> Flow:
> User Console Input (`/stats memory`) -> Network Layer -> Command Parser -> CommandRegistry (lookup "stats") -> **ServerStatsCommand** (delegates "memory") -> ServerStatsMemoryCommand (executes) -> Formatted Response -> User Console Output

