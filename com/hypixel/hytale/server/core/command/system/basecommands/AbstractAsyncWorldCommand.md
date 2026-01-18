---
description: Architectural reference for AbstractAsyncWorldCommand
---

# AbstractAsyncWorldCommand

**Package:** com.hypixel.hytale.server.core.command.system.basecommands
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class AbstractAsyncWorldCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The AbstractAsyncWorldCommand is a foundational component within the server's command processing framework. It serves as a specialized base class for all commands that must execute within the context of a specific game world.

This class employs the **Template Method Pattern** to enforce a consistent, robust execution flow for world-specific operations. It defines the primary execution skeleton in its final `executeAsync` method, which encapsulates three critical steps:
1.  **Argument Definition:** It automatically injects an optional *world* argument into any command that inherits from it.
2.  **Context Resolution:** It parses the command context to find the target world.
3.  **Validation & Error Handling:** It performs a mandatory null-check on the resolved world, sending a standardized error message to the command source if no world is found.

By handling this boilerplate logic, it frees developers to focus exclusively on the command's core business logic. The actual work is deferred to subclasses, which must implement the abstract `executeAsync(context, world)` method. This design guarantees that any implementation receives a valid, non-null World object, significantly reducing errors and improving code clarity.

## Lifecycle & Ownership
- **Creation:** This abstract class is never instantiated directly. Concrete subclasses (e.g., a `SetTimeCommand`) are instantiated once by the server's command registration system during server bootstrap or plugin loading.
- **Scope:** An instance of a concrete command subclass persists for the entire server session, held as a singleton within the central command registry.
- **Destruction:** The command instance is dereferenced and becomes eligible for garbage collection when the server shuts down or its parent plugin is unloaded.

## Internal State & Concurrency
- **State:** This base class is stateless regarding execution. It holds an immutable `OptionalArg` field which defines the command argument's metadata. This state is configured at construction and never changes.
- **Thread Safety:** This class is inherently thread-safe. The `executeAsync` method is designed to be invoked by the command system's worker threads. The `CompletableFuture` return type is a critical part of its concurrent design, signaling that the actual command logic is performed asynchronously and will not block the main command processor.

**WARNING:** While the base class is thread-safe, implementers are responsible for ensuring that the logic within their `executeAsync(context, world)` override is also thread-safe. All mutations to world state must be scheduled on that specific world's scheduler.

## API Surface
The primary API surface is the abstract method that subclasses are required to implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context, world) | CompletableFuture<Void> | Implementation-dependent | Abstract method for command logic. Guaranteed to receive a non-null World. |

## Integration Patterns

### Standard Usage
The intended use is to extend this class to create a new command that operates on a world. The developer implements the abstract `executeAsync` method, which contains the core logic.

```java
// A concrete command that sets the time in a specific world.
public class SetTimeCommand extends AbstractAsyncWorldCommand {

    public SetTimeCommand() {
        super("settime", "Sets the time of day for a world.");
    }

    @Nonnull
    @Override
    protected CompletableFuture<Void> executeAsync(@Nonnull CommandContext context, @Nonnull World world) {
        // The 'world' parameter is guaranteed to be non-null here.
        // Schedule the state mutation on the world's dedicated thread.
        return world.getScheduler().runTask(() -> {
            world.setTime(6000); // Set time to midday.
            context.sendMessage(Message.translation("command.settime.success", world.getName()));
        });
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the World Parameter:** Writing logic inside the override that does not use the provided `world` object. If your command is not world-specific, extend `AbstractAsyncCommand` instead.
- **Blocking Operations:** Do not perform long-running or blocking I/O operations directly within the `executeAsync` method. This will stall the command system. Defer work to the world's scheduler or a separate thread pool and use the `CompletableFuture` to signal completion.
- **Direct Instantiation for Execution:** Never create an instance of a command class using `new` to execute it. Commands are singletons managed and executed exclusively by the server's command registry.

## Data Pipeline
The class acts as a crucial validation and dispatching gateway in the command execution pipeline. It ensures the necessary world context exists before forwarding control to the specific command's implementation.

> Flow:
> Player Chat Input -> Network Layer -> Command Parser -> **AbstractAsyncWorldCommand.executeAsync(context)** -> World Argument Resolution & Validation -> **Subclass.executeAsync(context, world)** -> World Scheduler -> Game State Mutation

