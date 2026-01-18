---
description: Architectural reference for MoveCommand
---

# MoveCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient Command Handler

## Definition
```java
// Signature
public class MoveCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The MoveCommand class is a server-side command handler responsible for processing the player-facing `/move` command within the Builder Tools plugin. It serves as the primary interface between player chat input and the underlying world manipulation systems.

Architecturally, this class embodies the **Command Pattern**. It encapsulates a request (to move a selection of blocks and entities) as an object, decoupling the request's issuer (the player) from the object that performs the action (the BuilderToolsPlugin).

Its primary responsibilities are:
1.  **Syntax Definition:** It declaratively defines the arguments and flags for the `/move` command using the command system framework (e.g., `withFlagArg`, `withRequiredArg`). This includes support for multiple usage variants through nested classes, allowing for flexible command structures like `/move 5` and `/move forward 5`.
2.  **Context Validation:** It performs initial checks to ensure the command can be executed, such as verifying the player's selection state via `PrototypePlayerBuilderToolSettings.isOkayToDoCommandsOnSelection`.
3.  **Intent Translation:** It translates the parsed command arguments (direction, distance) and the player's current state (head rotation) into a concrete, world-relative directional vector.
4.  **Delegation:** Crucially, MoveCommand does **not** perform the world modification itself. It delegates the operation by creating a lambda function and submitting it to the `BuilderToolsPlugin`'s work queue. This is a critical design choice to prevent the command execution thread from blocking on potentially long-running world edits, ensuring server performance and responsiveness.

The class operates within an Entity-Component-System (ECS) framework, receiving an `EntityStore` and `PlayerRef` to access player-specific components like `HeadRotation` and `Player`.

## Lifecycle & Ownership
-   **Creation:** An instance of MoveCommand is created by the server's command registration system, typically during the initialization phase of the `BuilderToolsPlugin`. It is not intended for manual instantiation during the game loop.
-   **Scope:** The command definition object is long-lived, persisting in the server's command registry for the entire server session, or until the parent plugin is disabled. The object itself is stateless regarding individual executions; all contextual data is passed into the `execute` method.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the `BuilderToolsPlugin` is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** The MoveCommand instance holds configuration state defined during its construction (e.g., `emptyFlag`, `entitiesFlag`). This state is immutable after the constructor completes. The class is stateless concerning individual command invocations.
-   **Thread Safety:** This class is thread-safe. Its internal state is read-only post-construction. The `execute` method is invoked by the server's main command processing thread. The handoff to `BuilderToolsPlugin.addToQueue` is a thread-safe operation that transfers the execution logic to a dedicated worker thread, preventing concurrency issues and ensuring that all builder tool operations are serialized.

## API Surface
The public API is minimal, designed for integration with the command system framework, not for direct invocation by other game systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | Framework entry point. Validates context and enqueues the move operation. The complexity is constant time as it only performs a lightweight handoff. |

## Integration Patterns

### Standard Usage
This class is not used directly in code. It is invoked by the server's command dispatcher when a player with the appropriate permissions (Creative GameMode) executes the command in chat.

*Player Input Example:*
```
/move forward 10 --entities
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new MoveCommand()` for any purpose other than registering it with the command system. Manually calling `execute` will fail due to the lack of a valid `CommandContext`.
-   **Synchronous World Edits:** Do not modify this class to perform the move operation directly within the `execute` method. All heavy-lifting must be delegated to the `BuilderToolsPlugin` queue to avoid blocking the server thread.

## Data Pipeline
The flow of data from player input to world state change is a clear example of decoupled, asynchronous processing.

> Flow:
> Player Chat Input (`/move ...`) -> Server Command Dispatcher -> **MoveCommand.execute** -> Parameter & State Aggregation -> `BuilderToolsPlugin.addToQueue` -> Builder Tools Worker Thread -> `Selection.move` -> World State Mutation -> Network Replication to Clients

