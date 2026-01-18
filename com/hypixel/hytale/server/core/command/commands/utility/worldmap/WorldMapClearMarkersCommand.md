---
description: Architectural reference for WorldMapClearMarkersCommand
---

# WorldMapClearMarkersCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class WorldMapClearMarkersCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The WorldMapClearMarkersCommand is a server-side command object that implements a specific piece of game logic: clearing a player's custom map markers for their current world. As a subclass of AbstractPlayerCommand, it is designed to be executed exclusively by a player entity, not by the server console or other non-player actors.

This class operates within the server's Command System framework. The framework is responsible for parsing player input, identifying the command, and invoking its execution logic. Crucially, the command system uses a form of dependency injection to provide the execute method with all necessary contextual objects, including the entity component Store, a Ref to the player entity, and a reference to the World the player is in.

Its primary architectural role is to act as a transactional endpoint for a specific player data mutation. It directly accesses and modifies the PlayerWorldData component, which is a nested data structure within the primary Player component, demonstrating a common pattern of commands manipulating the Entity Component System (ECS).

### Lifecycle & Ownership
- **Creation:** A single instance of this command is created by the server's command registration system during server bootstrap. It is then registered under the alias "clearmarkers".
- **Scope:** The command object itself is a stateless singleton that persists for the entire server session. The execution context provided to the *execute* method is transient and scoped to a single command invocation.
- **Destruction:** The object is garbage collected when the server shuts down and its CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. Its only member field is a static final Message constant. All state required for its operation (the player, the world, the entity store) is passed as arguments to the execute method. This design ensures that a single instance can be safely reused for all players on the server.
- **Thread Safety:** The class itself is thread-safe due to its stateless nature. However, the execute method is not designed for concurrent invocation for the same player. The server's game loop and command system are expected to enforce a single-threaded execution model per-player or per-world, ensuring that modifications to player data via the Store and Ref are serialized and safe. Direct, multi-threaded invocation of execute would bypass these engine-level guarantees and is unsupported.

## API Surface
The public contract is defined by its constructor for registration and the overridden execute method for invocation by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldMapClearMarkersCommand() | constructor | O(1) | Constructs the command, setting its name and description key. |
| execute(context, store, ref, playerRef, world) | void | O(log N) | Executes the core logic. Retrieves and modifies player data. Complexity is dominated by the map lookup in getPerWorldData. |

## Integration Patterns

### Standard Usage
This command is not intended to be invoked directly from code. It is triggered by a player typing the command into the game client's chat console.

1.  A player types `/clearmarkers` and presses enter.
2.  The server's network layer receives the chat packet.
3.  The Command System parses the input, identifies "clearmarkers", and finds the registered WorldMapClearMarkersCommand instance.
4.  The system populates a CommandContext and invokes the command's execute method with the appropriate player and world references.

```java
// System-level invocation (conceptual)
// This code does NOT exist in the game logic, but illustrates the pattern.

String commandInput = "/clearmarkers";
PlayerRef executingPlayer = ...; // The player who sent the command

// The command system parses the input and finds the command object
AbstractCommand command = commandRegistry.findCommand("clearmarkers");

if (command instanceof WorldMapClearMarkersCommand) {
    // The system provides the full context for execution
    CommandContext ctx = create_context_for_player(executingPlayer);
    World world = executingPlayer.getWorld();
    Store<EntityStore> store = world.getEntityStore();
    Ref<EntityStore> ref = executingPlayer.getRef();

    command.execute(ctx, store, ref, executingPlayer, world);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldMapClearMarkersCommand()` in game logic. The command system manages the lifecycle. Instantiating it elsewhere serves no purpose as it cannot be invoked without the context provided by the system.
- **Manual Execution:** Never call the `execute` method directly. It relies on a fully-formed context that is only guaranteed to be correct when provided by the server's command processing pipeline. Manual invocation will likely lead to inconsistent state or runtime exceptions.

## Data Pipeline
The data flow for this command is initiated by player action and results in a targeted data mutation on the server.

> Flow:
> Player Chat Input -> Network Packet -> Server Command Parser -> **WorldMapClearMarkersCommand.execute()** -> PlayerWorldData Component Access -> Data Mutation (markers set to null) -> Feedback Message sent to Player

