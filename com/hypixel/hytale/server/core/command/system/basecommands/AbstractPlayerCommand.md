---
description: Architectural reference for AbstractPlayerCommand
---

# AbstractPlayerCommand

**Package:** com.hypixel.hytale.server.core.command.system.basecommands
**Type:** Template Class

## Definition
```java
// Signature
public abstract class AbstractPlayerCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The AbstractPlayerCommand class is a foundational component of the server's command system. It serves as a specialized base class for any command that must be executed by a player entity. Its primary architectural role is to act as a **guard and context resolver**, enforcing the precondition that the command's sender is a valid, in-world player.

By extending this class, developers are freed from writing repetitive and error-prone boilerplate for player validation and data retrieval. The class guarantees that by the time the core command logic is executed, essential context objects—such as the player's EntityStore, World, and PlayerRef component—are already resolved, validated, and provided.

This class embodies the **Template Method** design pattern. The `final` method executeAsync defines the skeleton of the operation (validation, context fetching, scheduling), and defers the actual command logic to subclasses through the `abstract` execute method. This ensures a consistent, safe execution model for all player-centric commands.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses of AbstractPlayerCommand are instantiated once by the server's CommandSystem during the bootstrap phase. They are discovered and registered to create a central command registry. This abstract class is never instantiated directly.
- **Scope:** Registered command instances are singletons that persist for the entire server session. They are stateless templates used to process executions.
- **Destruction:** The command objects are dereferenced and eligible for garbage collection when the server shuts down and the CommandSystem is cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally **stateless**. Its only member fields are `static final` Message objects, which are immutable constants. All state required for an execution is passed transiently through the CommandContext parameter.

- **Thread Safety:** The class is inherently thread-safe. The primary entry point, executeAsync, can be safely invoked from any thread, typically the server's main network thread. The implementation guarantees that the subclass's execute method is invoked on the correct world-specific thread via the `runAsync` scheduler inherited from AbstractAsyncCommand. This prevents race conditions and ensures safe mutation of game state.

## API Surface
The public contract is defined by the abstract method that subclasses must implement. The `final` executeAsync method is the entry point but is not part of the extension contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | User-defined | **Extension Point.** Abstract method where concrete command logic is implemented. Guaranteed to be called with a valid, in-world player context. |

## Integration Patterns

### Standard Usage
Developers should extend this class to create any command intended for execution by a player. The implementation should focus solely on the command's logic within the overridden execute method.

```java
// A concrete command to set the player's spawn point.
public class SetSpawnCommand extends AbstractPlayerCommand {

    public SetSpawnCommand() {
        super("setspawn", "Sets your personal spawn point to your current location.");
    }

    @Override
    protected void execute(
        @Nonnull CommandContext context,
        @Nonnull Store<EntityStore> store,
        @Nonnull Ref<EntityStore> ref,
        @Nonnull PlayerRef playerRef,
        @Nonnull World world
    ) {
        // The system has already validated the player and world.
        // We can safely access player and world data here.
        Position currentPosition = world.getPosition(ref);
        playerRef.setSpawnPoint(currentPosition);

        context.sendMessage(Message.translation("server.commands.setspawn.success"));
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Player Validation:** Do not re-implement player checks within the execute method. The base class contract guarantees the sender is a valid player. Adding redundant checks indicates a misunderstanding of the architecture.
- **Assuming Console Sender:** Do not design a command that could logically be run by the server console while inheriting from this class. This class will always fail the execution if the sender is not a player. Use a more generic base class like AbstractCommand for such cases.
- **Blocking Operations:** The execute method runs on a world-specific game thread. Performing long-running or blocking I/O operations within this method will stall the world's tick loop, causing severe server lag. Defer such work to a separate worker thread pool.

## Data Pipeline
The class sits in the middle of the command processing pipeline, acting as the primary dispatcher and validator for player-initiated commands.

> Flow:
> Player Chat Input -> Network Layer -> Command Parser -> **AbstractPlayerCommand.executeAsync** -> World Thread Scheduler -> ConcreteCommand.execute -> Game State Mutation

