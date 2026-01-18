---
description: Architectural reference for ChunkForceTickCommand
---

# ChunkForceTickCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient (Stateless Command Handler)

## Definition
```java
// Signature
public class ChunkForceTickCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The ChunkForceTickCommand is a server-side administrative command designed for debugging and direct world manipulation. It serves a single, highly-specific purpose: to force every block within a designated chunk to become active in the next game tick.

Architecturally, this class is a concrete implementation within the server's Command System. It extends AbstractWorldCommand, inheriting the necessary structure to be discovered, registered, and executed by the central command handler. Its primary role is to act as a bridge between a user-invoked command and the low-level world storage system.

Upon execution, it performs the following critical operations:
1.  Parses relative or absolute chunk coordinates from the command context.
2.  Resolves these coordinates into a canonical chunk index.
3.  Directly queries the active World's ChunkStore to locate the target chunk's data.
4.  If the chunk is loaded, it acquires a reference to the BlockChunk component.
5.  It then iterates through the entire 3D volume of the chunk (32x320x32 blocks) and sets the "is ticking" flag to true for each block.

This forces the game's ticking engine to process updates for all blocks in that chunk, which is useful for diagnosing issues related to block updates, random ticks, or chunk-level processing failures.

### Lifecycle & Ownership
-   **Creation:** A single instance of ChunkForceTickCommand is instantiated by the server's command registration service during the server bootstrap sequence. It is not created on a per-request basis.
-   **Scope:** The object is a long-lived singleton managed by the command registry. It persists for the entire duration of the server session.
-   **Destruction:** The instance is eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless. Its fields consist of immutable, static message templates and a final argument parser definition. All state required for an operation (the command issuer, the target world, entity stores) is passed into the execute method via the CommandContext parameter. This design ensures that a single instance can safely handle sequential command invocations without side effects.

-   **Thread Safety:** **This class is not thread-safe and is not designed to be.** The execute method performs direct, unsynchronized mutations on BlockChunk data. It is imperative that the command system invokes this method exclusively from the main server thread, which has exclusive write access to world state. Calling this from any other thread will lead to chunk corruption, race conditions, and server instability.

## API Surface
The public contract is defined by its constructor and the overridden execute method from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ChunkForceTickCommand() | constructor | O(1) | Instantiates the command and its argument definitions. Intended for framework use only. |
| execute(context, world, store) | void | O(N) | Executes the command logic. N is the fixed number of blocks in a chunk (327,680). Throws exceptions on invalid arguments. |

## Integration Patterns

### Standard Usage
This command is not intended to be invoked directly from application code. It is exclusively handled by the server's command processing system. The system parses user input, identifies this command, and dispatches the call.

```java
// Conceptual example of how the Command System invokes this command
// DO NOT REPLICATE THIS LOGIC.

CommandContext context = createFromUserInput("/chunk forcetick ~ ~");
World currentWorld = context.getWorld();
Store<EntityStore> entityStore = context.getEntityStore();

// The registry finds the singleton instance of the command
ChunkForceTickCommand command = commandRegistry.find("forcetick");

// The system invokes the execute method
command.execute(context, currentWorld, entityStore);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ChunkForceTickCommand()` in your code. The command is managed by the command registry. Manual instantiation bypasses the entire framework.
-   **Asynchronous Execution:** Never call the execute method from a separate thread, a future, or any asynchronous task. Modifying chunk state must be synchronized with the main server tick, and this command offers no protection against concurrent modification.
-   **Stateful Extension:** Do not extend this class to add state. The command system relies on its stateless nature.

## Data Pipeline
The data flow for this command is simple and direct, originating from a user and terminating in a direct world state mutation.

> Flow:
> User Input (`/chunk forcetick ...`) -> Network Layer -> Command Parser -> **ChunkForceTickCommand.execute()** -> ChunkStore -> BlockChunk Data Mutation -> Game Tick Processor

