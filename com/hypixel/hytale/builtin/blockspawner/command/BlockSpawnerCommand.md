---
description: Architectural reference for BlockSpawnerCommand
---

# BlockSpawnerCommand

**Package:** com.hypixel.hytale.builtin.blockspawner.command
**Type:** Transient

## Definition
```java
// Signature
public class BlockSpawnerCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The BlockSpawnerCommand class serves as a top-level command router within the server's command handling system. It does not implement any game logic itself; its sole responsibility is to group related subcommands under a single, unified namespace: *blockspawner*.

This class is a concrete implementation of the **Composite Pattern**. It inherits from AbstractCommandCollection, which provides the core infrastructure for registering and managing a list of child command objects. By composing other commands (BlockSpawnerSetCommand, BlockSpawnerGetCommand), it allows the command parser to treat a group of commands and a single command polymorphically. This simplifies the command tree and provides a clear, hierarchical structure for server administration commands related to block spawners.

## Lifecycle & Ownership
- **Creation:** An instance of BlockSpawnerCommand is created during the server's bootstrap phase. The central command registry service instantiates it, along with all other built-in commands, to populate the server's command map.
- **Scope:** The object's lifecycle is tied to the server session. It persists in memory as part of the command registry for as long as the server is running.
- **Destruction:** The instance is eligible for garbage collection upon server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** The internal state consists of a collection of subcommand instances. This state is populated exclusively within the constructor and is considered **effectively immutable** after initialization. The class itself holds no mutable game-world state.
- **Thread Safety:** This class is **not thread-safe**. Command execution is expected to occur synchronously on the main server thread. The internal collection of subcommands is not protected against concurrent modification, which could lead to unpredictable behavior if accessed from multiple threads after construction.

## API Surface
The programmatic API is minimal, as the primary interaction is through the server console or in-game chat.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BlockSpawnerCommand() | Constructor | O(1) | Initializes the command group, setting its name and registering its constituent subcommands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by game logic or other services. It is designed to be discovered and registered by the server's command system at startup.

```java
// Example of how the command system might register this command
CommandRegistry registry = server.getCommandRegistry();
registry.register(new BlockSpawnerCommand());
```

### Anti-Patterns (Do NOT do this)
- **Post-Construction Modification:** Do not attempt to add or remove subcommands from an instance after it has been constructed. The command collection is not designed for runtime mutation.
- **Direct Execution:** Never instantiate and execute this command's methods directly. All command execution must be routed through the server's central command processor to ensure permissions, syntax, and state are handled correctly.

## Data Pipeline
The flow for this command begins with user input and terminates with a potential modification to the game world. The BlockSpawnerCommand acts as an intermediary router in this flow.

> Flow:
> Player Chat Input (`/blockspawner set ...`) -> Server Command Parser -> **BlockSpawnerCommand** (Routing) -> BlockSpawnerSetCommand (Execution) -> World State Update

---

