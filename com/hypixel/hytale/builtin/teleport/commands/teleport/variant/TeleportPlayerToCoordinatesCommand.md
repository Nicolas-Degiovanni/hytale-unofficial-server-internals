---
description: Architectural reference for TeleportPlayerToCoordinatesCommand
---

# TeleportPlayerToCoordinatesCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport.variant
**Type:** Command Object

## Definition
```java
// Signature
public class TeleportPlayerToCoordinatesCommand extends CommandBase {
```

## Architecture & Concepts
The TeleportPlayerToCoordinatesCommand is a server-side command handler that implements the logic for teleporting a player to a specific set of world coordinates. It is a concrete implementation within the server's Command System, responsible for parsing arguments, validating permissions, and initiating the teleportation process.

Architecturally, this class serves as a bridge between the user input (a typed command) and the server's Entity Component System (ECS). Its primary design principle is the **decoupling of intent from execution**. This command does not directly manipulate the player's position. Instead, it follows a deferred execution pattern:

1.  It parses the target player, coordinates, and optional rotation from the CommandContext.
2.  It calculates the final destination, resolving any relative coordinates (e.g., `~5`, `^10`).
3.  It schedules a task on the target entity's **World thread** using `World.execute`. This is a critical step for ensuring thread safety, as all modifications to world state must occur on the world's main tick loop.
4.  Within the scheduled task, it adds a **Teleport** component to the target entity.

The actual movement of the entity is handled by a separate, dedicated system (e.g., a TeleportSystem) that processes entities with the Teleport component during the server tick. This Command-Component pattern is fundamental to the engine's design, preventing command handlers from causing concurrency issues or directly meddling with complex game logic systems.

### Lifecycle & Ownership
-   **Creation:** An instance of TeleportPlayerToCoordinatesCommand is created by the Command System during server initialization or when its parent module is loaded. It is registered as a handler for a specific command signature.
-   **Scope:** The object is a long-lived singleton for the duration of the server's runtime. A single instance handles all executions of this command variant.
-   **Destruction:** The instance is de-referenced and eligible for garbage collection upon server shutdown or when the command is explicitly unregistered.

## Internal State & Concurrency
-   **State:** This class is effectively **stateless** between executions. Its member fields, such as playerArg and xArg, are final definitions of the command's argument structure. All state related to a specific command execution is contained within the CommandContext object passed to the `executeSync` method.

-   **Thread Safety:** This class is **not thread-safe** and must not be invoked from multiple threads. The framework guarantees that `executeSync` is called from a designated command processing thread. The implementation ensures safety for world modifications by scheduling the core logic onto the appropriate World thread via `World.execute`. Any attempt to modify entity components outside of this scheduled lambda would introduce severe race conditions.

## API Surface
The public contract of this class is with the Command System framework, not with general-purpose developers. The primary entry point is the overridden `executeSync` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) (scheduling) | Parses arguments from the context and schedules a teleport task on the target World thread. The operation itself is asynchronous. Throws if arguments are invalid. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly. It is automatically invoked by the server's command dispatcher when a player or the console executes the corresponding command. The standard interaction is through the game's command-line interface.

**Example Command Execution:**
```plaintext
/teleport PlayerName 100 ~ -50
/teleport OtherPlayer 0 128 0 90.0 -45.0
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new TeleportPlayerToCoordinatesCommand()`. The command system manages the lifecycle of command objects. Manual instantiation serves no purpose.
-   **Direct Invocation:** Never call the `executeSync` method directly. It requires a fully-formed CommandContext, which is constructed by the command parsing and dispatching pipeline. Bypassing this will lead to unpredictable behavior and NullPointerExceptions.
-   **Assuming Synchronous Execution:** The name `executeSync` refers to its execution on the command thread, not that the teleport is synchronous. The teleportation is deferred and occurs on a subsequent server tick. Code that dispatches this command cannot assume the entity has moved immediately after the command returns.

## Data Pipeline
The flow of data from user input to final entity state change follows a clear, multi-stage pipeline, ensuring separation of concerns and thread safety.

> Flow:
> Player Chat Input -> Server Network Listener -> Command Parser -> **TeleportPlayerToCoordinatesCommand** -> World Thread Scheduler -> Add Teleport Component -> ECS TeleportSystem -> Update TransformComponent -> Network Sync to Clients

