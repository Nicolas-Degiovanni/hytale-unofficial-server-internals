---
description: Architectural reference for BlockRowCommand
---

# BlockRowCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block
**Type:** Singleton

## Definition
```java
// Signature
public class BlockRowCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The BlockRowCommand is a server-side, player-executable command handler responsible for a developer-centric world editing feature. It allows privileged users to spawn a line of blocks matching a wildcard text query.

Architecturally, this class serves as a high-level orchestrator, bridging several core engine systems:
- **Command System:** It registers itself with the server's command dispatcher, defining its name, description, and required arguments.
- **Entity Component System (ECS):** It queries the ECS to retrieve the position and orientation of the executing player entity. This data is essential for determining the origin and direction of the block row.
- **Asset System:** It performs a live query against the global **BlockTypeAssetMap**, a memory-resident map of all block definitions loaded from game assets. This allows for dynamic, data-driven block selection.
- **World & Chunk System:** It submits asynchronous requests to the **World** to place blocks. This interaction is non-blocking; the command does not wait for the blocks to be physically placed before completing.

The command's design prioritizes server performance by offloading the expensive I/O operation of chunk loading and modification to the world's background processing threads.

### Lifecycle & Ownership
- **Creation:** A single instance of **BlockRowCommand** is instantiated by the server's **CommandSystem** during the server bootstrap and command registration phase.
- **Scope:** The instance is a singleton that persists for the entire lifetime of the server. It does not hold state related to any specific execution, making it safe to be used across multiple command invocations.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server shuts down or the **CommandSystem** is reloaded.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its fields, such as **queryArg**, are configured once in the constructor and are not modified during execution. All necessary state (player position, world reference, command arguments) is passed into the **execute** method.
- **Thread Safety:** The **BlockRowCommand** instance itself is thread-safe due to its stateless nature. The **execute** method is invoked on the server's main logic thread. However, it initiates asynchronous operations by calling **world.getChunkAsync**. The subsequent block placement logic within the **thenAccept** lambda is executed on a world processing thread, as managed by the **World** system. The command itself does not manage any locks or concurrency primitives, correctly delegating this responsibility to the underlying chunk system.

## API Surface
The public contract is fulfilled by inheriting from **AbstractPlayerCommand**. The primary entry point is the overridden **execute** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N + M) | Executes the command logic. Complexity is O(N) for the block type search (where N is the total number of registered block types) plus O(M) for scheduling block placements (where M is the number of matched blocks). |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked by the server's command system when a player with appropriate permissions types the command into chat.

```
/row stone*
```
This command would find all block types whose names begin with "stone" and spawn them in a row in front of the player.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockRowCommand()`. The command framework is responsible for the lifecycle of command objects. Direct instantiation will result in an object that is not registered with the server and cannot be executed.
- **Manual Execution:** Avoid calling the **execute** method directly. Doing so bypasses the command system's argument parsing, permission validation, and context setup, which can lead to unpredictable behavior and server instability.
- **Assuming Synchronous Block Placement:** The use of **world.getChunkAsync** means block placement is not immediate. Any subsequent logic that depends on the blocks being present must be designed to handle this asynchronicity, likely by using the same asynchronous patterns.

## Data Pipeline
The flow of data and control for a single execution of this command follows a clear, orchestrated path through multiple server systems.

> Flow:
> Player Chat Input -> Server Network Layer -> Command Parser -> **BlockRowCommand.execute** -> ECS Query (Player Transform) -> Asset System Query (BlockTypeAssetMap) -> World System (world.getChunkAsync) -> Asynchronous Block Placement in Chunk Data

