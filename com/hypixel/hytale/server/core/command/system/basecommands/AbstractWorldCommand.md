---
description: Architectural reference for AbstractWorldCommand
---

# AbstractWorldCommand

**Package:** com.hypixel.hytale.server.core.command.system.basecommands
**Type:** Framework Component

## Definition
```java
// Signature
public abstract class AbstractWorldCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
AbstractWorldCommand is a foundational base class for creating server commands that operate within the context of a specific game **World**. It embodies the **Template Method** design pattern, providing a rigid, thread-safe execution skeleton while delegating the core command logic to its subclasses.

Its primary architectural role is to abstract away three complex and error-prone tasks:
1.  **Argument Parsing:** It automatically defines and parses an optional *world* argument for any command that extends it. This standardizes how users target a specific world.
2.  **Context Validation:** It robustly checks for the existence of the resolved world. If no world is found (e.g., the command is run from a context without a world and none is specified), it terminates execution gracefully with a standardized error message.
3.  **Thread Dispatching:** Most critically, it ensures that the command's business logic is executed on the correct world's dedicated thread. It retrieves the target World and passes it to the underlying asynchronous task scheduler, `runAsync`. This prevents race conditions and data corruption by serializing all operations against a given world's state.

Subclasses are freed from this boilerplate, allowing developers to focus exclusively on implementing the command's logic within the `execute` method, with the guarantee that they are operating in a safe, world-specific context.

## Lifecycle & Ownership
-   **Creation:** Concrete subclasses of AbstractWorldCommand are not instantiated directly for execution. They are instantiated once by the CommandSystem during server bootstrap or plugin loading and registered with the command dispatcher.
-   **Scope:** A command definition instance is a stateless singleton that persists for the entire server session.
-   **Destruction:** The instance is de-registered and becomes eligible for garbage collection when the server shuts down or the owning plugin is disabled.

## Internal State & Concurrency
-   **State:** This class is **stateless**. Its fields, such as `worldArg`, are immutable definitions configured at creation time. All state required for execution, such as the CommandContext and target World, is passed as method parameters during the `executeAsync` call.
-   **Thread Safety:** This class is inherently thread-safe. The `executeAsync` method can be safely invoked from any thread (typically the server's main network thread). Its core responsibility is to safely marshal the execution context to the single-threaded environment of the target World, where the subclass's `execute` method is then invoked.

    **WARNING:** While the framework is thread-safe, implementers of the `execute` method must assume they are in a single-threaded context and should **never** spawn new threads or perform long-running blocking operations, as this will stall the target world's tick loop.

## API Surface
The primary contract for developers is the abstract `execute` method they must implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | Varies | **(Abstract)** The core logic of the command. This method is guaranteed to be invoked on the target world's update thread. |

## Integration Patterns

### Standard Usage
A developer should extend this class to create any command that reads or mutates world-specific data, such as blocks, entities, or time. The framework handles the resolution of the world and ensures thread safety.

```java
// A concrete command to set the time in a specific world.
public class SetTimeCommand extends AbstractWorldCommand {

    // Arguments for the command would be defined here.
    // e.g., private final RequiredArg<Integer> timeArg = ...

    public SetTimeCommand() {
        super("settime", "Sets the time of day for a world.");
    }

    @Override
    protected void execute(@Nonnull CommandContext context, @Nonnull World world, @Nonnull Store<EntityStore> entityStore) {
        // This logic is now guaranteed to run on the correct world thread.
        // int time = this.timeArg.getProcessed(context);
        // world.getUniverse().getTimeState().setTimeOfDay(time);

        context.sendMessage(Message.translation("server.commands.time.success"));
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual World Resolution:** Do not attempt to parse or resolve the world manually within the `execute` method. The framework provides a validated and contextually correct World object as a parameter.
-   **Bypassing the Scheduler:** The `executeAsync` method is final and must not be circumvented. Bypassing it would execute your logic on the wrong thread, leading to severe concurrency issues and server instability.
-   **Storing State in the Command:** Do not add mutable instance fields to your command subclass. Commands must be stateless, as a single instance is shared and used for all executions.

## Data Pipeline
The flow of data for a command invocation is strictly controlled to ensure safety and correctness.

> Flow:
> User Command String -> Command Dispatcher -> **AbstractWorldCommand.executeAsync** -> World Argument Resolution -> World Thread Scheduler -> **Subclass.execute** -> World State Mutation or Query

