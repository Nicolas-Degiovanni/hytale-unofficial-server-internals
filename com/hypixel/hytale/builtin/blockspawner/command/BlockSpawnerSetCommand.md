---
description: Architectural reference for BlockSpawnerSetCommand
---

# BlockSpawnerSetCommand

**Package:** com.hypixel.hytale.builtin.blockspawner.command
**Type:** Transient

## Definition
```java
// Signature
public class BlockSpawnerSetCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The BlockSpawnerSetCommand is a server-side command handler responsible for mutating the state of a BlockSpawner component within the game world. It serves as a direct, user-facing interface to the underlying Entity Component System (ECS), allowing administrators and players with sufficient permissions to configure block-based entity spawners.

This class embodies a standard pattern for world-mutating commands in the Hytale engine. It operates as a transactional unit that performs the following sequence:
1.  **Argument Parsing:** Defines and parses required and optional arguments from the raw command input string, such as the spawner asset ID and target position.
2.  **Target Resolution:** Determines the target block's world coordinates. It prioritizes an explicitly provided position argument. If omitted, it falls back to performing a raycast from the command sender's viewpoint to find the block they are targeting.
3.  **World State Access:** Safely retrieves the target WorldChunk from the World object.
4.  **Component Retrieval:** Accesses the block entity at the target coordinates and retrieves its attached BlockSpawner component instance from the ChunkStore.
5.  **State Mutation:** Modifies the BlockSpawner component's state by setting its spawner ID.
6.  **Persistence Flagging:** Marks the containing WorldChunk as dirty, signaling to the world persistence system that it needs to be saved to disk.

This command is a critical bridge between the high-level Command System and the low-level World and ECS storage layers.

### Lifecycle & Ownership
-   **Creation:** A single instance of BlockSpawnerSetCommand is instantiated by the server's CommandSystem during the server bootstrap and registration phase. It is discovered via reflection or an explicit registration list.
-   **Scope:** The object instance persists for the entire server session, acting as a stateless handler. The lifecycle of an *execution* of the command, however, is scoped to a single invocation of the execute method.
-   **Destruction:** The instance is dereferenced and garbage collected when the server shuts down and the CommandSystem is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless between executions. Its member fields, such as blockSpawnerIdArg and positionArg, are immutable definitions for the command argument parser. All state relevant to a specific execution is passed in via the CommandContext and method parameters.
-   **Thread Safety:** **This class is not thread-safe and must not be treated as such.** All command execution, especially methods that interact with the World or any Store, is strictly mandated to occur on the main server thread. Any attempt to invoke the execute method from an asynchronous task or different thread will lead to critical data corruption, race conditions, and server instability. The engine's core game loop ensures this contract is met for standard command invocations.

## API Surface
The public contract is fulfilled by its role as a command handler, primarily through the overridden execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(log n) or O(disk) | Executes the command logic. Complexity is dominated by chunk retrieval, which may trigger a disk read if the chunk is not in memory. Throws GeneralCommandException on validation or targeting failures. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly from Java code. It is invoked by players or the server console through the chat or command-line interface.

```sh
# Set the spawner at the player's target block to a zombie spawner
/blockspawner set hytale:zombie_spawner

# Set the spawner at a specific coordinate, ignoring asset validation checks
/blockspawner set mymod:custom_golem_spawner 100 64 -250 --ignoreChecks
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance using `new BlockSpawnerSetCommand()`. The command system manages the lifecycle. Manually created instances will not be registered or executable.
-   **Manual Execution:** Never call the `execute` method directly. This bypasses critical infrastructure, including argument parsing, permission checks, and context setup. All command execution must be routed through the central CommandSystem dispatcher.
-   **Asynchronous World Modification:** Do not wrap a call to this command in a separate thread or future. World state modification must be synchronized with the main server tick.

## Data Pipeline
The flow of data for a typical player-invoked command execution follows a clear path from network input to disk persistence.

> Flow:
> Player Input (`/blockspawner set...`) -> Client Network Packet -> Server Network Layer -> Command Parser -> **BlockSpawnerSetCommand.execute()** -> World API (getChunk) -> ChunkStore (getComponent) -> BlockSpawner Component State Mutation -> WorldChunk.markNeedsSaving() -> World Persistence System (Save to Disk)

