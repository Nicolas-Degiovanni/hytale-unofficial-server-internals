---
description: Architectural reference for EntityStatsGetCommand
---

# EntityStatsGetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity.stats
**Type:** Transient

## Definition
```java
// Signature
public class EntityStatsGetCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityStatsGetCommand class is a concrete implementation within the server's Command System framework. It provides a user-facing mechanism, typically via chat, for querying read-only state from the server's core Entity-Component-System (ECS).

By extending AbstractTargetEntityCommand, this class inherits the complex logic for parsing entity selectors (e.g., @p, @e[type=player]) and resolving them to a list of target entities. Its primary responsibility is to bridge the gap between a user command and the underlying EntityStatsModule.

Architecturally, it serves as a thin, stateless adapter. It receives a command invocation, parses its specific arguments (the stat name), and then queries the authoritative data source—the EntityStatMap component attached to each entity. It does not contain any game logic itself; it is purely a data retrieval and presentation layer for administrators and developers. The use of a static helper method, getEntityStat, promotes code reuse for other server systems that may need to perform the same lookup logic without invoking the full command parsing pipeline.

## Lifecycle & Ownership
- **Creation:** A single instance of EntityStatsGetCommand is instantiated by the server's CommandRegistry during the server bootstrap phase. The framework discovers this class and registers it as a handler for the *get* subcommand within the *entity stats* command group.
- **Scope:** The object instance is a long-lived singleton managed by the CommandRegistry, persisting for the entire server session. However, its methods are invoked on a per-command basis, and it holds no state between invocations.
- **Destruction:** The instance is de-referenced and eligible for garbage collection only when the server is shutting down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless and immutable after construction. Its only instance field, entityStatNameArg, is a final definition object configured in the constructor. The core execute method operates exclusively on parameters passed into it and does not modify any instance-level state.
- **Thread Safety:** The server's command system is designed to execute commands on the main server thread. As such, this class is not designed for concurrent execution and is not thread-safe. All interactions with the ECS via the Store parameter are assumed to be happening on the primary game loop thread.

**WARNING:** Do not invoke the execute method from an asynchronous task or worker thread. All entity and component interactions must be synchronized with the main server tick.

## API Surface
The primary contract is the execute method, invoked by the command framework. The static getEntityStat method provides a decoupled entry point for the core logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, entities, world, store) | void | O(N) | Framework entry point. Resolves arguments and delegates to getEntityStat. N is the number of targeted entities. |
| getEntityStat(context, entities, stat, store) | void | O(N) | Retrieves a stat from a list of entities and sends results to the command context. Throws no exceptions. |

## Integration Patterns

### Standard Usage
This class is invoked automatically by the server's command parser. For programmatic reuse of the underlying logic, other server systems should use the static getEntityStat method. This avoids the overhead and side-effects of the full command dispatch system.

```java
// Example: A custom game mode system needs to check a player's health stat.
// Note: This assumes execution on the main server thread.

public void checkPlayerHealth(CommandContext context, Ref<EntityStore> playerEntity, Store<EntityStore> store) {
    List<Ref<EntityStore>> targets = List.of(playerEntity);
    String healthStatName = "hytale:health"; // Use the canonical stat name

    // Re-use the command's logic without invoking the command itself
    EntityStatsGetCommand.getEntityStat(context, targets, healthStatName, store);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityStatsGetCommand()`. The instance is managed by the command framework. Creating a new one serves no purpose and will not be registered.
- **Directly Calling Execute:** Do not call the `execute` method from other parts of the codebase. This bypasses critical framework features like argument parsing, entity selection, and permission checking. Use the static `getEntityStat` method for shared logic.
- **String Manipulation:** Do not construct the stat name dynamically from untrusted user input. Always validate against the list of known stats provided by `EntityStatType.getAssetMap()` to prevent errors.

## Data Pipeline
The flow for a typical command invocation is a request-response cycle that traverses multiple server systems.

> Flow:
> Player Chat Input (`/entity stats @s get hytale:health`) -> Network Packet -> Command Parser -> **EntityStatsGetCommand.execute** -> EntityStatsModule -> EntityStore Component Lookup -> `EntityStatMap.get()` -> Message Formatter -> Network Layer -> Player Chat Output (`"hytale:health = 100.0"`)

