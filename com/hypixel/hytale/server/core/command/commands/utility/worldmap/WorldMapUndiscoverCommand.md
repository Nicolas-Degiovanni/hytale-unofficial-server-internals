---
description: Architectural reference for WorldMapUndiscoverCommand
---

# WorldMapUndiscoverCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class WorldMapUndiscoverCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The WorldMapUndiscoverCommand is a server-side command handler responsible for processing player requests to remove zones from their discovered world map. It acts as a direct interface between a player's chat input and the mutation of their persistent map discovery state.

This class is a concrete implementation within the server's Command System framework. It does not contain business logic for map tracking itself; instead, it serves as a marshalling and validation layer. It retrieves the necessary context—such as the target player and the current world—from the command system, validates the provided arguments (the zone name), and then delegates the core state modification logic to the player's WorldMapTracker component.

Its primary architectural role is to decouple the command parsing and execution engine from the underlying player and world state management systems.

## Lifecycle & Ownership
- **Creation:** A single instance of WorldMapUndiscoverCommand is instantiated by the server's central CommandManager or a similar registry during the server bootstrap sequence. It is registered under the name "undiscover".
- **Scope:** The object is a stateless singleton for the command system. It persists for the entire lifetime of the server application. It is not tied to any specific player session or world instance; it is a reusable handler.
- **Destruction:** The instance is destroyed and garbage collected only when the server shuts down and the CommandManager is dismantled.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The instance field zoneArg is a final definition for argument parsing and does not represent mutable state. All state modifications performed by this command are on external objects, specifically the Player component associated with the command's context.
- **Thread Safety:** This class is thread-safe by design due to its stateless nature. The execute method is invoked by the server's main command processing thread, which operates within the server's primary tick loop. All interactions with the Entity Component System, via the Store and Ref parameters, are guaranteed to occur in a thread-safe context managed by the ECS framework, preventing race conditions on player data.

**WARNING:** While the class itself is safe, the components it interacts with, such as WorldMapTracker and Player, are assumed to be managed by the server's single-threaded game loop or an equivalent synchronization mechanism.

## API Surface
The public contract of this class is not its Java methods but the command it exposes to the game server. The protected execute method is the sole entry point for the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | protected void | O(N) | Executes the command logic. Complexity is O(N) where N is the number of defined zones, due to set lookups and fuzzy string matching. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly via code. It is invoked by a player or administrator through the in-game chat console.

*Example Player Interaction:*
1. Player types `/undiscover all` in the chat.
2. The server's command parser identifies the "undiscover" command and routes the request to the registered WorldMapUndiscoverCommand instance.
3. The execute method is called with the context of the player who sent the command.
4. The command logic accesses the player's WorldMapTracker and removes all discovered zones.
5. A confirmation message is sent back to the player.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldMapUndiscoverCommand()`. The command system handles instantiation and registration. A manually created instance will not be registered and cannot be invoked.
- **Direct Method Invocation:** Do not call the `execute` method directly. This bypasses the entire command system's infrastructure, including permission checks, argument parsing, and context provision. This will lead to NullPointerExceptions and unstable server state.

## Data Pipeline
This command initiates a data flow that modifies server-side player state. The client's world map is updated as a consequence of this state change, likely through a subsequent data synchronization packet.

> Flow:
> Player Chat Input -> Server Network Layer -> Command Parser -> **WorldMapUndiscoverCommand.execute()** -> Player.getWorldMapTracker() -> PlayerConfigData Mutation -> (Implicit) State Synchronization -> Client World Map Update

