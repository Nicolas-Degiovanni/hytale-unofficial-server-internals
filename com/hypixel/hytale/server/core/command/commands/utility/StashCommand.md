---
description: Architectural reference for StashCommand
---

# StashCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility
**Type:** Transient

## Definition
```java
// Signature
public class StashCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The StashCommand class is a server-side controller that implements the in-game `/stash` command. It serves as a direct bridge between the Command System, which interprets player text input, and the World State Management system. Its primary architectural role is to provide a user-facing mechanism for interacting with the metadata of specific block entities, namely those implementing the ItemContainerState interface.

This command operates on the principle of "player-as-actor". It uses the executing player's context—specifically their line-of-sight—to target a block in the world. It then performs a series of lookups and validations to access and modify the `droplist` property on the target block's state. This isolates complex world data interactions, such as chunk lookups and block state manipulation, behind a simple, high-level command interface.

## Lifecycle & Ownership
- **Creation:** A single instance of StashCommand is created by the server's command registry during the server bootstrap phase. The system scans for classes extending command base types and instantiates them for registration.

- **Scope:** The StashCommand object is a singleton that persists for the entire lifetime of the server. However, its execution context is transient. Each invocation of the `/stash` command triggers a new call to the `execute` method on this shared instance, with a unique CommandContext for that specific execution.

- **Destruction:** The instance is destroyed when the server shuts down and the command registry is de-allocated. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The StashCommand class is effectively stateless with respect to its execution. Its only instance field, `setArg`, is an argument definition object that is initialized in the constructor and is immutable thereafter. All state required for execution (e.g., the player, the world, command arguments) is passed into the `execute` method.

- **Thread Safety:** The StashCommand instance itself is thread-safe due to its immutable internal state. However, the operations it performs within the `execute` method are **not** thread-safe. All interactions with the World, ChunkStore, and component states must be performed on the main server thread to prevent data corruption and race conditions. The command system framework guarantees this execution context.

**WARNING:** Calling the `execute` method or its private helpers from an asynchronous task or a separate thread will lead to severe world state corruption and server instability.

## API Surface
The public contract is defined by its base class, AbstractPlayerCommand. The primary entry point is the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(D) | Executes the command logic. Complexity is proportional to the raycast distance D. Throws exceptions on critical state errors. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation in developer code. It is automatically discovered and managed by the server's command system. A user or developer interacts with this component by typing the command into the game client's chat console.

**Get a container's droplist:**
```
/stash
```

**Set a container's droplist:**
```
/stash set my_custom_droplist
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new StashCommand()`. The command system manages the singleton instance. Direct instantiation will result in a non-functional object that is not registered to handle player input.

- **Manual Execution:** Do not call the `execute` method directly. This bypasses the entire command processing pipeline, including argument parsing, permission checks, and context validation, which can lead to unpredictable behavior and NullPointerExceptions.

- **Stateful Implementation:** Do not add mutable instance fields to this class to store data between executions. The command instance is shared across all players and all executions.

## Data Pipeline
The StashCommand acts as a controller in a data flow that begins with player input and ends with a world state change or a feedback message to the player.

> Flow:
> Player Input (`/stash ...`) -> Network Layer -> Server Command Parser -> **StashCommand.execute()** -> TargetUtil (Raycast) -> World & ChunkStore (Chunk/Block Lookup) -> ItemContainerState (Read/Write `droplist`) -> CommandContext (Send Feedback Message) -> Network Layer -> Client UI Update

