---
description: Architectural reference for InteractionRunCommand
---

# InteractionRunCommand

**Package:** com.hypixel.hytale.server.core.modules.interaction.commands
**Type:** Transient

## Definition
```java
// Signature
public class InteractionRunCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The InteractionRunCommand class is a server-side administrative command that provides a direct entry point into the Hytale Interaction System. It functions as a concrete implementation of the Command Pattern, translating a text-based console command into a high-level gameplay action.

Its primary architectural role is to serve as a diagnostic and debugging tool for developers and administrators. It allows for the manual invocation of an entity's interaction logic, bypassing the standard player-input triggers. This is critical for testing interaction chains, validating asset configurations, and debugging state-related issues within the Interaction System without requiring a live player client.

This command is not part of the standard gameplay loop. It directly retrieves the target entity's InteractionManager component and uses it to construct and queue an InteractionChain for execution on the next server tick.

## Lifecycle & Ownership
- **Creation:** A single instance of InteractionRunCommand is instantiated by the server's command registration system during server bootstrap. The system discovers and registers all classes extending AbstractPlayerCommand or similar base types.
- **Scope:** The command object is a long-lived singleton, persisting for the entire server session. Its internal state is configured once at creation and is immutable thereafter.
- **Destruction:** The instance is de-referenced and eligible for garbage collection only when the server shuts down or the parent module is unloaded.

## Internal State & Concurrency
- **State:** This class is effectively stateless from an execution perspective. It holds immutable, pre-configured fields defining its command arguments, such as interactionTypeArg. All state required for execution (the target entity, the world, the command context) is passed as parameters to the execute method.
- **Thread Safety:** The command's execute method is designed to be called exclusively by the server's main game thread. It directly manipulates core game state components like EntityStore and InteractionManager, which are not thread-safe. Concurrently invoking execute would lead to race conditions and severe world state corruption. The singleton instance itself is safe to access from multiple threads due to its immutable internal state, but its methods are not.

## API Surface
The public contract is defined by its inheritance from AbstractPlayerCommand. The primary entry point is the execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Resolves command arguments, retrieves the InteractionManager, and queues a new InteractionChain for execution. Complexity is constant time, dominated by component lookups and asset hashmap access. |

## Integration Patterns

### Standard Usage
This command is intended for use by server administrators or developers via the server console or in-game chat with sufficient permissions. It manually triggers an interaction type on the executing player.

```java
// Console or in-game command execution
/interaction run PRIMARY
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class manually with `new InteractionRunCommand()`. The server's command system is the sole owner of its lifecycle. Manually creating an instance will result in a non-functional, unregistered command.
- **Manual Invocation:** Do not call the `execute` method directly from other game logic systems. This violates the separation of concerns between gameplay logic and administrative commands. If a game system needs to trigger an interaction, it must acquire the InteractionManager component and use its API directly.
- **Stateful Logic:** Do not modify this class to hold state between executions. Command objects must be stateless to ensure predictable behavior.

## Data Pipeline
The command initiates a flow of data that begins with user input and results in a change to the game state. It acts as the trigger for the Interaction System's own internal pipeline.

> Flow:
> Server Console Input -> Command Parser -> **InteractionRunCommand.execute()** -> InteractionManager.initChain() -> InteractionManager.queueExecuteChain() -> Interaction System Tick -> Game State Mutation

