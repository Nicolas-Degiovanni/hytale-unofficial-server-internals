---
description: Architectural reference for WaitCommand
---

# WaitCommand

**Package:** com.hypixel.hytale.builtin.commandmacro
**Type:** Transient Command Handler

## Definition
```java
// Signature
public class WaitCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The WaitCommand is a server-side command processor that introduces a non-blocking delay into a command execution sequence. It is a fundamental component of the command macro system, allowing for scripted events to be spaced out over time without halting the main server thread.

Architecturally, its most critical feature is its asynchronous nature, inherited from AbstractAsyncCommand. Instead of using a blocking call like Thread.sleep, which would freeze the server tick loop, WaitCommand leverages the Java Concurrency framework. It submits a task to a delayed executor service, freeing the command processor to continue handling other operations immediately. The command's execution is considered complete only when the returned CompletableFuture is resolved after the specified delay.

This design is essential for server performance and stability, ensuring that even long wait times within scripts do not impact the server's responsiveness to player actions or other system events.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's CommandRegistry during the bootstrap phase. The registry scans for command classes and creates a single, long-lived instance.
- **Scope:** The WaitCommand object itself is stateless and persists for the entire lifetime of the server process. It acts as a singleton-like handler for all `/wait` command invocations.
- **Destruction:** The object is dereferenced and eligible for garbage collection only during server shutdown when the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** The WaitCommand instance is effectively immutable after its constructor is run. Its fields, which define the command's arguments (timeArg, printArg), are final. All state related to a specific invocation, such as the duration of the wait or the command sender, is encapsulated within the CommandContext object passed to the execution method.
- **Thread Safety:** This class is inherently thread-safe. The instance holds no mutable state. The executeAsync method is designed to be called from the server's main command processing thread. The core logic is then offloaded to the thread-safe CompletableFuture.delayedExecutor, preventing any concurrency issues within the command itself.

## API Surface
The primary contract is fulfilled by overriding the executeAsync method from its parent class. Direct invocation of this method is discouraged; interaction should occur via the server's command dispatch system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture&lt;Void&gt; | O(1) | Schedules the wait task. Returns a future that completes after the specified delay. This method does not block. |

## Integration Patterns

### Standard Usage
This command is intended to be executed by a player, a server console, or a command block/scripting engine. The developer interacts with it via the server's central command dispatcher.

```java
// Server-side invocation via the command dispatcher
// This is the correct way to programmatically execute the command.
CommandDispatcher dispatcher = server.getCommandDispatcher();
CommandSender console = server.getConsoleSender();

// The dispatcher handles parsing, validation, and execution.
dispatcher.dispatch("wait 5.5 --print", console);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new WaitCommand()`. The command system relies on the single instance managed by the CommandRegistry for registration and lifecycle management.
- **Blocking on the Future:** The primary purpose of an async command is to avoid blocking. Calling `.get()` or `.join()` on the returned CompletableFuture from a critical server thread (like the main tick loop) will block that thread, defeating the purpose of the asynchronous design and likely causing server lag or crashes.

## Data Pipeline
WaitCommand does not transform data; it orchestrates a delay in a control flow. The flow for a typical invocation is as follows:

> Flow:
> Raw Text Command (`/wait 10`) -> Server Command Parser -> Argument Validation -> **WaitCommand.executeAsync** -> Java Delayed Executor -> Scheduled Task -> Optional Message to CommandSender

