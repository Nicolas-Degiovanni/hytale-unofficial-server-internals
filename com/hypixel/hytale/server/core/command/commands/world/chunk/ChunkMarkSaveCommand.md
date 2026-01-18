---
description: Architectural reference for ChunkMarkSaveCommand
---

# ChunkMarkSaveCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient

## Definition
```java
// Signature
public class ChunkMarkSaveCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The ChunkMarkSaveCommand is a server-side administrative command that interfaces with the world persistence layer. It is a concrete implementation within the server's command system framework, inheriting from AbstractWorldCommand.

Its primary function is to explicitly flag a specific WorldChunk as "dirty", forcing the world storage system to write its current in-memory state to persistent storage during the next save cycle. This is a critical debugging and administrative tool for ensuring data integrity or forcing updates without waiting for natural game state changes.

The command's logic bifurcates into two distinct operational paths:
1.  **Synchronous Path (Chunk Loaded):** If the target chunk is already present in memory, the command directly accesses its component data via the ChunkStore and sets the "needs saving" flag. This operation is fast and executes immediately on the main server thread.
2.  **Asynchronous Path (Chunk Unloaded):** If the target chunk is not in memory, the command initiates an asynchronous load operation from disk via `world.getChunkAsync`. This prevents the main server thread from blocking on file I/O. Upon successful loading, a callback is scheduled via `world.execute` to run on the main world thread, where it then safely marks the newly loaded chunk for saving. This pattern is essential for maintaining server performance.

## Lifecycle & Ownership
- **Creation:** A single instance of ChunkMarkSaveCommand is instantiated by the command registration system during server bootstrap. It is registered under the name "marksave" within the world command hierarchy.
- **Scope:** The command object itself is a long-lived singleton that persists for the entire server session. However, each execution of the command is a transient, short-lived operation.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. Its only instance field, `chunkPosArg`, is an immutable definition of a command argument, configured in the constructor. The command does not store or cache any data between executions; all state it manipulates is external, residing within the World and WorldChunk components.

- **Thread Safety:** The `execute` method is designed to be called exclusively by the command system, which operates on the main server thread. The class achieves thread safety for I/O-bound operations by delegating to the asynchronous `world.getChunkAsync` method.

    **WARNING:** All state-mutating logic within the asynchronous callback (the `thenAccept` block) **must** be scheduled back onto the main world thread using `world.execute`. Direct modification of the WorldChunk from the I/O thread pool would create a severe race condition and lead to world corruption. The current implementation correctly follows this pattern.

## API Surface
The public contract is defined by the command framework. Direct invocation outside this framework is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | I/O Bound (Async) | Executes the command logic. Locates the target chunk, loading it from disk if necessary, and marks it for saving. Sends feedback messages to the command issuer. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use by developers. It is invoked by the server's command handler in response to a player or console command.

The user-facing interaction is through the server console or in-game chat:
```
/chunk marksave <x> <z>
```
*Example:*
```
/chunk marksave 150 -32
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChunkMarkSaveCommand()` in game logic. The command system manages its lifecycle. Manually calling `execute` is unsupported, as it requires a fully-formed CommandContext that is only available from the command handler.
- **Blocking on Chunk Load:** Modifying the asynchronous path to block the thread while waiting for a chunk to load is a critical performance anti-pattern. This would cause the main server thread to freeze, leading to severe lag or a server crash. Always use the provided asynchronous patterns.

## Data Pipeline
The flow of data for this command begins with user input and ends with a state change in a world data component.

> Flow:
> User Command String (`/chunk marksave 10 20`) -> Server Command Parser -> **ChunkMarkSaveCommand.execute()** -> World.getChunkStore() -> Check if chunk is loaded -> (Path A: Sync) -> WorldChunk.markNeedsSaving() -> CommandContext.sendMessage()
>
> (Path B: Async) -> World.getChunkAsync() -> I/O Thread Pool loads data -> Main Thread Callback (`world.execute`) -> WorldChunk.markNeedsSaving() -> CommandContext.sendMessage()

