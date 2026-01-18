---
description: Architectural reference for BlockBulkFindCommand
---

# BlockBulkFindCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block.bulk
**Type:** Transient

## Definition
```java
// Signature
public class BlockBulkFindCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The BlockBulkFindCommand is a server-side administrative command responsible for locating a specified number of a particular block type within the game world. It is a diagnostic and moderation tool, not a component of core gameplay logic.

Architecturally, this command is designed as a self-contained, asynchronous task executor. It plugs into the server's central Command System, which manages its instantiation and invocation. Its most critical design feature is the offloading of the search operation from the main server thread to a background worker thread using a CompletableFuture. This prevents the potentially long-running and I/O-heavy task of loading and scanning chunks from stalling the server's primary game loop, which would otherwise result in catastrophic performance degradation.

The search algorithm employs a SpiralIterator, an efficient pattern for exploring the 2D chunk grid. This ensures that the search expands outwards from a central point, finding the nearest instances of the target block first. This is generally the most desirable behavior for a "find" command.

## Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by the server's Command System when a user or automated process executes the corresponding command string (e.g., `/find`). It is not pre-allocated or pooled.
-   **Scope:** The object's lifetime is extremely short, scoped exclusively to a single command execution. Once the `execute` method is called and the asynchronous task is dispatched, the BlockBulkFindCommand object itself has fulfilled its purpose and is eligible for garbage collection.
-   **Destruction:** The object is garbage collected shortly after the `execute` method returns. The background task it spawns will continue to live until it completes, times out, or is cancelled, but it holds no reference back to the original command object.

## Internal State & Concurrency
-   **State:** The class instance holds immutable configuration for its command arguments. All mutable state required for the search operation (e.g., counters, temporary collections) is created and confined within the scope of the lambda expression passed to `CompletableFuture.runAsync`. This state is therefore local to the background thread executing the search.
-   **Thread Safety:** This class is not thread-safe and is not intended to be. It is designed to be instantiated and used on the main server thread, which then safely dispatches a task to a worker thread.

    **WARNING:** The core search logic executes on a background thread but interacts with components that may be accessed by the main server thread. Specifically, it calls `CommandSender.sendMessage` and accesses the `World` and its `ChunkStore`. These downstream systems **must** be thread-safe to prevent race conditions, data corruption, or concurrency exceptions. The use of `getChunkReferenceAsync` indicates the `ChunkStore` is designed for such asynchronous, multi-threaded access.

## API Surface
The primary contract is the `execute` method, which is the entry point for the Command System.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N) | Dispatches an asynchronous, long-running task to search for blocks. N is the number of chunks searched, bounded by the timeout and count arguments. This method itself returns immediately with O(1) complexity. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use by developers. It is automatically discovered and managed by the server's Command System. A user, such as a server administrator, invokes it via the server console or in-game chat.

The following example illustrates how the *Command System* would internally invoke this command after parsing user input.

```java
// Pseudo-code for Command System invocation
CommandContext context = commandParser.parse(userInput, sender);
BlockBulkFindCommand command = new BlockBulkFindCommand(); // System instantiates the command
World world = server.getWorld();
Store<EntityStore> store = world.getEntityStore();

// The system calls execute, which returns immediately
command.execute(context, world, store);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually create instances of this class using `new BlockBulkFindCommand()`. The command framework handles the lifecycle.
-   **Synchronous Execution:** The search logic within the `CompletableFuture` is computationally expensive and involves I/O. Never extract this logic and run it on the main server thread, as it will freeze the server.
-   **Modifying Shared State:** Do not modify the command to alter shared game state (e.g., `World` or `EntityStore` components) from the background thread without proper synchronization mechanisms. The current implementation correctly limits itself to read-only operations and thread-safe message sending.

## Data Pipeline
The flow of data for a single execution is a one-way path from user input to user feedback, with a significant detour through a background processing thread.

> Flow:
> User Command String -> Command System Parser -> **BlockBulkFindCommand.execute** -> CompletableFuture (Worker Thread) -> ChunkStore (I/O) -> Block/Section Data Scan -> CommandSender.sendMessage -> User Feedback

