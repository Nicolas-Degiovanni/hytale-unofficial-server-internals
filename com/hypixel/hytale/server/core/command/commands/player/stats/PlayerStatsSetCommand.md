---
description: Architectural reference for PlayerStatsSetCommand
---

# PlayerStatsSetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.stats
**Type:** Transient Command Object

## Definition
```java
// Signature
public class PlayerStatsSetCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The **PlayerStatsSetCommand** is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. It provides a user-facing entry point for administrators and game systems to modify a specific statistic on a target player entity.

Architecturally, this class serves as a specialized **Adapter** or **Facade**. It does not contain the core logic for stat modification. Instead, its primary responsibilities are:
1.  Defining the command's signature, including its name (`set`) and required arguments (`statName`, `statValue`).
2.  Leveraging the `AbstractTargetPlayerCommand` superclass to handle the complex logic of resolving a player entity from the command's input.
3.  Parsing and validating its specific arguments from the **CommandContext**.
4.  Delegating the final operation to the more generic **EntityStatsSetCommand** utility method.

This design decouples player-specific command handling from the underlying, more general entity stat system, promoting code reuse and a clear separation of concerns.

### Lifecycle & Ownership
-   **Creation:** A single instance is created by the server's command registration service during the server bootstrap sequence. The system scans for command classes and instantiates them to build the server's command tree.
-   **Scope:** The object instance is long-lived, persisting for the entire duration of the server session. It is held in memory by the central command registry.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively immutable after construction. Its internal fields, **entityStatNameArg** and **statValueArg**, are final references to argument definitions. The object holds no mutable state related to any specific execution; all necessary data is provided via the **CommandContext** parameter in the **execute** method.

-   **Thread Safety:** The instance itself is thread-safe due to its stateless nature. However, the **execute** method is designed to be invoked by a single thread from the server's main command processing loop.
    **Warning:** The methods it calls, particularly **EntityStatsSetCommand.setEntityStat**, perform modifications on the world state. These underlying systems are responsible for their own concurrency control. Invoking **execute** concurrently on different threads with contexts targeting the same player would create a race condition and is not supported.

## API Surface
The primary contract is the `execute` method, which is invoked by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerStatsSetCommand() | constructor | O(1) | Constructs the command, defining its name, description, and argument structure. |
| execute(...) | protected void | O(log N) | Parses arguments from the context and delegates to **EntityStatsSetCommand** to modify the target player's stat. Complexity depends on the **EntityStore** lookup, where N is the number of entities. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is invoked automatically by the server's command dispatcher in response to a corresponding in-game command.

For programmatic invocation (e.g., from a script or another system), one would use the server's command execution service, not call the method directly.

```java
// PSEUDO-CODE: How the system programmatically dispatches a command
// This is a conceptual example. Do not use this class directly.

CommandService commandService = server.getCommandService();
String commandToRun = "playerstats somePlayer set health 100";
CommandSource source = server.getConsoleSource();

commandService.executeCommand(source, commandToRun);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PlayerStatsSetCommand()`. The command system handles the lifecycle. Creating a new instance will result in a disconnected object that is not registered to handle any commands.
-   **Direct Invocation:** Never call the **execute** method directly. The framework is responsible for constructing the **CommandContext** and managing the execution environment. Bypassing the command service can lead to inconsistent world state, skipped permissions checks, and null pointer exceptions.
-   **Adding State:** Do not add mutable instance fields to this class. All state required for execution must be passed via the **CommandContext** to ensure the command is re-entrant and safe within the server's architecture.

## Data Pipeline
The flow of data for a typical command execution is managed by the server framework and passes through this component.

> Flow:
> Player Command Input (`/playerstats ...`) → Server Network Handler → Command Dispatcher → **PlayerStatsSetCommand**.execute() → Argument Parsing → **EntityStatsSetCommand**.setEntityStat() → World / EntityStore Update → (Optional) State Synchronization Packet to Client

