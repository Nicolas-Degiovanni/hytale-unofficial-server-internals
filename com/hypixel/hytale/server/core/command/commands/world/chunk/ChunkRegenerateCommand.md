---
description: Architectural reference for ChunkRegenerateCommand
---

# ChunkRegenerateCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient

## Definition
```java
// Signature
public class ChunkRegenerateCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The ChunkRegenerateCommand is a component of the server's Command System, designed to expose administrative world-manipulation functions to operators. As a subclass of AbstractWorldCommand, it is guaranteed to be executed within a valid World context, providing safe access to world-specific services like the ChunkStore.

Its primary architectural function is to act as a user-facing entry point for a complex, asynchronous backend process. It parses user-provided coordinates, translates them into a chunk index, and initiates a non-blocking chunk regeneration operation.

A key design feature is its use of `CompletableFuture` via `getChunkReferenceAsync`. This prevents the command execution thread from blocking on potentially slow disk or database I/O. The command dispatches the request and immediately returns, ensuring server responsiveness. The final confirmation message is scheduled for execution on the target world's main thread using `world.execute`, adhering to the engine's strict thread-ownership model for world modifications and interactions.

## Lifecycle & Ownership
- **Creation:** Instantiated once by the server's command registration service during server bootstrap. The system scans for command classes and adds an instance to the central command tree.
- **Scope:** Session-scoped. The single instance persists for the entire lifetime of the server process.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The class is effectively stateless from an execution perspective. It contains a single `final` field, `chunkPosArg`, which is an immutable configuration object defining the command's argument structure. All state required for execution is passed as parameters to the `execute` method.
- **Thread Safety:** This class is thread-safe. The `execute` method is invoked by the command system's worker thread. The implementation is carefully designed to handle concurrency by offloading the I/O-bound work to the ChunkStore's asynchronous systems. It then safely "hops" back to the world's dedicated thread for the final step, preventing any race conditions related to world state.

## API Surface
The public contract is fulfilled by overriding the `execute` method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(I/O) | Parses chunk coordinates from the context and initiates an asynchronous chunk regeneration process. The complexity is dominated by the underlying storage system's I/O. Sends a success message to the command source upon completion. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and managed by the server's command system. A user or system triggers its execution by issuing the corresponding text command.

The conceptual flow of invocation by the system is as follows:

```java
// System-level invocation (pseudo-code)
CommandManager commandManager = server.getCommandManager();
CommandContext context = createFromPlayerInput("/regenerate ~ ~");

// The manager finds and executes the registered command instance
commandManager.dispatch(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChunkRegenerateCommand()`. The command will not be registered with the server and will be non-functional.
- **Manual Execution:** Calling the `execute` method directly bypasses critical infrastructure, including permission checks, argument validation, and exception handling provided by the command system.
- **Blocking on the Future:** The underlying `getChunkReferenceAsync` method returns a `CompletableFuture`. An anti-pattern would be to modify this command to block on that future (e.g., using `.join()`). This would freeze the command executor thread, potentially stalling a portion of the server.

## Data Pipeline
The command orchestrates a flow of control and data across multiple threads and systems.

> Flow:
> User Command String -> Command System Parser -> **ChunkRegenerateCommand.execute()** -> Argument Parser (`chunkPosArg`) -> ChunkStore (`getChunkReferenceAsync`) -> I/O Worker Thread (Chunk Load/Regen) -> World's Main Thread (`world.execute`) -> CommandContext (`sendMessage`) -> Network Packet -> Client UI

