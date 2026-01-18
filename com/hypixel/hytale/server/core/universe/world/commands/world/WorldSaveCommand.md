---
description: Architectural reference for WorldSaveCommand
---

# WorldSaveCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldSaveCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts

The WorldSaveCommand class provides the server-side logic for the `/world save` command, acting as the primary user-facing entry point into the world persistence subsystem. It is not a data management class itself; rather, it is an orchestrator that translates a user command into a series of asynchronous I/O operations.

Its architecture is defined by its extension of AbstractAsyncCommand. This design choice is critical for server performance, ensuring that potentially long-running disk write operations do not block the main server thread, which is responsible for processing game ticks. The command achieves this by leveraging Java's CompletableFuture API to schedule and manage save tasks on background threads.

A key architectural pattern is the use of the World object itself as an Executor for the save tasks. By submitting the save logic via `CompletableFuture.runAsync(..., world)`, the command guarantees that all persistence operations for a specific world are executed serially on that world's dedicated task queue. This elegantly prevents data races and inconsistent state during the save process without requiring explicit locks. While multiple worlds can be saved concurrently, each individual world's save is an atomic, ordered operation.

The command delegates the low-level persistence logic to two specialized subsystems:
1.  **WorldConfigSaveSystem:** Responsible for serializing world-level metadata, configuration, and associated resources.
2.  **ChunkSavingSystems:** Responsible for the more intensive task of serializing all modified chunk data within a world.

## Lifecycle & Ownership

-   **Creation:** A single instance of WorldSaveCommand is created by the core CommandSystem during server initialization. The system discovers and registers all command classes at this time.
-   **Scope:** The singleton instance persists for the entire server session, held in the CommandSystem's registry. The execution of the command is a transient operation; no state is stored in the instance between invocations.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the CommandSystem is shut down as part of the server shutdown sequence.

## Internal State & Concurrency

-   **State:** WorldSaveCommand is stateless. Its fields, such as worldArg and saveAllFlag, are declarative definitions of the command's argument structure, not containers for execution-specific data. All state required for an invocation is passed within the CommandContext object.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless design. The primary `executeAsync` method is invoked by the CommandSystem, and it immediately offloads all I/O-bound work to background threads managed by the CompletableFuture framework and the per-world executors. This design makes it safe from concurrency issues at the command level.

## API Surface

The public contract is almost entirely defined by its parent, AbstractAsyncCommand. Direct invocation is not a standard use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | I/O Bound | Orchestrates the asynchronous save of a single world or all worlds. Returns a future that completes when all requested save operations are finished. |

## Integration Patterns

### Standard Usage

Developers do not typically instantiate or invoke this class directly. It is designed to be automatically discovered and executed by the server's command processing system. A server administrator or a player with sufficient permissions would trigger it via the server console or in-game chat.

A programmatic trigger from another server system would involve submitting the command string to the CommandSystem.

```java
// Conceptual example of a system triggering a global save
// Do not call executeAsync directly.

CommandSystem commandSystem = server.getService(CommandSystem.class);
CommandSource consoleSource = server.getConsoleSource();

// The CommandSystem handles parsing and invoking the correct command instance
commandSystem.execute("world save --all", consoleSource);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new WorldSaveCommand()`. An un-registered command instance is non-functional and will not be recognized by the server.
-   **Blocking on the Future:** Calling `.get()` or `.join()` on the CompletableFuture returned by `executeAsync` from a main game loop or network thread is a severe anti-pattern. This will block the thread and freeze the server, defeating the purpose of the asynchronous architecture.
-   **External State Modification:** While the per-world executor prevents data races from within its own task queue, modifying world data from an external thread while a save is in progress can lead to partially written or corrupted save files.

## Data Pipeline

The command initiates a data flow from the server's live memory representation of a world to its persistent state on disk.

> Flow:
> Console Input (`/world save ...`) -> Command Parser -> **WorldSaveCommand.executeAsync** -> Task Submission to World Executor -> WorldConfigSaveSystem / ChunkSavingSystems -> Read World & Chunk Data (Memory) -> Data Serialization -> Disk I/O (Files)

