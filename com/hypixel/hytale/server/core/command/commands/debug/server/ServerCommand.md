---
description: Architectural reference for ServerCommand
---

# ServerCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.server
**Type:** Transient

## Definition
```java
// Signature
public class ServerCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The ServerCommand class is an implementation of the **Composite Pattern** within the server's command handling system. It functions as a non-executable, organizational node in the command tree, designed exclusively to group related administrative and debugging commands under a single, top-level namespace: *server*.

Architecturally, this class does not contain any command logic itself. Its sole responsibility is to act as a parent container and dispatcher for its registered sub-commands, such as ServerStatsCommand and ServerGCCommand. When the command system processes an input like `/server stats`, it first resolves the "server" token to this ServerCommand instance. This instance then takes responsibility for resolving the next token, "stats", and delegating execution to the appropriate child command.

This approach provides a clean, hierarchical command structure that is easily extensible. By inheriting from AbstractCommandCollection, it leverages a shared framework for sub-command registration and dispatch, promoting code reuse and a consistent command system design.

### Lifecycle & Ownership
- **Creation:** An instance of ServerCommand is created once during the server's bootstrap sequence. A central CommandRegistry service is responsible for discovering and instantiating all command classes, including this one.
- **Scope:** The object's lifecycle is tied directly to the server session. It is instantiated on server start and persists in memory until the server is shut down.
- **Destruction:** The instance is eligible for garbage collection only upon server shutdown, when the primary CommandRegistry is cleared. There is no manual destruction or cleanup mechanism.

## Internal State & Concurrency
- **State:** The internal state of a ServerCommand instance consists of a collection of its child commands. This state is populated exclusively within the constructor and is considered **immutable** for the remainder of the object's lifecycle. The class itself is stateless regarding server or world data.
- **Thread Safety:** This class is inherently **thread-safe**. Its state is established in a single-threaded context during server initialization and is not modified thereafter. Subsequent access from multiple threads (e.g., different players or console inputs executing commands concurrently) is read-only and therefore safe without requiring explicit locks.

## API Surface
The public contract is almost entirely inherited from AbstractCommandCollection. The only direct symbol is the constructor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ServerCommand() | Constructor | O(1) | Instantiates the command collection, setting its name and description key. Registers a fixed set of child commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is designed to be instantiated and registered with the server's central command authority.

```java
// Correct registration during server initialization
// This logic resides within the command system, not user code.
CommandRegistry registry = server.getCommandRegistry();
registry.register(new ServerCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of ServerCommand for any purpose other than registration. It provides no utility outside the context of the command dispatch system.
- **State Mutation:** Do not attempt to add or remove sub-commands from this collection after its initial construction. The command tree is designed to be static after the server has started. Modifying it at runtime can lead to unpredictable behavior and race conditions.

## Data Pipeline
ServerCommand acts as a routing node in the command processing pipeline. It receives control from the main parser and directs it to a more specific handler.

> Flow:
> Raw User Input (`/server gc`) -> Command Parser -> CommandRegistry (resolves "server") -> **ServerCommand** (resolves "gc") -> ServerGCCommand.execute() -> CommandResult -> Console Output

