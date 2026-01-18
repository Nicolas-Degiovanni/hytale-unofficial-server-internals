---
description: Architectural reference for EntityStatsResetCommand
---

# EntityStatsResetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity.stats
**Type:** Transient

## Definition
```java
// Signature
public class EntityStatsResetCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityStatsResetCommand is a server-side, user-invokable command that modifies the state of game entities. It serves as a concrete implementation of the Command Pattern, integrating directly with the server's central command processing system. Its primary architectural function is to provide a controlled interface for administrators and game logic to revert a specific entity statistic to its default value.

This class extends AbstractTargetEntityCommand, which abstracts away the complex logic of parsing entity selectors (e.g., @p, @e[type=player]) and resolving them into a list of target entities. EntityStatsResetCommand is therefore a "leaf" command, focused exclusively on the business logic of stat manipulation.

The core logic is implemented in the static helper method resetEntityStat. This design choice isolates the state-mutating operation from the command's argument parsing and execution lifecycle, promoting potential reuse, although its primary consumer is the command itself. Interaction with the game world is performed through the Entity Component System (ECS) by retrieving and mutating the EntityStatMap component associated with each target entity.

## Lifecycle & Ownership
- **Creation:** A single prototype instance of EntityStatsResetCommand is instantiated by the CommandRegistry during server bootstrap. This instance is used to register the command's name, description, and argument structure. It is not created on a per-execution basis.
- **Scope:** The prototype instance is a singleton that persists for the entire server session, managed by the CommandRegistry. The execution of the command, however, is a transient operation tied to a short-lived CommandContext object.
- **Destruction:** The prototype instance is de-referenced and eligible for garbage collection when the CommandRegistry is cleared during server shutdown.

## Internal State & Concurrency
- **State:** This class holds minimal, immutable state post-construction. The `entityStatNameArg` field is a definition of a required command argument. The class itself is stateless with respect to its execution; all state it modifies is external, residing within the EntityStatMap components of the target entities via the master entity Store.

- **Thread Safety:** **This class is not thread-safe.** Like most game-world-mutating logic, its methods must be invoked exclusively from the main server thread. The command system guarantees that the `execute` method is called within the server's primary update loop. Any attempt to invoke its logic from an asynchronous task or a different thread will lead to severe concurrency issues, including data corruption and server instability.

## API Surface
The public API is primarily for internal use by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, entities, world, store) | void | O(N) | Overrides the parent method to orchestrate the stat reset operation. N is the number of target entities. |
| resetEntityStat(context, entities, entityStat, store) | void | O(N) | Static worker method containing the core logic for finding and resetting a stat on a list of entities. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation by developers. It is automatically discovered and executed by the server's command parser in response to player or console input. The standard interaction is through the game's command line interface.

**Example Player Input:**
```plaintext
/entity stats @s reset hytale:health
```

**Conceptual System Invocation:**
```java
// The CommandSystem receives input and dispatches it.
// This is a conceptual representation, not for direct use.
CommandSystem commandSystem = server.getCommandSystem();
String commandLine = "/entity stats @s reset hytale:health";

// The system parses, resolves the command, and invokes execute.
commandSystem.dispatch(consoleCommandSender, commandLine);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityStatsResetCommand()`. The command system manages the lifecycle of command objects. Manually creating an instance will result in an un-registered and non-functional command.
- **Asynchronous Execution:** Never call `execute` or `resetEntityStat` from a separate thread. All entity state mutations must be synchronized with the main server tick. Off-thread execution will corrupt the entity Store.
- **Manual Invocation:** Avoid calling the `execute` method directly. To programmatically run a command, use the server's `CommandSystem.dispatch` service. This ensures that the full context, including permissions and argument parsing, is correctly established before execution.

## Data Pipeline
The flow of data for this command begins with user input and ends with a state change in the Entity Component System.

> Flow:
> Player Command Input -> Server Network Layer -> Command Parser -> **EntityStatsResetCommand.execute()** -> EntityStatType Asset Lookup -> Entity Store Query -> **EntityStatMap Component Mutation** -> Success/Failure Message to Player
---

