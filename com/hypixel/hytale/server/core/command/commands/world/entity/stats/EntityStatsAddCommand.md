---
description: Architectural reference for EntityStatsAddCommand
---

# EntityStatsAddCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity.stats
**Type:** Singleton

## Definition
```java
// Signature
public class EntityStatsAddCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityStatsAddCommand class is a server-side command responsible for modifying a specific statistic on one or more game entities. It serves as a direct interface between a user-invoked command and the core Entity-Component-System (ECS) state of the game world.

This class is a concrete node within the server's command processing tree. It inherits from AbstractTargetEntityCommand, which delegates the complex and error-prone task of parsing entity selectors (e.g., @p, @e[type=zombie]) and resolving them into a list of valid entity references.

Its primary architectural function is to:
1.  Define the command's syntax, including its name (add) and required arguments (statName, statAmount).
2.  Parse and validate these arguments from a CommandContext provided by the command system.
3.  Translate a human-readable stat name (e.g., hytale:health) into a performance-optimized integer index via the EntityStatType asset map.
4.  Interact with the EntityStatsModule to retrieve and mutate the EntityStatMap component for each targeted entity.
5.  Provide user feedback regarding the success or failure of the operation.

The class also exposes its core logic via a public static method, addEntityStat, decoupling the state modification from the command execution context. This allows other server systems, such as scripting or quest engines, to grant stats programmatically without simulating a full command invocation.

### Lifecycle & Ownership
- **Creation:** A single instance of EntityStatsAddCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is not created on a per-execution basis.
- **Scope:** The object is a long-lived singleton. It persists for the entire server session, held within a central command registry.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class holds minimal internal state. Its member fields, entityStatNameArg and statAmountArg, are final references to argument definitions. These are configured once in the constructor and are immutable for the lifetime of the object. The class itself is stateless with respect to command execution; all state it modifies is external, located within the EntityStatMap components of world entities.

- **Thread Safety:** **This class is not thread-safe and must not be treated as such.** All command execution within the Hytale server is expected to occur on the main server thread (the primary game loop). Any calls to its methods, especially execute and addEntityStat, from an asynchronous task or a different thread will lead to severe concurrency issues, including ConcurrentModificationExceptions and corrupted game state.

## API Surface
The public contract is designed for two distinct use cases: invocation by the command system and direct programmatic use by other server modules.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, entities, world, store) | void | O(N) | Overridden entry point called by the command system. Iterates over N resolved entities and applies the stat modification. |
| addEntityStat(context, refs, amount, name, store) | void | O(N) | Static utility method containing the core logic. Decoupled from the command class instance for reuse by other systems. |

## Integration Patterns

### Standard Usage
The primary integration is via the static addEntityStat method, which allows other server-side logic to modify entity stats without needing to construct or dispatch a command.

```java
// Example: A quest reward script granting 10 strength to a player
// Assume 'playerEntityRef' and 'worldStore' are in scope

String statToModify = "hytale:strength";
int amountToAdd = 10;
List<Ref<EntityStore>> targets = List.of(playerEntityRef);

// The CommandContext can be null if no user feedback is required
EntityStatsAddCommand.addEntityStat(null, targets, amountToAdd, statToModify, worldStore);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityStatsAddCommand()`. The command system is responsible for the lifecycle of command objects. If you need the functionality, use the static `addEntityStat` method.
- **Multi-threaded Access:** Never call `execute` or `addEntityStat` from a worker thread. All world and entity modifications must be synchronized with the main server tick. Failure to do so will corrupt world state.
- **Stateful Logic:** Do not extend this class with the expectation of storing state between command executions. The same instance is used for all invocations of the "add" subcommand.

## Data Pipeline
The flow of data from user input to world state modification follows a clear, sequential path orchestrated by the command system.

> Flow:
> User Input (`/entity stats @s add hytale:health 10`) -> Command System Parser -> **EntityStatsAddCommand.execute** -> Argument Parsing (`statName`, `statAmount`) -> Entity Selector Resolution (`@s` -> Player Entity) -> **EntityStatsAddCommand.addEntityStat** -> EntityStatMap Component Lookup -> State Mutation (`addStatValue`) -> Feedback Message to User

