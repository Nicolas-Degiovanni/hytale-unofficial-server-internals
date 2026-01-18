---
description: Architectural reference for CommandBase
---

# CommandBase

**Package:** com.hypixel.hytale.server.core.command.system.basecommands
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class CommandBase extends AbstractCommand {
```

## Architecture & Concepts
CommandBase is an abstract template class that serves as the foundation for all **synchronous** server-side commands. It is a critical component of the server's Command System, designed to simplify command implementation by abstracting away the complexities of asynchronous execution.

The parent class, AbstractCommand, utilizes a CompletableFuture to handle potentially long-running, asynchronous operations. However, the vast majority of server commands are simple, synchronous state modifications (e.g., changing a game rule, teleporting a player). CommandBase provides a specialized contract for this common use case.

It achieves this by providing a `final` implementation of the asynchronous `execute` method, which in turn calls a new `protected abstract` method: `executeSync`. This forces all subclasses to provide a simple, synchronous block of code, guaranteeing that their logic is executed within the server's main command processing tick. This design intentionally shields developers from managing futures and thread safety for routine command creation.

## Lifecycle & Ownership
- **Creation:** Subclasses of CommandBase are instantiated by the CommandRegistry during server initialization or when a plugin is loaded. The registry scans the classpath for command definitions and creates a single instance of each command to be registered. Direct instantiation by developers is an anti-pattern.
- **Scope:** Command instances are singletons within the scope of the CommandRegistry. They persist for the entire lifecycle of the server.
- **Destruction:** Instances are dereferenced and eligible for garbage collection only when the CommandRegistry is cleared, typically during a server shutdown sequence.

## Internal State & Concurrency
- **State:** The base class itself is stateless beyond its initial configuration (name, description). Its state is immutable after the constructor is called. Subclasses may introduce their own state, but this is heavily discouraged. Commands should be treated as stateless service objects that operate on the state of the world, not on their own internal state.
- **Thread Safety:** This class is **not thread-safe** and is not designed to be. The Command System guarantees that the `executeSync` method is always called from a specific, predictable thread (e.g., the main server thread). All command logic should be written with the assumption of single-threaded execution. Attempting to invoke a command instance from multiple threads will lead to race conditions and world corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | protected abstract void | Varies | The primary implementation point for subclasses. Contains the synchronous logic for the command. |
| execute(CommandContext) | protected final CompletableFuture | O(1) | **Sealed.** Implements the parent contract and delegates to `executeSync`. Returns null to signify immediate, synchronous completion. |

## Integration Patterns

### Standard Usage
The standard pattern is to extend CommandBase and implement the `executeSync` method. The class is then left for the server's automatic discovery and registration mechanism to handle.

```java
// How a developer should normally use this
public class CommandHeal extends CommandBase {

    public CommandHeal() {
        super("heal", "Restores a player's health.");
    }

    @Override
    protected void executeSync(@Nonnull CommandContext context) {
        Player target = context.getArgument("player");
        target.setHealth(target.getMaxHealth());
        context.getSource().sendMessage("Healed " + target.getName());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Asynchronous Logic:** Do not perform blocking I/O or long-running computations within `executeSync`. If an operation is asynchronous, you must extend AbstractCommand directly and use its CompletableFuture-based API.
- **Direct Invocation:** Never call `executeSync` or `execute` directly. The CommandDispatcher is responsible for managing the command lifecycle and context.
- **Mutable State:** Avoid storing session-specific or request-specific data as fields in your command class. This will cause severe concurrency issues, as a single command instance is shared across all server players.

## Data Pipeline
The class acts as a terminal execution point in the server's command processing pipeline.

> Flow:
> Player Chat Input -> Network Ingress -> Command Parser -> CommandDispatcher -> **CommandBase.execute()** -> **Subclass.executeSync()** -> World State Mutation -> Network Egress (Feedback Message)

