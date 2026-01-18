---
description: Architectural reference for ChunkUnloadCommand
---

# ChunkUnloadCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient

## Definition
```java
// Signature
public class ChunkUnloadCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The ChunkUnloadCommand is a server-side administrative command responsible for forcefully removing a specific world chunk from active memory. It serves as a direct interface for server operators to interact with the world's in-memory state, primarily for debugging, performance management, or resolving corrupted chunk data without a full server restart.

As a subclass of AbstractWorldCommand, it integrates into the server's command processing system and is guaranteed to execute within the context of a valid World instance. Its core function is to translate a user-provided position into a chunk index, locate that chunk within the World's ChunkStore, and trigger its removal. This process involves not only de-referencing the chunk data but also notifying other systems, via the NotificationHandler, that the chunk's state has changed, which is critical for client-side updates.

This command directly manipulates the lowest levels of the world storage cache, making it a powerful but potentially disruptive tool.

### Lifecycle & Ownership
- **Creation:** A single instance of ChunkUnloadCommand is instantiated by the command registration system during server bootstrap. It is registered as a subcommand under the primary "chunk" command.
- **Scope:** The command object persists for the entire server session, acting as a stateless handler. The lifecycle of its operation is confined to the scope of the `execute` method call.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. Its only field, `chunkPosArg`, is a final definition for argument parsing, configured once during construction. All state required for execution, such as the target world and command issuer, is provided via the `CommandContext` argument in the `execute` method.
- **Thread Safety:** This class is not thread-safe. It is designed to be executed exclusively on the main server thread as part of the command processing pipeline. The underlying `ChunkStore` and `NotificationHandler` systems are expected to manage their own concurrency, but direct, multi-threaded invocation of the `execute` method will lead to race conditions and world corruption.

## API Surface
The public API is implicitly defined by its role as a command and is not intended for direct programmatic invocation. The primary entry point is the overridden `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Parses chunk coordinates from the context, locates the chunk in the world's ChunkStore, and requests its removal. Sends feedback messages to the command issuer. Complexity assumes hash map lookups in the ChunkStore. |

## Integration Patterns

### Standard Usage
This class is not invoked via code. It is triggered by a server administrator or a player with sufficient permissions executing the command in-game or via the server console.

```text
// Console or in-game chat command
/chunk unload <x> <z>
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChunkUnloadCommand()`. The command system manages its lifecycle. Creating a new instance serves no purpose as it will not be registered to handle any input.
- **Manual Execution:** Never call the `execute` method directly. Doing so bypasses the entire command system infrastructure, including argument parsing, permission checks, and context setup. This can lead to NullPointerExceptions and unpredictable server state.

## Data Pipeline
The data flow for this command is initiated by user input and results in a change to the server's in-memory world state.

> Flow:
> User Input (`/chunk unload 10 20`) -> Command Parser -> **ChunkUnloadCommand.execute()** -> World.getChunkStore().remove() -> World.getNotificationHandler().updateChunk() -> Network Subsystem (sends chunk unload packet to clients)

