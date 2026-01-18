---
description: Architectural reference for PlayerStatsResetCommand
---

# PlayerStatsResetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.stats
**Type:** Transient

## Definition
```java
// Signature
public class PlayerStatsResetCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerStatsResetCommand is a server-side command handler responsible for resetting a specific statistic on a target player entity. Architecturally, this class serves as a specialized adapter within the command system. It translates a player-centric command invocation into a more generic entity-based operation.

Its primary design function is to provide a user-facing command that operates on players, while delegating the core business logic to the more abstract EntityStatsResetCommand. This pattern promotes code reuse and separates the concerns of command parsing and player resolution from the underlying mechanics of stat manipulation. The parent class, AbstractTargetPlayerCommand, provides the foundational logic for identifying and targeting a specific player entity based on the command's input arguments.

## Lifecycle & Ownership
- **Creation:** A single instance of PlayerStatsResetCommand is instantiated by the server's command registration system during the initial server bootstrap phase. It is discovered through classpath scanning and added to a central command registry.
- **Scope:** The command object is a long-lived singleton that persists for the entire server session. It does not hold any per-execution state.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server is shutting down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** The class contains a single, immutable state field: entityStatNameArg. This field defines the metadata for the required "statName" command argument. This state is configured once in the constructor and never changes, making the command object itself effectively immutable post-construction.
- **Thread Safety:** This class is thread-safe. The command system invokes the execute method with a unique CommandContext for each execution, ensuring that there are no shared mutable state conflicts between concurrent command invocations. All logic is contained within the method scope or delegated to other systems, which are responsible for their own concurrency management.

## API Surface
The public contract is primarily defined by its constructor for instantiation by the framework and the protected execute method which implements the command logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerStatsResetCommand() | constructor | O(1) | Initializes the command definition and registers its required arguments. For framework use only. |
| execute(...) | protected void | O(log N) | Executes the stat reset logic. Complexity is dependent on the underlying EntityStore lookup. |

## Integration Patterns

### Standard Usage
This command is not intended to be invoked directly via code. It is designed to be executed by the server's command processor in response to a player or console input.

The typical invocation is via the in-game console:
> /playerstats *targetPlayerName* reset *statName*

For example, to reset the health of the player "Notch":
> /playerstats Notch reset health

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create an instance using **new PlayerStatsResetCommand()** and call its methods. Doing so bypasses the entire command processing pipeline, including argument parsing, permission checks, and context resolution. This will lead to NullPointerExceptions and an unstable server state.
- **Delegation Bypass:** Do not replicate the logic of this class to call EntityStatsResetCommand directly. The PlayerStatsResetCommand exists to correctly resolve player context and ensure the command is handled within the correct operational scope.

## Data Pipeline
The flow of data for a stat reset operation begins with user input and terminates with a modification in the world's entity storage.

> Flow:
> User Command String -> Command Parser -> **PlayerStatsResetCommand** -> EntityStatsResetCommand.resetEntityStat -> EntityStore Update -> World State Change

