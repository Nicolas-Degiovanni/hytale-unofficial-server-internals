---
description: Architectural reference for DebugCommand
---

# DebugCommand

**Package:** com.hypixel.hytale.server.core.modules.debug.commands
**Type:** Transient

## Definition
```java
// Signature
public class DebugCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The DebugCommand class is a concrete implementation of the Command pattern, specifically designed to act as a container or namespace for a group of related debugging subcommands. It extends AbstractCommandCollection, which provides the foundational logic for managing a hierarchy of commands.

In the server's command processing system, this class represents the top-level `/debug` command. It does not contain any execution logic itself; its sole responsibility is to group child commands (like DebugShapeSubCommand) and delegate execution to them based on the arguments provided by the user. This architectural choice promotes modularity and organization by preventing the pollution of the global command namespace. It allows developers to logically group related functionalities under a single, memorable entry point.

## Lifecycle & Ownership
- **Creation:** An instance of DebugCommand is created during the server's bootstrap sequence. A central command registry or module loader is responsible for discovering and instantiating all command classes, including this one.
- **Scope:** The object is a long-lived transient. Once instantiated and registered with the command system, it persists in memory for the entire lifecycle of the server process.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection only upon server shutdown when the central command registry is cleared. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state of a DebugCommand instance is effectively **immutable** after its constructor completes. The command name ("debug"), description key, and the list of subcommands are initialized once and are not designed to be modified at runtime.
- **Thread Safety:** The object itself is thread-safe due to its immutable nature post-construction. However, command execution is dispatched by the server's main thread. Any logic within its subcommands is **not** thread-safe by default and must assume it is operating on the primary game loop thread.

**WARNING:** Do not attempt to modify the subcommand list from another thread after the object has been registered with the server. This will lead to concurrency violations and undefined behavior.

## API Surface
The public API is limited to the constructor, as the class is intended for declaration and registration, not for direct interaction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DebugCommand() | Constructor | O(N) | Initializes the command with its name and description. Adds all defined subcommands. N is the number of subcommands. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Instead, they create new command collections and register them with the server's command management system. The system handles the lifecycle and invocation.

To add a new debug subcommand, a developer would modify the DebugCommand constructor:

```java
// Example: Adding a new subcommand
public class DebugCommand extends AbstractCommandCollection {
   public DebugCommand() {
      super("debug", "server.commands.debug.desc");
      this.addSubCommand(new DebugShapeSubCommand());
      // Add a new, hypothetical subcommand here
      this.addSubCommand(new DebugNetworkSubCommand());
   }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of DebugCommand manually via `new DebugCommand()`. The object will be created but will not be registered with the server's command processor, rendering it non-functional.
- **State Mutation:** Do not attempt to call `addSubCommand` or otherwise modify the object's state after it has been constructed and registered. The command system does not expect the command hierarchy to change at runtime.

## Data Pipeline
The DebugCommand acts as a routing node in the server's command processing pipeline. It receives control from the central command parser and delegates it to the appropriate subcommand.

> Flow:
> User Input (`/debug shape ...`) -> Network Layer -> Command Parser -> Command Registry -> **DebugCommand** (Routing) -> DebugShapeSubCommand (Execution) -> Game World State

