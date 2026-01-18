---
description: Architectural reference for WhoCommand
---

# WhoCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server
**Type:** Singleton

## Definition
```java
// Signature
public class WhoCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The WhoCommand class implements the server-side logic for the `/who` command, which lists all online players grouped by the world they inhabit. Architecturally, it serves as a specialized, stateless handler within the server's command processing system.

Its most critical design characteristic is its extension of AbstractAsyncCommand. This inheritance dictates that its primary logic must execute asynchronously, preventing the main server thread from blocking on potentially long-running operations like iterating through numerous players across multiple worlds.

The command operates by querying the top-level Universe singleton to get a collection of all active Worlds. For each World, it dispatches a separate asynchronous task. This task accesses the world's EntityStore to retrieve data for each connected player, specifically querying for the Player and DisplayNameComponent components from the Entity Component System (ECS). This fan-out/fan-in pattern, orchestrated via CompletableFuture, ensures high performance and responsiveness, even on servers with a large number of worlds and players.

## Lifecycle & Ownership
- **Creation:** A single instance of WhoCommand is created by the server's command registration service during the server bootstrap sequence. It is not instantiated on a per-request basis.
- **Scope:** The instance persists for the entire lifecycle of the server. As a stateless handler, this single instance is sufficient to serve all incoming `/who` command requests.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** WhoCommand is fundamentally stateless. It contains no mutable instance fields. The only member field is a static, immutable Message template used for formatting the output. All necessary state is acquired at runtime from the CommandContext or globally accessible systems like the Universe.
- **Thread Safety:** This class is thread-safe. The `executeAsync` method is designed to be invoked concurrently. The AbstractAsyncCommand framework guarantees that the core logic is executed on a worker thread pool. The command performs read-only operations on shared data structures (Universe, World, EntityStore), which are assumed to be designed for concurrent access. The use of CompletableFuture correctly orchestrates the asynchronous operations without introducing race conditions within the command itself.

## API Surface
The public contract is defined by its role as a command handler and is limited to the execution method inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(P) | Asynchronously queries all worlds to build a list of online players. P is the total number of players online across all worlds. The future completes when all worlds have been queried and all messages have been sent to the command source. |

## Integration Patterns

### Standard Usage
Direct invocation is rare. The class is primarily invoked by the server's command dispatcher when a player or console executes the command. Programmatic invocation, if necessary, should be done through the command system.

```java
// Example of programmatically dispatching the command
// Note: This is an advanced use case. Direct execution is an anti-pattern.
CommandManager commandManager = server.getCommandManager();
CommandSource source = getSomeSource(); // e.g., a player or console
commandManager.execute(source, "who");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WhoCommand()`. The command system manages the lifecycle of command instances. Creating your own instance will result in a disconnected object that is not registered to handle any commands.
- **Synchronous Blocking:** Do not call `.get()` or `.join()` on the returned CompletableFuture from the main server thread. This will block the server tick loop, causing catastrophic lag and defeating the purpose of the asynchronous design.
- **Direct Execution:** Avoid calling `executeAsync` directly. Bypassing the command manager means you also bypass critical infrastructure such as permission checks, argument parsing, and telemetry.

## Data Pipeline
The flow of data for a single `/who` command execution is a multi-world scatter-gather operation.

> Flow:
> Command Input -> CommandManager Dispatch -> **WhoCommand.executeAsync** -> Universe Query -> Parallel World Queries -> EntityStore Component Read -> Message Aggregation -> CommandContext.sendMessage -> Network Packet -> Client Chat


