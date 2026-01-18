---
description: Architectural reference for ChunkLoadCommand
---

# ChunkLoadCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient

## Definition
```java
// Signature
public class ChunkLoadCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The ChunkLoadCommand class is a server-side command handler responsible for forcing a specific world chunk to be loaded from storage into active memory. It serves as a direct interface for administrators and automated systems to manipulate the live state of the world's chunk lifecycle, bypassing the natural player-proximity loading mechanism.

Architecturally, this command is a node in the server's Command System. It extends AbstractWorldCommand, which guarantees that the command operates within the context of a valid World instance, providing safe access to world-specific subsystems like the ChunkStore.

Its primary design feature is the use of asynchronous I/O. To prevent the main server thread from stalling while waiting for disk access, the command initiates a non-blocking load request via `world.getChunkAsync`. The subsequent logic, including marking the chunk for saving and notifying the user, is executed in a callback scheduled safely back onto the world's primary execution thread. This pattern is essential for maintaining server performance and responsiveness.

## Lifecycle & Ownership
- **Creation:** A single instance of ChunkLoadCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is not created on a per-request basis.
- **Scope:** The command object is a stateless singleton that persists for the entire lifetime of the server process. Its internal state consists only of immutable argument definitions.
- **Destruction:** The object is eligible for garbage collection only upon server shutdown when the central command registry is cleared.

## Internal State & Concurrency
- **State:** The ChunkLoadCommand object is effectively immutable after construction. Its fields, `chunkPosArg` and `markDirtyArg`, define the command's argument structure and are not modified during execution. All execution-specific state is passed via the CommandContext parameter.
- **Thread Safety:** This class is thread-safe by design. The `execute` method is invoked by the command system on a designated thread. The most critical operation, chunk loading, is delegated to the world's asynchronous task system. The completion callback is then marshaled back to the world's single-threaded executor via `world.execute`, preventing any race conditions when modifying world state, such as marking a chunk as needing to be saved.

## API Surface
The public contract is fulfilled by overriding the `execute` method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) + Async I/O | Executes the chunk loading logic. The method itself returns immediately, but it initiates a long-running, asynchronous I/O operation. Throws exceptions if arguments are malformed. |

## Integration Patterns

### Standard Usage
This command is not intended to be used programmatically by instantiating the class. It is designed to be invoked by the server's command parser, typically through a console or in-game chat message.

The standard invocation is via a command string:
```
# Loads the chunk at coordinates 150, -200
/chunk load 150 -200

# Loads the chunk at the executor's current position and marks it for saving
/chunk load ~ ~ --markdirty
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ChunkLoadCommand()`. The command system manages the lifecycle of this object. Manually created instances will not be registered and will be non-functional.
- **Synchronous Assumption:** Do not call this command and immediately assume the chunk is loaded. The operation is asynchronous. Any subsequent logic that depends on the chunk being present must be designed to handle this delay, for example by using the same asynchronous patterns.
- **Bypassing the Command System:** Do not invoke the `execute` method directly. This bypasses critical parsing, permission checks, and context setup performed by the command system, leading to unpredictable behavior and potential server instability.

## Data Pipeline
The flow of data and control for this command follows a clear, asynchronous path from user input to world state modification.

> Flow:
> Command String -> Server Command Parser -> **ChunkLoadCommand.execute** -> World.getChunkAsync -> I/O Thread Pool (Disk Read) -> World Executor Thread (Callback) -> WorldChunk.markNeedsSaving -> CommandContext.sendMessage (User Notification)

