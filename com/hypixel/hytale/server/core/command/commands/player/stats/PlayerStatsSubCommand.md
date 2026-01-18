---
description: Architectural reference for PlayerStatsSubCommand
---

# PlayerStatsSubCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.stats
**Type:** Transient

## Definition
```java
// Signature
public class PlayerStatsSubCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PlayerStatsSubCommand class is a structural component within the server's command system, implementing the *Composite* design pattern. It functions as a non-executable parent node, or a command router, designed exclusively to group related sub-commands under the primary *stats* command.

This class does not contain any logic for executing game actions or modifying player data. Its sole responsibility is to parse an incoming command string and delegate the execution to one of its registered child commands, such as PlayerStatsAddCommand or PlayerStatsGetCommand. This design promotes high cohesion and low coupling; each specific stat operation is encapsulated within its own dedicated class, making the command system significantly easier to maintain, test, and extend without modifying the parent container.

It serves as a foundational routing point for all administrative commands related to player statistics.

### Lifecycle & Ownership
- **Creation:** Instantiated once during the server's bootstrap sequence by a higher-level command registrar. It is part of the static definition of the server's command tree.
- **Scope:** Application-scoped. The single instance persists in the server's command registry for the entire runtime of the server. It is not created on a per-request or per-player basis.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server is shutting down or if the entire command registry is reloaded.

## Internal State & Concurrency
- **State:** Effectively **immutable** after construction. The internal list of sub-commands is populated within the constructor and is not designed to be modified at runtime. This class holds no mutable state related to players, sessions, or command execution.
- **Thread Safety:** This class is inherently **thread-safe**. Its immutable nature allows multiple command execution threads to safely traverse its structure to resolve the appropriate sub-command without contention or the need for synchronization primitives. The parent AbstractCommandCollection is designed for concurrent lookups.

## API Surface
The primary interaction with this class occurs during its construction and registration. It exposes no meaningful public API for direct invocation during gameplay. Its "API" is the command-line interface it defines for server administrators.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerStatsSubCommand() | void | O(1) | Constructor. Initializes the command group, setting its primary name, alias, and registering its fixed set of child commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by feature developers. It is registered declaratively with a parent command collection, which is then loaded by the server's central command registry. The system handles the routing and execution automatically.

```java
// Illustrative example of how the system registers this command group
// This logic typically resides in a higher-level registrar.

// A parent collection for all "/player" commands
PlayerCommandCollection playerCommands = new PlayerCommandCollection();

// Register the "stats" sub-command group
playerCommands.addSubCommand(new PlayerStatsSubCommand());

// The server's command registry can now dispatch "/player stats ..."
server.getCommandRegistry().register(playerCommands);
```

### Anti-Patterns (Do NOT do this)
- **Direct Execution:** Never attempt to call an `execute` method on an instance of PlayerStatsSubCommand. It is a non-executable container and will fail, as it has no action to perform itself.
- **Dynamic Modification:** Do not add or remove sub-commands from this collection after it has been registered. The command tree is considered static post-initialization. Modifying it at runtime can lead to unpredictable behavior and race conditions.
- **Manual Instantiation:** Do not instantiate this class within game logic. It is part of the server's static infrastructure and should only be created during the command registration phase.

## Data Pipeline
PlayerStatsSubCommand acts as a routing point in the command processing pipeline. It receives a partially parsed command and dispatches it to the appropriate handler for final execution.

> Flow:
> Raw Command String -> Command Parser -> Command Dispatcher -> **PlayerStatsSubCommand** (Routing) -> PlayerStatsSetCommand (Execution) -> Player Stat Service -> Database Update

