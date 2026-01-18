---
description: Architectural reference for BlockBulkReplaceCommand
---

# BlockBulkReplaceCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block.bulk
**Type:** Transient

## Definition
```java
// Signature
public class BlockBulkReplaceCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The BlockBulkReplaceCommand is a server-side administrative tool responsible for performing large-scale, in-world block replacement operations. It is designed as a player-executable command that operates within a defined radius of the player's current position.

Architecturally, its most critical feature is its asynchronous execution model. To prevent catastrophic performance degradation of the main server thread, the entire block-finding and replacement logic is delegated to a background thread pool using CompletableFuture. This ensures that even a massive replacement operation does not cause server lag or unresponsiveness.

The command's search pattern is a SpiralIterator, which efficiently expands outwards from the player's origin chunk. This is an optimal approach for radius-based searches in a grid-based world system.

A key aspect of its logic is the handling of block variants. The system does not perform a simple integer ID swap. It correctly resolves block types with rotational variants (e.g., stairs, logs) by querying the BlockType asset map. It builds a list of all rotational variants for both the target and replacement blocks, ensuring that a north-facing stair is replaced by a north-facing stair of the new type, preserving world structure and orientation.

## Lifecycle & Ownership
- **Creation:** A single instance of BlockBulkReplaceCommand is instantiated and registered by the server's command system during the server bootstrap sequence.
- **Scope:** The command object itself is a long-lived singleton that persists for the entire server session. However, each execution of the command spawns a short-lived, asynchronous task. This task exists only for the duration of the replacement operation.
- **Destruction:** The singleton instance is discarded during server shutdown. The asynchronous tasks are eligible for garbage collection upon completion or if the server shuts down prematurely.

## Internal State & Concurrency
- **State:** The BlockBulkReplaceCommand instance itself is effectively stateless. Its fields are final argument definitions. All state related to a specific operation (e.g., radius, block types, counters) is managed on the stack within the scope of the execute method and its spawned asynchronous task.

- **Thread Safety:** The class is not designed to be thread-safe if the execute method were invoked concurrently on the same instance. However, the command system's single-threaded dispatch model prevents this. The core logic is intentionally moved off the main server thread.

    **WARNING:** The internal concurrency model is complex.
    1.  The primary loop over the SpiralIterator occurs within a single background thread.
    2.  Inside the loop, `chunkComponentStore.getChunkReferenceAsync(...).join()` is a **blocking call** within the background thread. This serializes the processing of each chunk, preventing an uncontrolled flood of chunk loading operations.
    3.  The final block setting (`chunk.setBlock`) is dispatched via `thenAccept`, allowing multiple block updates to be in-flight concurrently.
    4.  The `replaced` counter is an AtomicInteger to safely handle concurrent increments from these final `thenAccept` callbacks, which may be executed by different threads in the common fork-join pool.

## API Surface
The public contract is fulfilled by inheriting from AbstractPlayerCommand. The primary entry point is the `execute` method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(R² * C) | Invoked by the command system. Triggers the asynchronous block replacement. R is the radius in chunks; C is the cost of processing each chunk. Throws exceptions on invalid arguments. |

## Integration Patterns

### Standard Usage
This command is intended to be used by a privileged user (e.g., an administrator) via the in-game chat console. The command system parses the input, validates arguments, and dispatches the call.

```java
// This logic is handled by the server's command dispatcher.
// A player in-game would type:
// /replace stone dirt 10

// The conceptual flow inside the engine:
CommandContext context = commandParser.parse("/replace stone dirt 10");
BlockBulkReplaceCommand command = commandRegistry.get("replace");
command.execute(context, ...);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockBulkReplaceCommand()`. The command must be retrieved from the server's command registry to ensure it is properly initialized and integrated.
- **Excessive Radius:** Invoking this command with a very large radius can still cause significant server strain. While asynchronous, it will aggressively load chunks into memory and consume considerable CPU resources for scanning and modification. This can lead to high memory usage and potential out-of-memory errors.
- **Ignoring Asynchronicity:** Do not write subsequent code that assumes the replacement is complete immediately after the command is executed. The operation runs in the background, and the only notification of completion is the chat message sent to the originating player.

## Data Pipeline
The data flow for a single command execution begins with player input and terminates with world modification and a feedback message.

> Flow:
> Player Chat Input -> Server Network Parser -> Command Dispatcher -> **BlockBulkReplaceCommand.execute** -> CompletableFuture Task -> SpiralIterator -> ChunkStore (Async Load) -> BlockSection Scan -> World.setBlock (Async Update) -> PlayerRef.sendMessage

