---
description: Architectural reference for EntityStatsSetCommand
---

# EntityStatsSetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity.stats
**Type:** Singleton Command Handler

## Definition
```java
// Signature
public class EntityStatsSetCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityStatsSetCommand class is a server-side command handler responsible for modifying the numerical statistics of one or more entities. It serves as a direct interface between a user-initiated command and the underlying Entity Component System, specifically the EntityStatsModule.

Architecturally, this class is a concrete implementation within the server's command processing framework. By extending AbstractTargetEntityCommand, it delegates the complex logic of parsing entity selectors (e.g., @s, @e[type=player]) to its parent class. Its primary responsibility is to define the specific arguments it requires—a stat name and a value—and to implement the logic for applying this change to the targeted entities.

A key design aspect is its reliance on asset-defined data. The command does not contain a hardcoded list of valid statistics. Instead, it queries the EntityStatType asset map to resolve the provided stat name into a numerical index. This decouples the command's logic from game design data, allowing designers to add or change entity stats without requiring code modifications. If a stat name is not found, the command provides a "did you mean" suggestion by performing a fuzzy string search against the available stat names, enhancing its usability.

## Lifecycle & Ownership
- **Creation:** A single instance of EntityStatsSetCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is registered as the handler for the *set* subcommand within the *entity stats* command group.
- **Scope:** The object instance persists for the entire server session. It is a stateless singleton whose methods are invoked with context-specific data for each command execution.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the command registry is cleared during server shutdown.

## Internal State & Concurrency
- **State:** The class instance is effectively immutable after construction. It holds final references to its argument definitions (entityStatNameArg, statValueArg), but it stores no state related to any specific command execution. All necessary state, such as the command sender and target entities, is passed into the execute method via the CommandContext.
- **Thread Safety:** This class is not thread-safe and is designed to be operated exclusively by the main server thread. All modifications to entity components, including EntityStatMap, must be synchronized with the server's primary tick loop to prevent race conditions and data corruption.

**WARNING:** Invoking the execute method or the static setEntityStat method from any thread other than the main server thread will lead to undefined behavior and likely world state corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, entities, world, store) | void | O(N) | Overridden entry point called by the command system. N is the number of targeted entities. |
| setEntityStat(context, entities, value, name, store) | static void | O(N) | Core logic for applying a stat change to a list of entities. Throws no exceptions but sends failure messages to the context. |

## Integration Patterns

### Standard Usage
While this class is typically invoked by a user typing a command, its static helper method provides a clean, reusable entry point for other server-side systems to modify entity stats programmatically.

```java
// How a developer should modify an entity's stat from another system
// Assume 'myEntity' is a Ref<EntityStore> and 'world' is the current World

// 1. Get the entity stat module and component store
EntityStatsModule statsModule = EntityStatsModule.get();
Store<EntityStore> store = world.getComponentStore();

// 2. Prepare arguments
List<Ref<EntityStore>> targets = List.of(myEntity);
String statToChange = "hytale:health";
int newValue = 100;

// 3. Invoke the static utility method
// Note: A CommandContext is required for feedback messages.
EntityStatsSetCommand.setEntityStat(
    commandContext,
    targets,
    newValue,
    statToChange,
    store
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityStatsSetCommand()`. The command system manages the lifecycle of command handlers. If you need the functionality, use the static `setEntityStat` method.
- **Manual Execution:** Do not call the `execute` method directly. This bypasses critical steps performed by the command framework, such as argument parsing and entity selection, and will result in a failure state.
- **Off-Thread Modification:** Never call `setEntityStat` from an asynchronous task or a different thread. All entity state modifications must be scheduled to run on the main server thread.

## Data Pipeline
The flow of data for a typical command execution is as follows:

> Flow:
> User Command String (`/entity stats @s set hytale:health 100`) -> Server Command System Parser -> **EntityStatsSetCommand.execute** -> EntityStatType Asset Map (for name resolution) -> EntityStatsModule (to get component type) -> World Component Store (to get EntityStatMap) -> **EntityStatMap.setStatValue** -> CommandContext (for feedback message) -> User


