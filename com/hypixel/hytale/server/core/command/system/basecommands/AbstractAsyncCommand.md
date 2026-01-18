---
description: Architectural reference for AbstractAsyncCommand
---

# AbstractAsyncCommand

**Package:** com.hypixel.hytale.server.core.command.system.basecommands
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class AbstractAsyncCommand extends AbstractCommand {
```

## Architecture & Concepts
The AbstractAsyncCommand class is the architectural foundation for all non-blocking, long-running server commands. Its primary design goal is to prevent expensive operations from stalling the main server thread, which is critical for maintaining server performance and responsiveness.

This class acts as a contract enforcer. By overriding the parent AbstractCommand's `execute` method and marking it as **final**, it forces all derivative commands to adopt an asynchronous execution model. The synchronous entry point is sealed, and developers are required to implement the `executeAsync` method, which returns a CompletableFuture. This design pattern makes it impossible to accidentally introduce a blocking command where an asynchronous one is expected.

Furthermore, it provides a standardized, exception-safe utility method, `runAsync`, for delegating work to a background thread pool. This helper abstracts away the boilerplate of try-catch blocks and exception logging, ensuring that all asynchronous commands have consistent and robust error handling.

## Lifecycle & Ownership
- **Creation:** Concrete implementations of AbstractAsyncCommand are not instantiated directly by gameplay logic. They are discovered and instantiated by the server's Command Registry during the bootstrap phase or when a game module is loaded.
- **Scope:** An instance of a command persists for the entire server session. It is a stateless service object registered once and reused for every invocation of that command.
- **Destruction:** The command instance is dereferenced and eligible for garbage collection when the Command Registry is cleared, typically during server shutdown or when a module is unloaded.

## Internal State & Concurrency
- **State:** This base class is stateless. It contains no mutable fields and its behavior does not change after construction. Configuration, such as the command name and description, is set in the constructor and is immutable thereafter. The static error message is also a constant.
- **Thread Safety:** This class is inherently thread-safe due to its stateless nature. Its core purpose is to facilitate safe concurrency for its subclasses. The `runAsync` helper method provides a safe context for executing logic on a background thread, with built-in exception handling that prevents crashes and ensures errors are logged correctly.

**WARNING:** While the base class is thread-safe, subclasses that introduce their own mutable state are responsible for ensuring their own thread safety.

## API Surface
The public contract is designed to guide developers into a safe, asynchronous pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(N) | **Abstract Contract.** Subclasses must implement this method to define the command's asynchronous logic. |
| runAsync(context, runnable, executor) | CompletableFuture<Void> | O(N) | **Utility.** A helper to safely execute a Runnable on a given Executor, with built-in exception handling and logging. |

## Integration Patterns

### Standard Usage
The standard pattern is to extend this class and implement the `executeAsync` method. For non-trivial logic, use the provided `runAsync` helper to delegate the work to a managed thread pool.

```java
// Example: A command to perform a slow world-saving operation
public class SaveWorldCommand extends AbstractAsyncCommand {

    private final WorldManager worldManager;
    private final Executor backgroundExecutor;

    public SaveWorldCommand(WorldManager worldManager, Executor backgroundExecutor) {
        super("save-world", "Saves the world state to disk asynchronously.");
        this.worldManager = worldManager;
        this.backgroundExecutor = backgroundExecutor;
    }

    @Override
    @Nonnull
    protected CompletableFuture<Void> executeAsync(@Nonnull CommandContext context) {
        context.sendMessage(Message.translation("server.world.save.started"));

        // Delegate the slow I/O operation to a background thread
        return runAsync(context, () -> {
            worldManager.saveAll();
            // This message is sent from the background thread, but sendMessage is thread-safe
            context.sendMessage(Message.translation("server.world.save.complete"));
        }, backgroundExecutor);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Blocking in executeAsync:** Performing blocking I/O, database calls, or long computations directly within `executeAsync` without using a CompletableFuture factory method (like `supplyAsync` or the provided `runAsync`) defeats the entire purpose of this class and will cause server lag.
- **Swallowing Exceptions:** Implementing `executeAsync` and wrapping the entire logic in a broad `try-catch (Exception e)` block that does not complete the future exceptionally. This will hide errors and prevent the system's default error handling from activating.
- **Direct Instantiation:** Never create an instance with `new MyAsyncCommand()`. Commands must be registered with the server's Command Registry to be discoverable and executable by players.

## Data Pipeline
The flow of data for an asynchronous command begins with user input and ends with an operation on a background thread.

> Flow:
> Player Input (`/save-world`) -> Network Packet -> Server Command Parser -> Command Registry Dispatch -> **AbstractAsyncCommand.execute** -> **Subclass.executeAsync** -> CompletableFuture -> Worker Thread Pool -> WorldManager.saveAll()

