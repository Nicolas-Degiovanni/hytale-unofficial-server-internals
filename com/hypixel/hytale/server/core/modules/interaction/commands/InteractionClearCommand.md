---
description: Architectural reference for InteractionClearCommand
---

# InteractionClearCommand

**Package:** com.hypixel.hytale.server.core.modules.interaction.commands
**Type:** Transient

## Definition
```java
// Signature
public class InteractionClearCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The InteractionClearCommand class is a concrete implementation of the Command design pattern, tailored for the server's command processing system. It serves as a thin, stateless adapter that translates a player-initiated text command into a direct action on a core game system.

Its primary architectural role is to decouple the command input layer from the game logic layer. The server's Command Dispatcher identifies this class as the handler for the "clear" command within the interaction module's namespace. Upon invocation, it retrieves the session's InteractionManager component and triggers its state-clearing functionality.

As a subclass of AbstractPlayerCommand, it is intrinsically linked to a player context, ensuring that its execution is always associated with a specific player entity within a world.

### Lifecycle & Ownership
- **Creation:** A single instance of InteractionClearCommand is instantiated by the server's command registration system during the server bootstrap phase. It is then stored in a central command registry, mapped to its designated command string "clear".
- **Scope:** The object instance persists for the entire server session, held statically by the command registry. The object itself is long-lived, but its execution context is ephemeral, lasting only for the duration of the execute method call.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. It contains no mutable instance fields. All required data for its operation, such as the World, PlayerRef, and component stores, are provided as arguments to the execute method. This design ensures that each command execution is isolated and idempotent.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. Multiple threads could theoretically invoke the execute method on the shared instance without risk of data corruption within the command object itself.

    **WARNING:** While the command object is thread-safe, the underlying InteractionManager component it operates on must implement its own concurrency control. The command itself offers no protection against race conditions within the systems it modifies.

## API Surface
The public contract is minimal, consisting of the constructor for initialization by the framework and the overridden execute method for invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| InteractionClearCommand() | constructor | O(1) | Registers the command name "clear" and its description key with the parent class. |
| execute(...) | protected void | O(N) | Retrieves the InteractionManager and invokes its clear method. Complexity is dependent on the number of interactions (N) to be cleared. |

## Integration Patterns

### Standard Usage
A developer or user does not interact with this class directly. The system is designed to be driven by commands entered into the server console or chat. The framework is responsible for routing the command to this handler.

The conceptual flow managed by the command system is as follows:
```java
// PSEUDOCODE: How the Command Dispatcher uses this class
CommandRegistry registry = server.getCommandRegistry();
AbstractPlayerCommand command = registry.find("clear"); // Finds the InteractionClearCommand instance

// The dispatcher builds the context and invokes the command
command.execute(context, store, ref, playerRef, world);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new InteractionClearCommand()`. The command system requires a single, registered instance to function correctly. Manual instantiation bypasses the registry and will have no effect.
- **Manual Invocation:** Avoid calling the `execute` method directly. Always dispatch commands through the server's central command handling service. Bypassing the service skips critical steps like permission checks, argument parsing, and context validation, which can lead to an unstable or insecure state.

## Data Pipeline
This component acts as a control-flow endpoint, not a data-processing stage. It receives a command signal and translates it into a method call on another system.

> Flow:
> Player Chat Input (`/interaction clear`) -> Network Layer -> Command Parser -> Command Dispatcher -> **InteractionClearCommand.execute()** -> InteractionManager.clear()

