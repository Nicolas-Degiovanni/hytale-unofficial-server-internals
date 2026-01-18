---
description: Architectural reference for ChunkInfoCommand
---

# ChunkInfoCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Singleton

## Definition
```java
// Signature
public class ChunkInfoCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The ChunkInfoCommand is a server-side diagnostic tool that provides a low-level snapshot of a single world chunk's state. It functions as a bridge between a user-initiated command and the core world data structures.

As a subclass of AbstractWorldCommand, it is designed to operate within the server's command execution framework, ensuring it has safe access to the current World context. Its primary purpose is to query the ChunkStore, the in-memory cache and manager for all loaded chunk data. The command retrieves multiple component chunks associated with a specific chunk coordinate, such as WorldChunk, BlockChunk, and EntityChunk, and aggregates their state into a human-readable message.

This command is a **read-only** operation. It inspects the state of various chunk flags, block data, entity counts, and save statuses but never modifies world state. It is a critical tool for debugging issues related to chunk loading, generation, corruption, and performance.

### Lifecycle & Ownership
- **Creation:** A single instance of ChunkInfoCommand is created by the server's command registration system during the server bootstrap sequence. It is registered under the parent command "chunk" with the subcommand "info".
- **Scope:** The instance persists for the entire lifetime of the server. It is a stateless service object whose execute method is invoked on demand.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The field chunkPosArg is an argument definition object, configured once in the constructor and treated as immutable thereafter. All state that the command operates on is passed into the execute method, primarily through the World and CommandContext parameters.
- **Thread Safety:** **This class is not thread-safe and must only be used on the main server thread.** The execute method directly accesses the World and its associated ChunkStore without any locking mechanisms. These core data structures are not designed for concurrent access. The server's command system guarantees that all command execution happens synchronously on the main game loop thread, preventing race conditions with world ticking, chunk I/O, and physics.

## API Surface
The public API is defined by its role as a command. Direct invocation outside the command system is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Executes the command logic. Reads data for a single chunk from the ChunkStore, builds a detailed message, and sends it to the command source. Throws exceptions if arguments are invalid. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a server administrator or a player with appropriate permissions through the server console or in-game chat. The command system handles parsing, argument resolution, and invocation.

```
// User-typed command in the server console or chat
/chunk info 10 20
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChunkInfoCommand()`. The command system manages the lifecycle of command objects. Creating a new instance serves no purpose as it will not be registered to handle user input.
- **Manual Execution:** Do not call the `execute` method directly from other parts of the codebase. This bypasses the permission checks, argument parsing, and context setup provided by the command framework and can lead to unpredictable behavior or crashes if the required context is not correctly mocked.
- **Asynchronous Access:** Never invoke this command's logic from a background or worker thread. Direct access to the `World` or `ChunkStore` from any thread other than the main server thread will cause severe concurrency issues, including data corruption and server crashes.

## Data Pipeline
The command acts as a terminal point in a user-input-driven data-querying pipeline. It translates a high-level request into a low-level data report.

> Flow:
> User Input String (`/chunk info ...`) -> Command Dispatcher -> **ChunkInfoCommand** -> World.getChunkStore() -> Component Lookups (BlockChunk, EntityChunk, etc.) -> Message Builder -> CommandContext.sendMessage() -> Network Packet -> Client Chat UI

