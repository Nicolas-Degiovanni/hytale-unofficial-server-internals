---
description: Architectural reference for ChunkResendCommand
---

# ChunkResendCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Singleton

## Definition
```java
// Signature
public class ChunkResendCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The ChunkResendCommand is a server-side administrative command designed for debugging and resolving world streaming issues for a specific player. It functions as a concrete implementation of the Command Pattern, registered within the server's central command processing system.

Its primary architectural role is to serve as a direct, high-level interface to the server's chunk management subsystem. By invoking this command, an administrator can force a hard reset of a player's loaded world data. This is achieved by manipulating the state of the target player's **ChunkTracker** component, which is the canonical server-side representation of the chunks a client has received and loaded.

The command has two modes of operation:
1.  **Standard Resend:** Instructs the player's ChunkTracker to unload all currently tracked chunks. This triggers the client to re-request the chunks it needs from the server, effectively refreshing the player's view of the world.
2.  **Cache Clearing Resend:** When the *clearcache* flag is provided, the command first invalidates the server's own cached data for each chunk in the player's view radius before triggering the unload. This is a more destructive operation intended to resolve issues of server-side chunk corruption or desynchronization.

This class directly interacts with the server's Entity Component System (ECS) stores, reading the player's position from a TransformComponent and modifying the ChunkTracker component.

### Lifecycle & Ownership
- **Creation:** A single instance of ChunkResendCommand is created by the CommandSystem during server initialization when all commands are discovered and registered.
- **Scope:** The object is a singleton that persists for the entire server session. It does not hold any per-execution state.
- **Destruction:** The instance is de-referenced and eligible for garbage collection only when the server shuts down and the CommandSystem is dismantled.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The only instance field, clearCacheArg, is an immutable definition of a command-line flag initialized in the constructor. All stateful operations performed by the execute method are on external objects passed in as parameters, such as the World and ECS stores.

- **Thread Safety:** **This class is not thread-safe and must not be treated as such.** The execute method is designed to be invoked exclusively by the main server thread as part of the game's tick cycle. It performs direct, unlocked reads and writes on critical game state components like ChunkStore and ChunkTracker. Any concurrent access from another thread would result in race conditions, data corruption, and likely server crashes.

## API Surface
The public contract is defined by its role as a command. The primary entry point is the execute method, which is called by the CommandSystem.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, ...) | void | O(R²) | Executes the command logic. Complexity is O(1) for a standard resend. With the clearcache flag, complexity becomes O(R²) where R is the player's view radius, due to the spiral iteration over all chunks in that radius. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use by other systems. It is designed to be invoked exclusively by the server's CommandSystem in response to user input from the console or in-game chat.

**Example User Invocation:**
```
/chunk resend PlayerName
```

**Example User Invocation with Cache Clearing:**
```
/chunk resend PlayerName --clearcache
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChunkResendCommand()`. The command will have no effect unless it is properly registered with the server's CommandSystem, which handles its lifecycle.
- **Asynchronous Execution:** Never invoke the `execute` method from a separate thread or asynchronous task. This will bypass the server's single-threaded game loop guarantees and corrupt world state.
- **Stateful Modification:** Do not modify this class to hold state between executions. Command objects are expected to be stateless singletons.

## Data Pipeline
The command initiates a complex data and state flow that crosses the client-server boundary. The primary purpose is to force a state reconciliation for world data.

> Flow:
> User Input (`/chunk resend ...`) -> Command Parser -> **ChunkResendCommand.execute()** -> [Optional] World.ChunkStore Cache Invalidation -> Player.ChunkTracker.unloadAll() -> Server Network Layer (sends unload packets) -> Client Network Layer (receives unload packets) -> Client World (unloads chunks) -> Client Chunk Request Logic (requests missing chunks) -> Server (sends fresh chunk data)

