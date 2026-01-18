---
description: Architectural reference for PlayerStatsGetCommand
---

# PlayerStatsGetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.stats
**Type:** Service

## Definition
```java
// Signature
public class PlayerStatsGetCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerStatsGetCommand class implements the server-side logic for the `/player stats get` command. It serves as a specialized entry point within the server's command processing framework.

Architecturally, this class is a **façade** or **adapter**. It does not contain any core stat-retrieval logic itself. Instead, it translates a player-centric command into a more generic entity-centric operation. It achieves this by inheriting from AbstractTargetPlayerCommand, which provides the foundational logic for resolving a command's target to a specific PlayerRef.

Its primary responsibility is to parse the required `statName` argument and then delegate the actual data lookup to the static utility method within EntityStatsGetCommand. This design promotes code reuse and separates the concerns of command argument parsing from the underlying game state manipulation. By delegating to a shared, generic entity-stat function, the system avoids duplicating logic for different entity types (e.g., players, mobs).

## Lifecycle & Ownership
-   **Creation:** A single instance of PlayerStatsGetCommand is instantiated by the server's command registration system during the server bootstrap sequence. The system discovers command classes and registers them to build the command tree.
-   **Scope:** The object is a singleton managed by the command registry. It persists for the entire lifecycle of the server process.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless with respect to command execution. It holds a final field, `entityStatNameArg`, which defines the command's argument structure. This field is initialized once in the constructor and is immutable thereafter. All data required for execution is passed via method parameters in the `execute` call.
-   **Thread Safety:** This class is thread-safe. The internal state is read-only after construction. The `execute` method is invoked by the server's command processing thread, and its operations are confined to the data within its method scope. It delegates to other systems which are expected to manage their own concurrency.

## API Surface
The primary contract is the `execute` method, which is called by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(1) | Executes the command logic. Parses the stat name and delegates to EntityStatsGetCommand. Throws exceptions if arguments are invalid. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically registered and executed by the server's command dispatcher in response to in-game or console commands. The framework handles the creation of the CommandContext and the invocation of the `execute` method.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PlayerStatsGetCommand()`. The command system is responsible for the lifecycle of all command objects. Manual instantiation will result in an object that is not registered with the dispatcher and will not function.
-   **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses critical framework features, including permission checks, argument parsing and validation, and context population, leading to unpredictable behavior and NullPointerExceptions.

## Data Pipeline
The flow of data for this command follows the standard server command processing pipeline.

> Flow:
> Command String (`/player ...`) -> Server Command Dispatcher -> Argument Parser -> **PlayerStatsGetCommand.execute** -> EntityStatsGetCommand.getEntityStat -> EntityStore Lookup -> CommandContext Response -> Client/Console Output

