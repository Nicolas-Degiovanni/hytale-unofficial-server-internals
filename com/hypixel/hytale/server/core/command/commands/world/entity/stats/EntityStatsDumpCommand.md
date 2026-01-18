---
description: Architectural reference for EntityStatsDumpCommand
---

# EntityStatsDumpCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity.stats
**Type:** Transient

## Definition
```java
// Signature
public class EntityStatsDumpCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityStatsDumpCommand is a concrete implementation within the server-side Command System, designed for administrative and debugging purposes. It follows the Command Pattern, encapsulating the logic required to inspect and display the statistics of in-world entities.

Architecturally, this class serves as a leaf node in the command hierarchy. It extends the AbstractTargetEntityCommand base class, which abstracts away the complex logic of parsing entity selectors (e.g., `@p`, `@e[type=npc]`) and resolving them into a list of target entities. This allows EntityStatsDumpCommand to focus solely on its core responsibility: querying the Entity Component System (ECS) for statistics data and formatting it for output.

Its primary interaction is with the EntityStatsModule, from which it retrieves the specific ComponentType for an entity's EntityStatMap. This map is the data-source component containing all defined statistics for a given entity.

## Lifecycle & Ownership
- **Creation:** A single instance of EntityStatsDumpCommand is instantiated by the server's CommandRegistry during the server bootstrap phase. The system scans for classes annotated or structured as commands and registers them for invocation.
- **Scope:** The registered instance persists for the entire server session. It is effectively a singleton within the context of the CommandRegistry. The object itself is stateless; all execution-specific data is passed as arguments to its methods.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the CommandRegistry is cleared, typically during a server shutdown sequence.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields. All state required for execution, such as the command sender and target entities, is provided via the `context` and `entities` parameters of the `execute` method.
- **Thread Safety:** The class instance is inherently thread-safe due to its stateless nature. However, command execution is typically synchronized by the server's main thread or a world-specific scheduler. Direct, asynchronous invocation of its methods without external locking against the `Store<EntityStore>` is **not safe** and will lead to `ConcurrentModificationException` or other race conditions within the underlying ECS.

## API Surface
The primary entry point is the `execute` method, invoked by the command system. A static utility method is also provided to allow other systems to reuse the core dumping logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, entities, world, store) | protected void | O(N * M) | Overrides the base class to implement the command's logic. Iterates through N entities and their M stats. Invoked by the command system. |
| dumpEntityStatsData(context, entities, store) | public static void | O(N * M) | A static utility method that performs the core logic of retrieving, formatting, and sending entity stat data. Can be used independently of the command framework. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or invocation by developers. It is automatically registered and executed by the server's command handling system in response to player or console input.

The static `dumpEntityStatsData` method, however, can be leveraged by other server-side systems for custom debugging or logging.

```java
// Example of reusing the dump logic in a custom administrative module
Store<EntityStore> mainEntityStore = world.getStore(EntityStore.class);
List<Ref<EntityStore>> entitiesToDebug = findProblematicEntities(); // A custom entity lookup
CommandContext playerContext = getContextForPlayer(adminPlayer); // A context to send messages to

// Re-use the command's core logic without invoking the full command system
EntityStatsDumpCommand.dumpEntityStatsData(playerContext, entitiesToDebug, mainEntityStore);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityStatsDumpCommand()`. The command system manages the lifecycle of command objects. Creating your own instance serves no purpose.
- **Performance Blindness:** Executing this command against a large number of entities, such as with an `@e` selector in a populated world, will cause a noticeable performance spike. The operation iterates every stat on every matched entity, which can block the server thread.
- **Incorrect Context:** Using the static `dumpEntityStatsData` method with a `CommandContext` from a different source (e.g., a different player, a stale context) will result in messages being sent to the wrong destination or failing entirely.

## Data Pipeline
The flow of data for a typical command execution is linear, originating from user input and terminating at the user's client.

> Flow:
> User Input (`/entity stats @p dump`) -> Server Network Layer -> Command Parser -> **EntityStatsDumpCommand**.execute -> `Store.getComponent` -> `EntityStatMap` (Data Read) -> `MessageFormat` (Formatting) -> `CommandContext.sendMessage` -> Server Network Layer -> Client Chat UI

