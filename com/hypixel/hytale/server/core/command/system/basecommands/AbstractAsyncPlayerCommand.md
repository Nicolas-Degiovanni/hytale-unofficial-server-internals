---
description: Architectural reference for AbstractAsyncPlayerCommand
---

# AbstractAsyncPlayerCommand

**Package:** com.hypixel.hytale.server.core.command.system.basecommands
**Type:** Framework Component / Abstract Base Class

## Definition
```java
// Signature
public abstract class AbstractAsyncPlayerCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The AbstractAsyncPlayerCommand is a foundational component of the server's command system. It serves as the required base class for any command that must be executed by a player entity and requires asynchronous processing.

Its primary architectural role is to act as a **thread-dispatching gateway**. It abstracts away the complex and error-prone boilerplate of validating the command sender, ensuring they are a valid player currently present in a world, and then marshaling the execution logic onto the correct **World thread**.

By enforcing this pattern, the framework guarantees that any command logic implemented in a subclass is executed in a thread-safe context relative to the game state of the player and their surrounding world. This prevents a wide class of concurrency bugs and simplifies command development by providing the implementer with a pre-validated, world-specific context.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of AbstractAsyncPlayerCommand are instantiated by the CommandSystem during server bootstrap or when a plugin is loaded. This is typically handled via reflection-based scanning of registered command classes. The abstract class itself is never instantiated directly.
- **Scope:** A single instance of a concrete command subclass persists for the entire server session. These objects are stateless and are treated as shared definitions.
- **Destruction:** The command instance is de-registered and becomes eligible for garbage collection when the server shuts down or the owning plugin is unloaded.

## Internal State & Concurrency
- **State:** **Stateless and Immutable.** This class holds no mutable instance state. Its fields are static final Message constants used for error reporting. All state required for execution, such as the command sender and arguments, is passed via the CommandContext parameter during method invocation.

- **Thread Safety:** This class is inherently thread-safe. Its core purpose is to *enforce* a concurrency model, not manage its own concurrent state. The final `executeAsync(CommandContext)` method can be safely invoked from any thread (e.g., a Netty network thread). It safely transitions execution to the single-threaded World context before invoking the subclass's implementation, thereby protecting the game simulation from unsafe concurrent modifications.

## API Surface
The primary API is the abstract contract that subclasses must fulfill.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context, store, ref, player, world) | CompletableFuture<Void> | N/A | **Abstract contract for subclasses.** Implementations contain the core command logic. This method is guaranteed to be executed on the correct World thread for the player who issued the command. |

## Integration Patterns

### Standard Usage
A developer must extend this class to create a player-only asynchronous command. The framework provides all necessary, validated context objects as parameters to the abstract method.

```java
// A concrete command must extend this class and implement the abstract method.
public class GamemodeCommand extends AbstractAsyncPlayerCommand {

    public GamemodeCommand() {
        super("gamemode", "Sets a player's gamemode.");
    }

    @Override
    @Nonnull
    protected CompletableFuture<Void> executeAsync(
        @Nonnull CommandContext context,
        @Nonnull Store<EntityStore> entityStore,
        @Nonnull Ref<EntityStore> playerEntityRef,
        @Nonnull PlayerRef playerRefComponent,
        @Nonnull World world
    ) {
        // This logic is now safely running on the player's world thread.
        // It is safe to interact with the world and the player entity.
        String desiredMode = context.getArgument("mode");
        // ... logic to change the player's gamemode ...

        context.sendMessage(Message.literal("Gamemode updated."));
        return CompletableFuture.completedFuture(null);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations:** Do not perform long-running or blocking operations (e.g., file I/O, external network requests, complex database queries) directly within the `executeAsync` implementation. Doing so will stall the World thread, causing significant server lag or "ticks per second" loss. Offload such work to a separate, dedicated thread pool.
- **Incorrect Superclass:** Do not use this class for commands that can be executed by the server console or other non-player entities. Such commands must extend a more generic base like AbstractAsyncCommand.
- **Re-fetching Context:** Do not attempt to re-fetch the World or PlayerRef from a global service locator. Always use the validated parameters passed into the abstract method. These are guaranteed to be correct and thread-local to the execution context.

## Data Pipeline
This class sits between the generic command dispatcher and the specific game logic, acting as a validation and thread-management layer.

> Flow:
> Command Input -> CommandSystem Dispatcher -> **AbstractAsyncPlayerCommand** (Validation & Thread Dispatch) -> World Thread -> Concrete Command Implementation

