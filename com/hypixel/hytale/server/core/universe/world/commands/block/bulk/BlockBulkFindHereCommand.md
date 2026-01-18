---
description: Architectural reference for BlockBulkFindHereCommand
---

# BlockBulkFindHereCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block.bulk
**Type:** Managed Singleton

## Definition
```java
// Signature
public class BlockBulkFindHereCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The BlockBulkFindHereCommand is a server-side administrative command responsible for executing large-scale, asynchronous block searches within a specified radius of a player. Its primary architectural purpose is to provide a diagnostic tool for world inspection without compromising server performance.

It acts as a bridge between the Command System, the Entity Component System (for locating the player), and the World Storage System. The most critical design choice is the immediate offloading of the entire search operation to a background thread via CompletableFuture. This ensures that the main server thread, which handles the game loop and player interactions, is never blocked by a potentially long-running and I/O-heavy task.

The command iterates through world chunks in an outward spiral from the player's current location, loading each chunk and scanning its sections for the target block type. This spiral pattern is an optimization to find nearby results first.

**WARNING:** The implementation uses a synchronous-within-asynchronous pattern by calling `join()` on the chunk loading future inside the worker thread's loop. While this protects the main server thread, it makes the worker thread block on I/O for each chunk, which can be inefficient. A fully asynchronous pipeline would provide higher throughput.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the server's central Command System during the server bootstrap and registration phase. It is not instantiated on a per-player or per-execution basis.
- **Scope:** The object instance persists for the entire lifetime of the server process. It is a stateless handler that is reused for every execution of the `find-here` command.
- **Destruction:** The instance is eligible for garbage collection only upon server shutdown when the Command System is dismantled.

## Internal State & Concurrency
- **State:** The BlockBulkFindHereCommand instance is effectively immutable after construction. Its fields, which define the command's arguments, are configured once and never change. All state related to a specific execution (e.g., the count of found blocks, the iterator's position) is confined to local variables within the `execute` method and the lambda passed to the CompletableFuture.

- **Thread Safety:** The class is inherently thread-safe due to its stateless design. The `execute` method is designed to be invoked concurrently by the Command System. The core logic is executed on a background thread from the common ForkJoinPool.
    - The `found` counter is an AtomicInteger, ensuring safe increments from multiple threads, although the current implementation is single-threaded within the async task.
    - All interactions with the World and ChunkStore from the background thread are assumed to be thread-safe operations as designed by the storage layer.
    - The final message is sent to the player via PlayerRef, which is a thread-safe mechanism for communicating back to the main game thread.

## API Surface
The primary contract is not with other developer code but with the Command System that invokes it. The public-facing API is the command string available to players.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(radius²) | The main entry point called by the Command System. Parses arguments and launches the asynchronous block search task. Complexity is proportional to the number of chunks searched. |

## Integration Patterns

### Standard Usage
Developers should not interact with this class directly. The standard pattern is to invoke the command programmatically through the server's Command System, which handles parsing, context provision, and execution.

```java
// Example of dispatching the command for a specific player
// Assumes access to the server's CommandSystem and a valid PlayerRef

CommandSystem commandSystem = server.getCommandSystem();
PlayerRef targetPlayer = ...;

// The system will locate and invoke the correct command handler
commandSystem.dispatch(targetPlayer, "find-here hytale:stone --radius 10 --print");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockBulkFindHereCommand()`. The instance is useless without being registered in the Command System, which provides the necessary execution context.
- **Direct Invocation:** Never call the `execute` method directly. Doing so bypasses argument parsing, permission checks, and the context injection managed by the Command System, leading to NullPointerExceptions and undefined behavior.
- **Blocking the Main Thread:** Do not attempt to modify this command to make it synchronous or to block on the returned CompletableFuture from the main server thread. This would freeze the server.

## Data Pipeline
The command initiates a one-way data flow from the world storage to the player's chat, with the core processing occurring off the main thread.

> Flow:
> Player Command Input -> Command System Parser -> **BlockBulkFindHereCommand.execute** -> (Background Thread) -> SpiralIterator -> ChunkStore Request -> Chunk Data -> BlockSection Scan -> AtomicInteger Update -> PlayerRef.sendMessage -> Network Message -> Client Chat UI
---

