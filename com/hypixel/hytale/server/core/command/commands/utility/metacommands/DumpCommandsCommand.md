---
description: Architectural reference for DumpCommandsCommand
---

# DumpCommandsCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.metacommands
**Type:** Transient Command Object

## Definition
```java
// Signature
public class DumpCommandsCommand extends CommandBase {
```

## Architecture & Concepts
The DumpCommandsCommand is a server-side utility and introspection tool. It operates as a meta-command, meaning its function is to provide information about the command system itself rather than altering game state.

Its primary architectural role is to traverse the live command registry, managed by the **CommandManager**, and serialize a complete map of all registered commands, sub-commands, and their associated metadata (permissions, owning class, etc.) into a JSON file.

A key design feature is the decoupling of I/O operations from the main server thread. While the command data collection occurs synchronously to ensure a consistent snapshot of the registry, the expensive file writing process is offloaded to a background thread using a CompletableFuture. This prevents the server from stalling while writing to disk, which is critical for maintaining server performance.

## Lifecycle & Ownership
- **Creation:** A single instance of DumpCommandsCommand is created during server bootstrap. It is registered with the central **CommandManager** as part of the initial command setup phase.
- **Scope:** The object is session-scoped. It persists for the entire duration of the server's runtime, held as a reference within the **CommandManager**'s registration map.
- **Destruction:** The instance is eligible for garbage collection upon server shutdown when the **CommandManager** and its command registrations are cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. It contains no mutable instance fields and relies entirely on data provided by the **CommandManager** and the CommandContext at the time of execution.
- **Thread Safety:** The class is not designed to be thread-safe for concurrent executions, but the command system's single-threaded dispatch model makes this a non-issue. The `executeSync` method is invoked on the main server thread. It safely reads from the command registry and then hands off the file writing operation to the Java common ForkJoinPool via CompletableFuture.

    **WARNING:** The synchronous data gathering phase assumes that the command registry is not being modified by another thread. This is a safe assumption in the context of the server's main loop but would be a significant hazard if the command were ever executed outside of this managed environment.

## API Surface
The public contract is fulfilled by implementing the abstract methods of its parent, CommandBase. The primary entry point is `executeSync`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | protected void | O(N) + Async I/O | Primary execution logic. Traverses N registered commands and sub-commands. Initiates an asynchronous file write. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is registered with the command system and invoked by users (typically administrators) via the server console or in-game chat. The system dispatches the execution request to the registered instance.

```java
// The CommandManager is responsible for resolving and executing the command.
// A developer does not write this code; this is how the system uses the class.

// User input: "/dump"
CommandContext context = createFromUserInput(...);
AbstractCommand command = CommandManager.get().findCommand("dump");

// The framework handles permission checks and dispatches the call.
command.execute(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create instances using `new DumpCommandsCommand()`. The command must be registered with and managed by the **CommandManager** to function correctly within the permission and dispatch system.
- **External Invocation:** Do not call the `executeSync` method directly. Bypassing the command framework can lead to inconsistent state, permission bypasses, and unexpected errors, especially concerning the CommandContext.
- **Synchronous I/O:** Do not modify the class to perform file I/O on the calling thread. This would introduce server stalls (lag) and defeats a core design principle of the class.

## Data Pipeline
The flow of data for this command begins with a user action and ends with a file on disk and a feedback message.

> Flow:
> User Input (`/dump`) -> **CommandManager** (Dispatch) -> **DumpCommandsCommand** (Synchronous Data Collection) -> CompletableFuture (Asynchronous Serialization & File Write) -> Server File System (`dumps/commands.dump.json`) & CommandContext (Feedback Message)

