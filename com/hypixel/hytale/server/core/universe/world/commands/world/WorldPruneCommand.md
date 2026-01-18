---
description: Architectural reference for WorldPruneCommand
---

# WorldPruneCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world
**Type:** Transient Command

## Definition
```java
// Signature
public class WorldPruneCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The WorldPruneCommand is a server-side administrative command responsible for reclaiming system resources by unloading and optionally deleting inactive game worlds. It operates as a concrete implementation within the server's asynchronous command processing framework, inheriting from AbstractAsyncCommand.

This class directly interfaces with the **Universe** singleton, which serves as the authoritative container for all active game worlds. Its core logic identifies worlds that are eligible for pruning based on two criteria:
1.  The world is not the default, primary world.
2.  The world has a player count of zero.

A critical architectural aspect is its asynchronous nature. By extending AbstractAsyncCommand, the potentially long-running operation of iterating, unloading, and deleting world data from disk is offloaded from the main server thread. This prevents server stalls or "lag spikes" during a potentially intensive I/O operation, ensuring server responsiveness is maintained. The command modifies the state of target World objects by enabling the **deleteOnRemove** flag, signaling to the Universe that the world's files should be purged from disk upon unload.

## Lifecycle & Ownership
-   **Creation:** A single instance of WorldPruneCommand is instantiated by the server's command registration system during the initial bootstrap sequence. It is discovered and registered alongside all other server commands.
-   **Scope:** The command object persists for the entire server session, held as a reference within the central command dispatcher or registry. The execution logic within the executeAsync method, however, is ephemeral and tied to a specific command invocation.
-   **Destruction:** The object is dereferenced and eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless. It does not contain any mutable instance fields. All state, such as the set of worlds to be removed, is allocated as local variables within the scope of the executeAsync method. The static Message fields are immutable, compile-time constants.

-   **Thread Safety:** The class is inherently thread-safe. The primary execution logic is encapsulated within a CompletableFuture and submitted to a worker thread pool.

    **WARNING:** While the command itself is safe, it operates on the shared, mutable state of the Universe singleton. The correctness and safety of this command are critically dependent on the thread-safety guarantees provided by the Universe class, particularly for the getWorlds and removeWorld methods. Any race conditions would originate in the underlying Universe implementation, not this command class.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldPruneCommand() | constructor | O(1) | Instantiates the command. Defines its name, description, and permission requirements. |
| executeAsync(context) | CompletableFuture<Void> | O(N) | Asynchronously identifies and removes all empty, non-default worlds. N is the number of loaded worlds. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic invocation. It is designed to be executed by an authorized user (e.g., a server administrator) or an automated system via the server console.

```text
// Console or in-game chat invocation
/world prune
```

The command system receives this input, resolves it to the registered WorldPruneCommand instance, and invokes its executeAsync method, passing the appropriate CommandContext.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WorldPruneCommand()` in application logic. The command framework is responsible for the lifecycle of command objects. Manually creating an instance bypasses registration and is useless.
-   **Synchronous Blocking:** Do not call `executeAsync(ctx).join()` or `.get()` from the main server thread. This completely negates the benefit of the asynchronous design and will block the server tick loop, causing a catastrophic freeze.
-   **State Assumption:** Do not assume a world will be immediately available after being removed. The file deletion operation is asynchronous and may take time to complete. Subsequent logic should not depend on the immediate absence of world files on disk.

## Data Pipeline
The WorldPruneCommand initiates a control flow rather than a data processing pipeline. It is a terminal operation that results in system state changes and user feedback.

> Flow:
> CommandSender Input -> Command Parser -> **WorldPruneCommand.executeAsync** -> Universe.getWorlds() -> Universe.removeWorld(worldKey) -> Filesystem I/O -> CommandSender Feedback Message

