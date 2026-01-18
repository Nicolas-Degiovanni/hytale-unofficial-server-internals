---
description: Architectural reference for RepulsionRemoveCommand
---

# RepulsionRemoveCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.component.repulsion
**Type:** Transient

## Definition
```java
// Signature
public class RepulsionRemoveCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The RepulsionRemoveCommand class serves as a container or namespace within the server's command processing system. It does not implement any game logic itself; instead, it follows the **Composite Pattern** by extending AbstractCommandCollection to group related sub-commands. Its sole architectural purpose is to register the `remove` action under a parent `repulsion` command, delegating all execution to its specialized inner classes.

This command is a server-side administrative tool designed for debugging entity behaviors by directly manipulating the Entity Component System (ECS). The core operation involves removing the Repulsion component from a target entity, thereby altering its physics and interaction properties in real-time.

The two nested commands handle specific targeting scenarios:
*   **RepulsionRemoveEntityCommand:** A generic command that targets any entity in the world via its unique identifier. It extends AbstractWorldCommand, ensuring it operates within a valid world context.
*   **RepulsionRemoveSelfCommand:** A player-centric command that targets the entity associated with the command's executor. It extends AbstractTargetPlayerCommand, which provides a pre-validated context including the player and their corresponding entity reference.

## Lifecycle & Ownership
- **Creation:** A single instance of RepulsionRemoveCommand is instantiated by the server's CommandSystem during the bootstrap phase. The system scans for command definitions and registers them for the server's operational lifetime.
- **Scope:** The command object is a long-lived singleton, persisting as long as the server is running. It is held within the central command registry. The execution context and arguments associated with any given invocation are, however, ephemeral and discarded after the command completes.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only during server shutdown when the CommandSystem clears its registry.

## Internal State & Concurrency
- **State:** This class and its sub-commands are fundamentally stateless. They do not cache data or maintain state between executions. All necessary context, such as the target entity and the world's entity store, is provided as arguments to the `execute` method for each invocation.
- **Thread Safety:** This command is not thread-safe and is not designed to be. Command execution is dispatched by the CommandSystem onto the main server thread, which serially processes game ticks. This design relies on the server's single-threaded game loop to prevent race conditions when modifying the world state, such as removing a component from an entity.

**WARNING:** Invoking the `execute` method from an asynchronous thread will lead to world state corruption, concurrency exceptions, and server instability. All command logic must be executed on the main server thread.

## API Surface
The primary interface for this class is not programmatic but rather the command syntax it exposes to the server console. The internal `execute` methods form the contract with the CommandSystem.

### RepulsionRemoveEntityCommand
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(log N) | Fetches an entity by its ID from the world store and removes its Repulsion component. Complexity is dependent on the underlying entity store's lookup performance. |

### RepulsionRemoveSelfCommand
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, source, ref, player, world, store) | void | O(1) | Removes the Repulsion component from the command executor's entity. The target reference is provided directly by the command system, making the operation constant time. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a server administrator or developer through the server console. The CommandSystem parses the input and routes it to the appropriate sub-command for execution.

```
// Example: Remove the Repulsion component from a specific entity
/repulsion remove entity <entityId>

// Example: Remove the Repulsion component from your own player entity
/repulsion remove self
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new RepulsionRemoveCommand()`. The server's CommandSystem is solely responsible for the lifecycle of command objects. Manual instantiation will result in a non-functional command that is not registered with the server.
- **Programmatic Invocation:** Avoid calling the `execute` method directly from other game systems. Doing so bypasses critical infrastructure provided by the CommandSystem, including argument parsing, permission validation, and context setup. If you need to share this logic, refactor the component removal operation into a separate, reusable service.

## Data Pipeline
The flow for this command begins with user input and ends with a direct mutation of the Entity Component System state.

> Flow:
> Server Console Input -> Command Parser -> Command Dispatcher -> **RepulsionRemoveCommand** -> Sub-command (`entity` or `self`) -> `Store.tryRemoveComponent` -> World State Mutation -> Client Synchronization (Implicit)

